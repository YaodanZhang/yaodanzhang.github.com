---
layout: post
title: "在线系统数据&服务的迁移策略"
date: 2013-08-19 10:11
comments: true
categories: [技术杂谈]
---

当需要在正在运行的在线系统中进行数据或服务的迁移时，有很多问题需要考虑，如何设计迁移策略以保证数据正确迁移，如何处理系统间的依赖，如何保证服务持续可用等等。本文将从一个服务提供者的角度，讨论如何进行数据迁移才能保证对外提供的服务接口前后一致且持续可用，实现对于客户端的无缝迁移。

考虑一个简单的场景，有一个web应用，存有一定数量的用户信息，以前用户的密码都是用明文存在数据库里面的，作为一个有理想有道德的程序员，我现在想升级系统，对这些密码信息进行加密，我已经准备好了加密算法和相关的实现代码，现在问题来了，我要如何加密现有用户的密码呢？

<!--more -->

Solution 1：写个程序在后台进行加密转换，然后重启系统，用新的代码访问加密后的数据。

存在的问题：如果在转换的过程中有新的数据进来或对原有数据有修改，如何处理？如何保证加密前后的数据同步？

Solution 2：停止服务 --> 执行加密转换 --> 重启服务，用新的代码访问加密后的数据。

存在的问题：也许在开发机器上测试100遍，这个过程每次都不超过1分钟，但是根据“Showcase必挂”原理，在产品环境上做同样的事情很可能就要1个小时，甚至更多。如果我的系统1分钟能挣1万块钱，你还告诉我要这么搞么？

那么该如何进行原有数据的加密呢？

![Alt img](/images/migration/Slide1.jpg)

图1：未加密时的调用

![Alt img](/images/migration/Slide2.jpg)

图2：加密后的调用

我们的目标是从图1转换成图2，数据的迁移肯定是不能一下子就完成的，那么我们就需要一步一步的去实现这样的```DAO```调用的转换。

## Step 0：现有系统接口。

#### Service层：

* `UserService`，用户处理接口，对外服务API

* `UserServiceImpl`，`UserService`接口的实现，调用`UserDAO`

#### DAO层：

* `UserDAO`，用户数据访问接口

{% codeblock UserDAO.java %}
public interface UserDAO {
    User add(User user);

    User update(User user);

    User get(String id);

    boolean delete(String id);

    boolean existed(String username, String password);
}
{% endcodeblock %}

* `UserDAOImpl`，`UserDAO`接口针对未加密数据的实现

{% codeblock UserDAOImpl.java %}
public class UserDAOImpl implements UserDAO {
    private UserMapper userMapper;

    @Override
    public User add(User user) {
        return userMapper.insert(user);
    }

    @Override
    public User update(User user) {
        return userMapper.update(user);
    }

    @Override
    public User get(String id) {
        return userMapper.selectById(id);
    }

    @Override
    public boolean delete(String id) {
        return userMapper.deleteById(id);
    }

    @Override
    public boolean existed(String username, String password) {
        User user = userMapper.selectByUsername(username);
        return user != null && nullToEmpty(password).equals(user.getPassword());
    }
}
{% endcodeblock %}

## Step 1：数据库的设计。

假设以前我们的用户表有三个字段：`id`，`username`，`password`，那么我们实现的第一步是加一个字段`encryptedPassword`。

这样乍看起来数据库是有冗余的，但是我们如果直接修改`password`列中的内容，实质上就是删除了一个原有的列，再新加了一列，我们知道，步子迈得大了，是容易扯着蛋的，具体原因Solution 1/2 中已经讲了。

相应的，我们需要增加一个`UserDAO`接口针对加密数据的实现：`UserDAOEncryptedImpl`。

{% codeblock UserDAOEncryptedImpl.java %}
public class UserDAOEncryptedImpl implements UserDAO {
    private UserMapper userMapperForEncryption;

    @Override
    public User add(User user) {
        user.setPassword(encrypt(user.getPassword()));
        return userMapperForEncryption.insert(user);
    }

    @Override
    public User update(User user) {
        user.setPassword(encrypt(user.getPassword()));
        return userMapperForEncryption.update(user);
    }

    @Override
    public User get(String id) {
        return userMapperForEncryption.selectById(id);
    }

    @Override
    public boolean delete(String id) {
        return userMapperForEncryption.deleteById(id);
    }

    @Override
    public boolean existed(String username, String password) {
        User user = userMapperForEncryption.selectByUsername(username);
        return user != null && encrypt(nullToEmpty(password)).equals(user.getPassword());
    }
}
{% endcodeblock %}

`UserDAOImpl`与`UserDAOEncryptedImpl`的主要区别在于这几个方法：

