---
published: false
layout: post
title: "如何将50行的if-else块重构成3行"
date: 2014-07-08 10:52
comments: true
categories: [技术杂谈]
---

## 背景

今天下午花了半天时间重构了客户遗留系统中一个*一万行*的类

<!--more -->

----------------------------------------------------------我是分割线----------------------------------------------------------

里面的500行代码。

心情大好，Mark一下。

## 业务

我们做的是一个批发商信息管理系统，提供了一个更新批发商信息的服务，有多个别的系统可以通过调用该服务发送更新批发商信息的请求。

在这里，批发商们有一个很重要的信息叫做公司简介（Summary）。

由于很多业务上的原因（具体原因就不讨论了），其他系统发过来的更新请求有可能是不受信任的（untrusted），因此我们针对Summary添加了一个标志位，如果消息源受信任，就把它设置成trusted，如果不受信任，就设成untrusted。

## 新的业务

在这个系统运行了十年过后（你没看错，就是*十年*），客户终于发现，这里面有个Bug！！！

*举个栗子*

---

\>> 我是一个快乐的批发商，我在一个受信任的系统中更新了我的Summary。

\>> 这时候，数据库里面有了我的一个Summary数据，这个数据是正确（Trusted）的。

\>> 突然有一天，某个奇怪的人想通过某个不受信任的系统更新*我*的Summary。

\>> 当然啦，我们的系统是很人性化的，它会把这个请求标记成untrusted。

\>> 但是，奇怪的事情发生了，

\>> 系统用这个untrusted的Summary覆盖了我自己提交的Summary（天哪！！！）

\>> 又有一天，我登录系统来看我的Summary

\>> 我不高兴了。。。。。

---

## 于是

客户找到了Dev，让我们改掉这个Bug。

> 很简单嘛， 就是不要让untrusted的信息覆盖trusted嘛，别的逻辑不变嘛

## 如何改

我们说过了，这是一个*一万行*的类，为了安全的修改，我们需要做如下的事情：

1. 找到要修改的需要修改的地方，
2. 将需要修改的部分提取到一个单独的类，
3. 给他加测试，让我们的测试覆盖所有可能的情况（需要注意的是，不光要保证行覆盖率和分支覆盖率都要达到100%，而且要覆盖所有的Scenario），
4. 重构抽出来的这部分代码，让一个健康的人可以理解它，
5. 把测试改成我们现在期望的行为，
6. 最后，改掉这个Bug。

下面，我们来演示一下。

## 找到需要修改的地方

（好像没什么技巧？那就略过把）

## 把找到的东西抽取出来

嗯，这是我们抽出来的代码：(还没开始改哦亲！)

{% codeblock SummaryUpdateHelper.java %}
public class SummaryUpdateHelper {
    private List<String> updateFieldNames;
    private Map<String, Object> updatedFields;

    public SummaryUpdateHelper(List<String> updateFieldNames,
                               Map<String, Object> updatedFields) {
        this.updateFieldNames = updateFieldNames;
        this.updatedFields = updatedFields;
    }

    public void updateSummary(Request request, Customer customer) {
        RequestSummary requestSummary = request.getSummary();
        Summary summaryFromDb = customer.getSummary();

        if (summaryFromDb == null && requestSummary != null) {
            summaryFromDb = new Summary();
        }

        if (null != requestSummary) {
            updateSummaryAttributes(requestSummary, summaryFromDb);
        }
        if (updatedFields.containsKey("summary")) {
            customer.setSummary(summaryFromDb);
        }
    }

    private void updateSummaryAttributes(RequestSummary requestSummary,
                                         Summary dbSummary) {
        String requestSummaryDetail = requestSummary.getSummaryDetail();
        TrustIndicator requestTrustIndicator = requestSummary.getTrustIndicator();

        String dbSummaryDetail = dbSummary.getDetail();
        TrustIndicator dbTrustIndicator = dbSummary.getTrustIndicator();

        SummaryUpdateAction summaryUpdateAction =
                getSummaryUpdateAction(requestSummaryDetail,
                dbSummaryDetail, requestTrustIndicator, dbTrustIndicator);

        if (summaryUpdateAction.updateDetail) {
            dbSummary.setDetail(requestSummaryDetail);
            updateFieldNames.add("summaryDetail");
            updatedFields.put("summary", dbSummary);
        }

        dbSummary.setTrustIndicator(summaryUpdateAction.updatedTrustIndicator);

        if (summaryUpdateAction.updatedTrustIndicator != dbTrustIndicator) {
            updateFieldNames.add("summaryIndicator");
            updatedFields.put("summary", dbSummary);
        }
    }

