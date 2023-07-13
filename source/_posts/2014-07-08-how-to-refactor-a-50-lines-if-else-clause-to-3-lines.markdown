---
published: true
layout: post
title: "如何将50行的if-else块重构成3行"
date: 2014-07-08 10:52
comments: true
categories: [技术杂谈]
---

## 背景

今天下午花了半天时间重构了客户遗留系统中一个*一万行*的类

<!--more -->

------------------------------------------我是分割线-------------------------------------------

里面的500行代码。

心情大好，Mark一下。

## 业务

我们做的是一个批发商信息管理系统，提供了一个更新批发商信息的服务，有多个别的系统可以通过调用该服务发送更新批发商信息的请求。

在这里，批发商们有一个很重要的信息叫做公司简介（*Summary*）。

由于很多业务上的原因（具体原因就不讨论了），其他系统发过来的更新请求有可能是不受信任的（*untrusted*），因此我们针对*Summary*添加了一个标志位，如果消息源受信任，就把它设置成*trusted*，如果不受信任，就设成*untrusted*。

> 我就这么一说，你就这么一听，这可不是我们真实的客户场景
>
> 客户代码自然是不能传上来给大家看的啦，我已经把这里涉及到的业务场景改的面目全非了
>
> 费了我老大的劲了

## 新的业务

在这个系统运行了十年过后（你没看错，就是*十年*），客户终于发现，这里面有个Bug！！！

*举个栗子*

---

\>> 我是一个快乐的批发商，我在一个受信任的系统中更新了我的*Summary*。

\>> 这时候，数据库里面有了我的一个*Summary*数据，这个数据是正确（*trusted*）的。

\>> 突然有一天，某个奇怪的人想通过某个不受信任的系统更新*我*的*Summary*。

\>> 当然啦，我们的系统是很人性化的，它会把这个请求标记成*untrusted*。

\>> 但是，奇怪的事情发生了，

\>> 系统用这个*untrusted*的*Summary*覆盖了我自己提交的*Summary*（天哪！！！）

\>> 又有一天，我登录系统来看我的*Summary*

\>> 我不高兴了。。。。。

---

## 于是

客户找到了Dev，让我们改掉这个Bug。

> 很简单嘛， 就是不要让untrusted的信息覆盖trusted嘛，别的逻辑不变嘛

## 如何改

我们说过了，这是一个*一万行*的类。

>“你不会让我直接在里面改代码吧？”
>
>“当然不会啦！为了愉快的修改，我们会做足前戏的。”
>
>“纳尼？”

恩，就是下面这些工（前）作（戏）：

1. 在茫茫的代码中找到需要修改的地方，
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