其中，对于`password`来说，`add()`，`update()`和`delete()`方法是写数据，`existed()`方法是读数据。

## Step 2：增加一个组合模式DAO实现。

![Alt img](/images/migration/Slide3.jpg)

实现代码如下：

{% codeblock CompositeUserDAOImpl.java %}
public class CompositeUserDAOImpl implements UserDAO {
    private UserDAO userDAOWithoutEncryption = new UserDAOImpl();
 
    @Override
    public User add(User user) {
        return userDAOWithoutEncryption.add(user);
    }

    @Override
    public User update(User user) {
        return userDAOWithoutEncryption.update(user);
    }

    @Override
    public User get(String id) {
        return userDAOWithoutEncryption.get(id);
    }

    @Override
    public boolean delete(String id) {
        return userDAOWithoutEncryption.delete(id);
    }

    @Override
    public boolean existed(String username, String password) {
        return userDAOWithoutEncryption.existed(username, password);
    }
}
{% endcodeblock %}

将`UserServiceImpl`中的`userDAO`改成`CompositeUserDAOImpl`的实例。

当前状态下，该`CompositeUserDAOImpl`对`user`的读/写都操作未加密数据，使用`UserDAOImpl`来实现。

## Step 3：

![Alt img](/images/migration/Slide4.jpg)

修改`CompositeUserDAOImpl`，使其读数据仍操作未加密数据，但写数据同时修改加密和未加密两个列。

{% codeblock CompositeUserDAOImpl.java %}
public class CompositeUserDAOImpl implements UserDAO {
    private UserDAO userDAOWithoutEncryption = new UserDAOImpl();
    private UserDAO userDAOWithEncryption = new UserDAOEncryptedImpl();

    @Override
    public User add(User user) {
        userDAOWithEncryption.add(user.clone());
        return userDAOWithoutEncryption.add(user);
    }

    @Override
    public User update(User user) {
        userDAOWithEncryption.update(user.clone());
        return userDAOWithoutEncryption.update(user);
    }

    @Override
    public User get(String id) {
        return userDAOWithoutEncryption.get(id);
    }

    @Override
    public boolean delete(String id) {
        userDAOWithEncryption.delete(id);
        return userDAOWithoutEncryption.delete(id);
    }

    @Override
    public boolean existed(String username, String password) {
        return userDAOWithoutEncryption.existed(username, password);
    }
}
{% endcodeblock %}

## Step 4：开始进行数据迁移。

可以写个脚本在后台执行，将`password`列中的数据加密后存入`encryptedPassword`列。

## Step 5：

![Alt img](/images/migration/Slide5.jpg)

数据迁移完成后，修改`CompositeUserDAOImpl`，使其读数据从加密数据列中读取，写数据仍同时修改加密和未加密两个列。

{% codeblock CompositeUserDAOImpl.java %}
public class CompositeUserDAOImpl implements UserDAO {
    private UserDAO userDAOWithoutEncryption = new UserDAOImpl();
    private UserDAO userDAOWithEncryption = new UserDAOEncryptedImpl();

    @Override
    public User add(User user) {
        userDAOWithoutEncryption.add(user);
        return userDAOWithEncryption.add(user.clone());
    }

    @Override
    public User update(User user) {
        userDAOWithoutEncryption.update(user.clone());
        return userDAOWithEncryption.update(user);
    }

    @Override
    public User get(String id) {
        return userDAOWithEncryption.get(id);
    }

    @Override
    public boolean delete(String id) {
        userDAOWithoutEncryption.delete(id);
        return userDAOWithEncryption.delete(id);
    }

    @Override
    public boolean existed(String username, String password) {
        return userDAOWithEncryption.existed(username, password);
    }
}
{% endcodeblock %}

## Step 6：

![Alt img](/images/migration/Slide6.jpg)

上线运行一段时间后，当确保`password`列中的数据已经完全正确迁移并且没有其他的程序依赖与它，便可以将这一列移除了。`CompositeUserDAOImpl`中将只调用加密数据的`DAO`。

{% codeblock CompositeUserDAOImpl.java %}
public class CompositeUserDAOImpl implements UserDAO {
    private UserDAO userDAOWithEncryption = new UserDAOEncryptedImpl();

    @Override
    public User add(User user) {
        return userDAOWithEncryption.add(user.clone());
    }

    @Override
    public User update(User user) {
        return userDAOWithEncryption.update(user);
    }

    @Override
    public User get(String id) {
        return userDAOWithEncryption.get(id);
    }

    @Override
    public boolean delete(String id) {
        return userDAOWithEncryption.delete(id);
    }

    @Override
    public boolean existed(String username, String password) {
        return userDAOWithEncryption.existed(username, password);
    }
}
{% endcodeblock %}