    private SummaryUpdateAction getSummaryUpdateAction(String requestSummaryDetail,
                                                       String dbSummaryDetail,
                                                       TrustIndicator requestTrustIndicator,
                                                       TrustIndicator dbTrustIndicator) {
        SummaryUpdateAction summaryUpdateAction;
        boolean isRequestDetailBlank;
        boolean isDbDetailBlank;
        boolean isRequestAndDbDetailSame;

        summaryUpdateAction = new SummaryUpdateAction();
        summaryUpdateAction.updatedTrustIndicator = dbTrustIndicator;

        if (null == dbSummaryDetail || "".equals(dbSummaryDetail)) {
            isDbDetailBlank = true;
        } else {
            isDbDetailBlank = false;
        }

        if (null == requestSummaryDetail || "".equals(requestSummaryDetail)) {
            isRequestDetailBlank = true;
        } else {
            isRequestDetailBlank = false;
        }

        if (null != requestSummaryDetail && null != dbSummaryDetail
                && requestSummaryDetail.equalsIgnoreCase(dbSummaryDetail)) {
            isRequestAndDbDetailSame = true;
        } else {
            isRequestAndDbDetailSame = false;
        }

        if (isRequestDetailBlank && !isDbDetailBlank && TRUSTED == requestTrustIndicator) {
            if (TRUSTED == dbTrustIndicator) {
                summaryUpdateAction.updateDetail = true;
            } else if (UNTRUSTED == dbTrustIndicator) {
                summaryUpdateAction.updateDetail = true;
                summaryUpdateAction.updatedTrustIndicator = TRUSTED;
            }
        } else if (!isRequestDetailBlank && isDbDetailBlank) {
            summaryUpdateAction.updateDetail = true;
            summaryUpdateAction.updatedTrustIndicator = requestTrustIndicator;
        } else if (!isRequestDetailBlank && !isDbDetailBlank) {
            if (TRUSTED == dbTrustIndicator
                    && TRUSTED == requestTrustIndicator
                    && !isRequestAndDbDetailSame) {
                summaryUpdateAction.updateDetail = true;
            } else if (UNTRUSTED == dbTrustIndicator && TRUSTED == requestTrustIndicator) {
                summaryUpdateAction.updatedTrustIndicator = TRUSTED;
                if (!isRequestAndDbDetailSame) {
                    summaryUpdateAction.updateDetail = true;
                }
            } else if (UNTRUSTED == dbTrustIndicator
                    && UNTRUSTED == requestTrustIndicator
                    && !isRequestAndDbDetailSame) {
                summaryUpdateAction.updateDetail = true;
            } else if (TRUSTED == dbTrustIndicator
                    && UNTRUSTED == requestTrustIndicator
                    && !isRequestAndDbDetailSame) {
                summaryUpdateAction.updateDetail = true;
                summaryUpdateAction.updatedTrustIndicator = UNTRUSTED;
            }
        }
        return summaryUpdateAction;
    }

    private class SummaryUpdateAction {
        public boolean updateDetail;
        public TrustIndicator updatedTrustIndicator;
    }
}
{% endcodeblock %}

## 加测试

我们用Spock写了一个很*性感*的测试：（实际情况是，用jUnit写出来的测试实在是看不懂，木有办法了才用这个。。。。。。）

{% codeblock SummaryUpdateHelperTest.groovy %}
class SummaryUpdateHelperTest extends Specification {