{% codeblock SummaryUpdateHelper.java https://github.com/YaodanZhang/code-kata/blob/master/src/main/java/com/thoughtworks/kata/refactor/SummaryUpdateHelper.java %}
public class SummaryUpdateHelper {
    private Map<String, Object> updatedFields;

    public SummaryUpdateHelper(Map<String, Object> updatedFields) {
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

    private void updateSummaryAttributes(RequestSummary requestSummary, Summary dbSummary) {
        String requestSummaryDetail = requestSummary.getSummaryDetail();
        TrustIndicator requestTrustIndicator = requestSummary.getTrustIndicator();

        String dbSummaryDetail = dbSummary.getDetail();
        TrustIndicator dbTrustIndicator = dbSummary.getTrustIndicator();

        UpdateAction updateAction = getUpdateAction(requestSummaryDetail,
                dbSummaryDetail, requestTrustIndicator, dbTrustIndicator);

        if (updateAction.updateDetail) {
            dbSummary.setDetail(trimToNull(requestSummaryDetail));
            updatedFields.put("summary", dbSummary);
        }

        dbSummary.setTrustIndicator(updateAction.updatedTrustIndicator);

        if (updateAction.updatedTrustIndicator != dbTrustIndicator) {
            updatedFields.put("summary", dbSummary);
        }
    }

    private UpdateAction getUpdateAction(String requestSummaryDetail, String dbSummaryDetail,
                                         TrustIndicator requestTrustIndicator,
                                         TrustIndicator dbTrustIndicator) {
        UpdateAction updateAction;
        boolean isRequestDetailBlank;
        boolean isDbDetailBlank;
        boolean isRequestAndDbDetailSame;

        updateAction = new UpdateAction();
        updateAction.updatedTrustIndicator = dbTrustIndicator;

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
                updateAction.updateDetail = true;
            } else if (UNTRUSTED == dbTrustIndicator) {
                updateAction.updateDetail = true;
                updateAction.updatedTrustIndicator = TRUSTED;
            }
        } else if (!isRequestDetailBlank && isDbDetailBlank) {
            updateAction.updateDetail = true;
            updateAction.updatedTrustIndicator = requestTrustIndicator;
        } else if (!isRequestDetailBlank && !isDbDetailBlank) {
            if (TRUSTED == dbTrustIndicator && TRUSTED == requestTrustIndicator
                    && !isRequestAndDbDetailSame) {
                updateAction.updateDetail = true;
            } else if (UNTRUSTED == dbTrustIndicator && TRUSTED == requestTrustIndicator) {
                updateAction.updatedTrustIndicator = TRUSTED;
                if (!isRequestAndDbDetailSame) {
                    updateAction.updateDetail = true;
                }
            } else if (UNTRUSTED == dbTrustIndicator && UNTRUSTED == requestTrustIndicator
                    && !isRequestAndDbDetailSame) {
                updateAction.updateDetail = true;
            } else if (TRUSTED == dbTrustIndicator && UNTRUSTED == requestTrustIndicator
                    && !isRequestAndDbDetailSame) {
                updateAction.updateDetail = true;
                updateAction.updatedTrustIndicator = UNTRUSTED;
            }
        }
        return updateAction;
    }

    private static String trimToNull(String target) {
        if (target == null) {
            return null;
        }
        String trimmed = target.trim();
        return "".equals(trimmed) ? null : trimmed;
    }

    private class UpdateAction {
        public boolean updateDetail;
        public TrustIndicator updatedTrustIndicator;
    }
}
{% endcodeblock %}

## 加测试

我们用Spock写了一个很*性感*的测试：（实际情况是，用jUnit写出来的测试实在是看不懂，木有办法了才用这个。。。。。。）