    @Unroll
    def "request[#reqSummary, #reqTrust] update db[#dbSummary, #dbTrust] = [#add, #finalSummary, #finalTrust]"() {
        given:
        def updateFieldNames = newArrayList()
        def updatedFields = newHashMap()
        def helper = new SummaryUpdateHelper(updateFieldNames, updatedFields)
        def request = createRequest(reqSummary, reqTrust)
        def customer = createCustomer(dbSummary, dbTrust)

        when:
        helper.updateSummary(request, customer)

        then:
        customer.summary == createSummary(finalSummary, finalTrust)
        updateFieldNames == createUpdateFieldNames(updatedType)
        updatedFields.get("summary") == createSummary(updatedSummary, finalTrust)

        where:
        reqSummary | reqTrust  | dbSummary | dbTrust   | finalSummary | finalTrust | updatedType | updatedSummary
        null       | null      | null      | null      | null         | null       | NONE        | null
        null       | null      | ORIGINAL  | TRUSTED   | ORIGINAL     | TRUSTED    | NONE        | null

        BLANK      | TRUSTED   | null      | null      | null         | null       | NONE        | null
        BLANK      | UNTRUSTED | null      | null      | null         | null       | NONE        | null
        BLANK      | TRUSTED   | BLANK     | TRUSTED   | BLANK        | TRUSTED    | NONE        | null
        BLANK      | TRUSTED   | BLANK     | UNTRUSTED | BLANK        | UNTRUSTED  | NONE        | null
        BLANK      | UNTRUSTED | BLANK     | TRUSTED   | BLANK        | TRUSTED    | NONE        | null
        BLANK      | UNTRUSTED | BLANK     | UNTRUSTED | BLANK        | UNTRUSTED  | NONE        | null
        BLANK      | TRUSTED   | EMPTY     | TRUSTED   | EMPTY        | TRUSTED    | NONE        | null
        BLANK      | TRUSTED   | EMPTY     | UNTRUSTED | EMPTY        | UNTRUSTED  | NONE        | null
        BLANK      | UNTRUSTED | EMPTY     | TRUSTED   | EMPTY        | TRUSTED    | NONE        | null
        BLANK      | UNTRUSTED | EMPTY     | UNTRUSTED | EMPTY        | UNTRUSTED  | NONE        | null
        BLANK      | TRUSTED   | ORIGINAL  | TRUSTED   | BLANK        | TRUSTED    | SUMMARY     | BLANK
        BLANK      | TRUSTED   | ORIGINAL  | UNTRUSTED | BLANK        | TRUSTED    | BOTH        | BLANK
        BLANK      | UNTRUSTED | ORIGINAL  | TRUSTED   | ORIGINAL     | TRUSTED    | NONE        | null
        BLANK      | UNTRUSTED | ORIGINAL  | UNTRUSTED | ORIGINAL     | UNTRUSTED  | NONE        | null

        EMPTY      | TRUSTED   | null      | null      | null         | null       | NONE        | null
        EMPTY      | UNTRUSTED | null      | null      | null         | null       | NONE        | null
        EMPTY      | TRUSTED   | BLANK     | TRUSTED   | BLANK        | TRUSTED    | NONE        | null
        EMPTY      | TRUSTED   | BLANK     | UNTRUSTED | BLANK        | UNTRUSTED  | NONE        | null
        EMPTY      | UNTRUSTED | BLANK     | TRUSTED   | BLANK        | TRUSTED    | NONE        | null
        EMPTY      | UNTRUSTED | BLANK     | UNTRUSTED | BLANK        | UNTRUSTED  | NONE        | null
        EMPTY      | TRUSTED   | EMPTY     | TRUSTED   | EMPTY        | TRUSTED    | NONE        | null
        EMPTY      | TRUSTED   | EMPTY     | UNTRUSTED | EMPTY        | UNTRUSTED  | NONE        | null
        EMPTY      | UNTRUSTED | EMPTY     | TRUSTED   | EMPTY        | TRUSTED    | NONE        | null
        EMPTY      | UNTRUSTED | EMPTY     | UNTRUSTED | EMPTY        | UNTRUSTED  | NONE        | null
        EMPTY      | TRUSTED   | ORIGINAL  | TRUSTED   | EMPTY        | TRUSTED    | SUMMARY     | EMPTY
        EMPTY      | TRUSTED   | ORIGINAL  | UNTRUSTED | EMPTY        | TRUSTED    | BOTH        | EMPTY
        EMPTY      | UNTRUSTED | ORIGINAL  | TRUSTED   | ORIGINAL     | TRUSTED    | NONE        | null
        EMPTY      | UNTRUSTED | ORIGINAL  | UNTRUSTED | ORIGINAL     | UNTRUSTED  | NONE        | null

        ORIGINAL   | TRUSTED   | null      | null      | ORIGINAL     | TRUSTED    | BOTH        | ORIGINAL
        ORIGINAL   | UNTRUSTED | null      | null      | ORIGINAL     | UNTRUSTED  | BOTH        | ORIGINAL
        ORIGINAL   | TRUSTED   | BLANK     | TRUSTED   | ORIGINAL     | TRUSTED    | SUMMARY     | ORIGINAL
        ORIGINAL   | TRUSTED   | BLANK     | UNTRUSTED | ORIGINAL     | TRUSTED    | BOTH        | ORIGINAL
        ORIGINAL   | UNTRUSTED | BLANK     | TRUSTED   | ORIGINAL     | UNTRUSTED  | BOTH        | ORIGINAL
        ORIGINAL   | UNTRUSTED | BLANK     | UNTRUSTED | ORIGINAL     | UNTRUSTED  | SUMMARY     | ORIGINAL
        ORIGINAL   | TRUSTED   | EMPTY     | TRUSTED   | ORIGINAL     | TRUSTED    | SUMMARY     | ORIGINAL
        ORIGINAL   | TRUSTED   | EMPTY     | UNTRUSTED | ORIGINAL     | TRUSTED    | BOTH        | ORIGINAL
        ORIGINAL   | UNTRUSTED | EMPTY     | TRUSTED   | ORIGINAL     | UNTRUSTED  | BOTH        | ORIGINAL
        ORIGINAL   | UNTRUSTED | EMPTY     | UNTRUSTED | ORIGINAL     | UNTRUSTED  | SUMMARY     | ORIGINAL
        ORIGINAL   | TRUSTED   | ORIGINAL  | TRUSTED   | ORIGINAL     | TRUSTED    | NONE        | null
        ORIGINAL   | TRUSTED   | ORIGINAL  | UNTRUSTED | ORIGINAL     | TRUSTED    | INDICATOR   | ORIGINAL
        ORIGINAL   | UNTRUSTED | ORIGINAL  | TRUSTED   | ORIGINAL     | TRUSTED    | NONE        | null
        ORIGINAL   | UNTRUSTED | ORIGINAL  | UNTRUSTED | ORIGINAL     | UNTRUSTED  | NONE        | null

        UPDATED    | TRUSTED   | null      | null      | UPDATED      | TRUSTED    | BOTH        | UPDATED
        UPDATED    | UNTRUSTED | null      | null      | UPDATED      | UNTRUSTED  | BOTH        | UPDATED
        UPDATED    | TRUSTED   | BLANK     | TRUSTED   | UPDATED      | TRUSTED    | SUMMARY     | UPDATED
        UPDATED    | TRUSTED   | BLANK     | UNTRUSTED | UPDATED      | TRUSTED    | BOTH        | UPDATED
        UPDATED    | UNTRUSTED | BLANK     | TRUSTED   | UPDATED      | UNTRUSTED  | BOTH        | UPDATED
        UPDATED    | UNTRUSTED | BLANK     | UNTRUSTED | UPDATED      | UNTRUSTED  | SUMMARY     | UPDATED
        UPDATED    | TRUSTED   | EMPTY     | TRUSTED   | UPDATED      | TRUSTED    | SUMMARY     | UPDATED
        UPDATED    | TRUSTED   | EMPTY     | UNTRUSTED | UPDATED      | TRUSTED    | BOTH        | UPDATED
        UPDATED    | UNTRUSTED | EMPTY     | TRUSTED   | UPDATED      | UNTRUSTED  | BOTH        | UPDATED
        UPDATED    | UNTRUSTED | EMPTY     | UNTRUSTED | UPDATED      | UNTRUSTED  | SUMMARY     | UPDATED
        UPDATED    | TRUSTED   | ORIGINAL  | TRUSTED   | UPDATED      | TRUSTED    | SUMMARY     | UPDATED
        UPDATED    | TRUSTED   | ORIGINAL  | UNTRUSTED | UPDATED      | TRUSTED    | BOTH        | UPDATED
        UPDATED    | UNTRUSTED | ORIGINAL  | TRUSTED   | UPDATED      | UNTRUSTED  | BOTH        | UPDATED
        UPDATED    | UNTRUSTED | ORIGINAL  | UNTRUSTED | UPDATED      | UNTRUSTED  | SUMMARY     | UPDATED
    }

    def createUpdateFieldNames(UpdateType updateType) {
        def updateFieldNames = newArrayList()
        if (updateType.isSummaryUpdated()) {
            updateFieldNames.add("summaryDetail")
        }
        if (updateType.isIndicatorUpdated()) {
            updateFieldNames.add("summaryIndicator")
        }
        updateFieldNames
    }

    def createCustomer(SummaryType summaryType, TrustIndicator trustIndicator) {
        new Customer(summary: createSummary(summaryType, trustIndicator))
    }

    private Summary createSummary(SummaryType summaryType, TrustIndicator trustIndicator) {
        summaryType ? summaryWithDetail(summaryType, trustIndicator) : null
    }

    def summaryWithDetail(SummaryType summaryType, TrustIndicator trustIndicator) {
        new Summary(
                detail: summaryType.summaryDetail,
                trustIndicator: trustIndicator
        )
    }

    def createRequest(SummaryType requestSummary, TrustIndicator trustIndicator) {
        new Request(summary: requestSummary ?
                        requestSummaryWithDetail(requestSummary, trustIndicator) : null
        )
    }

    def requestSummaryWithDetail(SummaryType summaryType, TrustIndicator trustIndicator) {
        new RequestSummary(summaryType.summaryDetail, trustIndicator)
    }