{% codeblock SummaryUpdateHelperTest.groovy https://github.com/YaodanZhang/code-kata/blob/master/src/test/groovy/com/thoughtworks/kata/refactor/SummaryUpdateHelperTest.groovy %}
class SummaryUpdateHelperTest extends Specification {

    @Unroll
    def "request[#requestSummary, #requestTrust] update db[#dbSummary, #dbTrust] = [#finalSummary, #finalTrust]"() {
        given:
        def helper = new SummaryUpdateHelper(newHashMap())
        def request = createRequest(requestSummary, requestTrust)
        def customer = createCustomer(dbSummary, dbTrust)

        when:
        helper.updateSummary(request, customer)

        then:
        customer.summary == createSummary(finalSummary, finalTrust)

        where:
        requestSummary | requestTrust | dbSummary | dbTrust   | finalSummary | finalTrust
        null           | null         | null      | null      | null         | null

        BLANK          | TRUSTED      | null      | null      | null         | null

        /********************************这里省掉58个测试用例*********************************/

        UPDATED        | UNTRUSTED    | ORIGINAL  | UNTRUSTED | UPDATED      | UNTRUSTED
    }

    /*************************************下面的不重要**************************************/
}
{% endcodeblock %}

## 重构

接下来，就是要做重构了。

在第一段代码中，我们一眼就能看到，这里面有一个很销魂的方法：`getUpdateAction()`。别的就略过啦，今天我们就来看看如何重构这个方法。

### 1. 干掉里面最二的代码
 
像下面这个：

``` java SummaryUpdateHelper.java
if (null == dbSummaryDetail || "".equals(dbSummaryDetail)) {
    isDbDetailBlank = true;
} else {
    isDbDetailBlank = false;
}
```

一行代码就搞定了：`isDbDetailBlank = isNullOrEmpty(dbSummaryDetail);`

简单来说，这一步完成过后，IDE里面就不会出现警告了。

### 2. 分离关注点

现在就需要动动脑筋了，首先我们可以看到，在方法`getUpdateAction()`中，他是把判断是否更新*Detail*和*Indicator*搅在一起了，想一次解决两个问题。但奇怪的是对于*Detail*，他返回的是是否需要更新；对于*Indicator*，他返回的是更新后的值。。。。。

我们可以首先把*Detail*和*Indicator*的判断分开。分开过后，我们就可以发现，以前专门用来保存*Detail*和*Indicator*的类`UpdateAction`就没有用了。我们先来看看抽出来后单独判断更新后的*Detail*的方法，关于*Indicator*的判断大家自己脑补。

{% codeblock SummaryUpdateHelper.java https://github.com/YaodanZhang/code-kata/blob/master/src/main/java/com/thoughtworks/kata/refactor/SummaryUpdateHelper.java %}
private boolean shouldUpdateDetail(String requestSummaryDetail, String dbSummaryDetail,
                                   TrustIndicator requestTrustIndicator,
                                   TrustIndicator dbTrustIndicator) {
    boolean isDbDetailBlank = isNullOrEmpty(dbSummaryDetail);
    boolean isRequestDetailBlank = isNullOrEmpty(requestSummaryDetail);
    boolean isRequestAndDbDetailSame = isSame(requestSummaryDetail, dbSummaryDetail);

    boolean shouldUpdateDetail = false;
    if (isRequestDetailBlank && !isDbDetailBlank && TRUSTED == requestTrustIndicator) {
        if (TRUSTED == dbTrustIndicator) {
            shouldUpdateDetail = true;
        } else if (UNTRUSTED == dbTrustIndicator) {
            shouldUpdateDetail = true;
        }
    } else if (!isRequestDetailBlank && isDbDetailBlank) {
        shouldUpdateDetail = true;
    } else if (!isRequestDetailBlank) {
        if (TRUSTED == dbTrustIndicator && TRUSTED == requestTrustIndicator
                && !isRequestAndDbDetailSame) {
            shouldUpdateDetail = true;
        } else if (UNTRUSTED == dbTrustIndicator && TRUSTED == requestTrustIndicator) {
            if (!isRequestAndDbDetailSame) {
                shouldUpdateDetail = true;
            }
        } else if (UNTRUSTED == dbTrustIndicator && UNTRUSTED == requestTrustIndicator
                && !isRequestAndDbDetailSame) {
            shouldUpdateDetail = true;
        } else if (TRUSTED == dbTrustIndicator && UNTRUSTED == requestTrustIndicator
                && !isRequestAndDbDetailSame) {
            shouldUpdateDetail = true;
        }
    }
    return shouldUpdateDetail;
}
{% endcodeblock %}

### 3. 重新理一下判断条件

从上面这个方法我们可以看到，他的逻辑判断也是很*二*的，很多分支可以合并或者简化，这一步就做这个事情。具体的过程就不一步一步的演示了，有兴趣的童鞋请在下面留言（我保证不及时回复）。

{% codeblock SummaryUpdateHelper.java https://github.com/YaodanZhang/code-kata/blob/master/src/main/java/com/thoughtworks/kata/refactor/SummaryUpdateHelper.java %}
private boolean shouldUpdateDetail(String requestSummaryDetail, String dbSummaryDetail,
                                   TrustIndicator requestTrustIndicator) {
    boolean isDbDetailBlank = isNullOrEmpty(dbSummaryDetail);

    if (isNullOrEmpty(requestSummaryDetail)) {
        return !isDbDetailBlank && TRUSTED == requestTrustIndicator;
    }

    return isDbDetailBlank || !isSame(requestSummaryDetail, dbSummaryDetail);
}

private TrustIndicator getUpdatedIndicator(String requestSummaryDetail, String dbSummaryDetail,
                                                TrustIndicator requestTrustIndicator,
                                                TrustIndicator dbTrustIndicator) {
    TrustIndicator updatedIndicator = dbTrustIndicator;

    boolean isDbDetailBlank = isNullOrEmpty(dbSummaryDetail);
    boolean isRequestDetailBlank = isNullOrEmpty(requestSummaryDetail);

    if (isRequestDetailBlank && !isDbDetailBlank && TRUSTED == requestTrustIndicator) {
        updatedIndicator = requestTrustIndicator;
    } else if (!isRequestDetailBlank && isDbDetailBlank) {
        updatedIndicator = requestTrustIndicator;
    } else if (!isRequestDetailBlank) {
        if (TRUSTED == requestTrustIndicator) {
            updatedIndicator = requestTrustIndicator;
        } else if (UNTRUSTED == requestTrustIndicator
                && !isSame(requestSummaryDetail, dbSummaryDetail)) {
            updatedIndicator = requestTrustIndicator;
        }
    }
    return updatedIndicator;
}
{% endcodeblock %}

有没有感觉瞬间清爽了不少？

### 4. 换一种思路去处理更新请求

我们看看这两个方法，可以发现他们其中的判断条件是很相似的。我们再考虑一下这两个方法的业务场景，无非就是判断是否要更新*Summary*，而如果*Summary*的*Detail*或者*Indicator*其中的任何一个更新了，另外一个最终的值肯定也是会被设为跟*Request*中的值一样的（如果*Request*中的值跟数据库中的一样，我们可以视为更新了，也可以视为没有更新）。所以我们换一种思路，不去判断是否需要更改，我们只判断最终，该请求执行完成后，*Summary*的值是否更请求的值一样即可。

按照这个思路去重构，首先可以把`shouldUpdateDetail()`这个方法的行为给改掉，然后再改掉`getUpdatedIndicator()`的行为，让他们俩行为一致，最终用一个新的方法`shouldUpdate()`。下面就是最终的方法，具体的重构过程大家有兴趣可以去看我*Github*的[提交记录](https://github.com/YaodanZhang/code-kata/tree/master/src/main/java/com/thoughtworks/kata/refactor)。

{% codeblock SummaryUpdateHelper.java https://github.com/YaodanZhang/code-kata/blob/master/src/main/java/com/thoughtworks/kata/refactor/SummaryUpdateHelper.java %}
private boolean shouldUpdate(String requestSummaryDetail, String dbSummaryDetail,
                             TrustIndicator requestTrustIndicator) {
    boolean isDbDetailBlank = isNullOrEmpty(dbSummaryDetail);
    boolean isRequestDetailBlank = isNullOrEmpty(requestSummaryDetail);

    if (isRequestDetailBlank) {
        return !isDbDetailBlank && TRUSTED == requestTrustIndicator;
    }

    return isDbDetailBlank || shouldUpdateWhenBothNotBlank(requestSummaryDetail,
            dbSummaryDetail, requestTrustIndicator);
}

private boolean shouldUpdateWhenBothNotBlank(String requestSummaryDetail,
                                             String dbSummaryDetail,
                                             TrustIndicator requestTrustIndicator) {
    return TRUSTED == requestTrustIndicator
            || UNTRUSTED == requestTrustIndicator && !isSame(requestSummaryDetail, dbSummaryDetail);
}
{% endcodeblock %}

到这里，能做的重构基本上就完成了，基本达到了能让一个健康人理解的程度（吧？）。

## 改掉这个Bug

### 1. 改测试

改Bug的第一步当然是改测试，把现在的测试用例中与新需求的部分改掉，也就帮我们明确了修改目标了。

回顾一下，我们新的需求是神马？

> 不要让untrusted的信息覆盖trusted

那么，我们就根据这个来修改我们的测试用例，发现下面的这些测试用例是需要修改的：

{% codeblock SummaryUpdateHelperTest.groovy https://github.com/YaodanZhang/code-kata/blob/master/src/test/groovy/com/thoughtworks/kata/refactor/SummaryUpdateHelperTest.groovy %}
...
where:
requestSummary | requestTrust | dbSummary | dbTrust   | finalSummary | finalTrust
BLANK          | TRUSTED      | null      | null      | BLANK        | TRUSTED
BLANK          | TRUSTED      | BLANK     | UNTRUSTED | BLANK        | TRUSTED
BLANK          | TRUSTED      | EMPTY     | UNTRUSTED | EMPTY        | TRUSTED

EMPTY          | TRUSTED      | null      | null      | BLANK        | TRUSTED
EMPTY          | TRUSTED      | BLANK     | UNTRUSTED | BLANK        | TRUSTED
EMPTY          | TRUSTED      | EMPTY     | UNTRUSTED | EMPTY        | TRUSTED

ORIGINAL       | UNTRUSTED    | BLANK     | TRUSTED   | BLANK        | TRUSTED
ORIGINAL       | UNTRUSTED    | EMPTY     | TRUSTED   | EMPTY        | TRUSTED

UPDATED        | UNTRUSTED    | BLANK     | TRUSTED   | BLANK        | TRUSTED
UPDATED        | UNTRUSTED    | EMPTY     | TRUSTED   | EMPTY        | TRUSTED
UPDATED        | UNTRUSTED    | ORIGINAL  | TRUSTED   | ORIGINAL     | TRUSTED
...
{% endcodeblock %}

### 2. 改实现代码

这里就是要改一个方法：`shouldUpdate()`。当*Request*中的值为空时，我们仅在其*Indicator*为*trusted*的时候更新数据库的值；否则，只要不是数据库为*trusted*且*Request*为*untrusted*这种情况，我们都应该更新数据库的值。有了这个理解，代码改起来就简单了：

{% codeblock SummaryUpdateHelper.java https://github.com/YaodanZhang/code-kata/blob/master/src/main/java/com/thoughtworks/kata/refactor/SummaryUpdateHelper.java %}
private boolean shouldUpdate(String requestSummaryDetail, TrustIndicator requestTrustIndicator,
                             TrustIndicator dbTrustIndicator) {
    if (isNullOrEmpty(requestSummaryDetail)) {
        return TRUSTED == requestTrustIndicator;
    }
    return !(TRUSTED == dbTrustIndicator && UNTRUSTED == requestTrustIndicator);
}
{% endcodeblock %}

这就是最终的结果，大家和之前的代码对比一下，是不是成功将一个50行的if-else块重构到了3行（除了那个无意义的反大括号）。

## 事后

这个一万行的类确实是真实存在于我们一个遗留系统中的，每次新的需求过来，如果涉及到要修改这个类，大家都分外的小心翼翼，因为没有单元测试，功能测试也没法覆盖所有的场景，导致每一行代码的修改造成的风险都是很高的。

渐渐的，在一次又一次的被坑了过后，我们总结出了一套修改这类代码的方法，这就是上面提到的步骤：

1. 找到需要修改的部分，把这个部分抽取出来
2. 根据业务场景和现有代码逻辑给这部分代码加测试（单元测试，端到端的功能测试等）
3. 重构
4. 修改
5. 重构
6. ...

改完这个，心情大好，想想，要是如果每天都能重构500行，那么这个一万行的类还是。。。。。。。。。。。。。。。。。。。。。。。。。。。。。。。。。。。。。。。。算了吧，能不改就别改了！