    static enum SummaryType {
        BLANK(null), EMPTY(''), ORIGINAL('init.summary'), UPDATED('changed.summary')

        final String summaryDetail

        private SummaryType(String summaryDetail) {
            this.summaryDetail = summaryDetail
        }
    }

    static enum UpdateType {
        NONE, SUMMARY, INDICATOR, BOTH

        boolean isSummaryUpdated() {
            this == SUMMARY || this == BOTH
        }

        boolean isIndicatorUpdated() {
            this == INDICATOR || this == BOTH
        }
    }
}
{% endcodeblock %}

## 重构

接下来，就是要做重构了。

我们一眼就能看到，这里面有一个很销魂的方法：`getSummaryUpdateAction()`。别的就略过啦，今天我们就来看看如何重构这个方法。

上面那段代码中，你对他的了解可能不是很真切，现在让我们来仔细看看他的真面目：

{% codeblock SummaryUpdateHelper.java %}
private SummaryUpdateAction getSummaryUpdateAction(String requestSummaryDetail,
                                                   String dbSummaryDetail,
                                                   TrustIndicator requestTrustIndicator,
                                                   TrustIndicator dbTrustIndicator) {
    SummaryUpdateAction summaryUpdateAction;
    boolean isRequestDetailBlank;
    boolean isDbDetailBlank;
    boolean isRequestAndDbDetailSame;

    summaryUpdateAction = new SummaryUpdateAction();
    summaryUpdateAction.updatedTrustIndicator = dbTrustIndicator;

    if (null == dbSummaryDetail || "".equals(dbSummaryDetail)) {
        isDbDetailBlank = true;
    } else {
        isDbDetailBlank = false;
    }

    if (null == requestSummaryDetail || "".equals(requestSummaryDetail)) {
        isRequestDetailBlank = true;
    } else {
        isRequestDetailBlank = false;
    }

    if (null != requestSummaryDetail && null != dbSummaryDetail
            && requestSummaryDetail.equalsIgnoreCase(dbSummaryDetail)) {
        isRequestAndDbDetailSame = true;
    } else {
        isRequestAndDbDetailSame = false;
    }

    if (isRequestDetailBlank && !isDbDetailBlank && TRUSTED == requestTrustIndicator) {
        if (TRUSTED == dbTrustIndicator) {
            summaryUpdateAction.updateDetail = true;
        } else if (UNTRUSTED == dbTrustIndicator) {
            summaryUpdateAction.updateDetail = true;
            summaryUpdateAction.updatedTrustIndicator = TRUSTED;
        }
    } else if (!isRequestDetailBlank && isDbDetailBlank) {
        summaryUpdateAction.updateDetail = true;
        summaryUpdateAction.updatedTrustIndicator = requestTrustIndicator;
    } else if (!isRequestDetailBlank && !isDbDetailBlank) {
        if (TRUSTED == dbTrustIndicator
                && TRUSTED == requestTrustIndicator
                && !isRequestAndDbDetailSame) {
            summaryUpdateAction.updateDetail = true;
        } else if (UNTRUSTED == dbTrustIndicator && TRUSTED == requestTrustIndicator) {
            summaryUpdateAction.updatedTrustIndicator = TRUSTED;
            if (!isRequestAndDbDetailSame) {
                summaryUpdateAction.updateDetail = true;
            }
        } else if (UNTRUSTED == dbTrustIndicator
                && UNTRUSTED == requestTrustIndicator
                && !isRequestAndDbDetailSame) {
            summaryUpdateAction.updateDetail = true;
        } else if (TRUSTED == dbTrustIndicator
                && UNTRUSTED == requestTrustIndicator
                && !isRequestAndDbDetailSame) {
            summaryUpdateAction.updateDetail = true;
            summaryUpdateAction.updatedTrustIndicator = UNTRUSTED;
        }
    }
    return summaryUpdateAction;
}
{% endcodeblock %}

### 第一步，干掉里面最二的代码
 
像下面这个：

def SummaryUpdateHelper.java
    if (null == dbSummaryDetail || "".equals(dbSummaryDetail)) {
        isDbDetailBlank = true;
    } else {
        isDbDetailBlank = false;
    }
end
{:lang="java"}

一行代码就搞定了：`isDbDetailBlank = isNullOrEmpty(dbSummaryDetail);`

简单来说，这一步完成过后，IDE里面就不会出现警告了。

### 第二步，分离关注点

现在就需要动动脑筋了，首先我们可以看到