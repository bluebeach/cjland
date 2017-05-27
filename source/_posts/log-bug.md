---
title: 一次日志打印问题的排查
date: 2016-04-04 02:23:08
tags: 
---
## 问题现象
测试服务器上无日志打印。

## 原因分析
第一次遇到这种情况，构建成功，部署成功，然后打日志毫无反应。

于是我开始怀疑人生...可是不是解决办法呀！

<!--more-->

于是我重新拉了一个崭新的线上分支部署到测试机上去，发现日志可以正常打印！

恩...仔细回想了一下我对开发分支的改动中...怀疑是因为修改了maven依赖导致日志的jar包冲突导致的。

跑到测试机上分别查询了加载的jar包，果然如此！

*正常日志依赖：*

![正常日志依赖](/images/good-lib.jpg)

*异常日志依赖：*

![异常日志依赖](/images/bad-lib.jpg)

对比得出，应该就是因为`logback-classic`和`logback-core`的依赖导致了日志打不出来。(该系统一直使用log4j日志框架)

## 问题解决
在项目根目录运行`mvn dependency:tree > tree`，打印maven依赖树。

排除掉所有对logback的直接和间接依赖即可。

**为了一劳永逸，直接在主pom.xml中dependencyManagement依赖logback的999-not-exist版本(空jar包)。**

## 深入分析
以上，问题的确解决了。

**But 究竟为什么logback包的依赖引入进来，日志就不打印了呢？**

为了解决这个疑惑，经过一番资料查询，结论是这样的：

**slf4j-log4j12.jar 与 logback-classic.jar 互斥，二者只能存在其一。**

## 详细拓展：
日志框架分为 **接口框架** 和 **实现框架**

接口框架包括 Apache Common Logging（之前叫 Jakarta Commons Logging，JCL）和SLF4J。

实现框架包括 log4j 和 logback。

*他们之间的关系如下图：*

![架构图](/images/log.jpg)

其中JCL-over-SLF4j的存在，是因为有些项目中已经采用了JCL架构的（因为出现的早）想要转换到SLF4J架构（因为性能高）而存在的一个桥接器。顾名思义，名字为 XXX-over-slf4j 表示将日志重定向到了slf4j中。

**由于JCL-over-SLF4J和原来的JCL具有完全相同的API，因此两者是不能共存的。排除JCL依赖的方法为将`commons-logging` 设置成`<scope>provided</scope>` 或者 依赖一个`99.0-does-not-exist`版本的`commons-logging`（一个空无一物的特殊jar包）**

具体的jar包调用关系为：（假设采用slf4j + log4j的组合）

```
commons-logging log
-> jcl-over-slf4j
-> slf4j-api
-> slf4j-log4j12
-> log4j
-> 输出日志
```

当既依赖了slf4j-log4j12，又依赖了logback-classic时，slf4j会产生让人非常迷惑的结果。

``` Java
LoggerFactory.getLogger(xxx.class);
```
跟进getLogger方法中

``` Java
ILoggerFactory iLoggerFactory = getILoggerFactory();
return iLoggerFactory.getLogger(name);
```
使用了getILoggerFactory()返回的工厂，跟进getILoggerFactory方法中

``` java
public static ILoggerFactory getILoggerFactory() {
  if (INITIALIZATION_STATE == UNINITIALIZED) {
    INITIALIZATION_STATE = ONGOING_INITIALIZATION;
    //初始化
    performInitialization();
  }
  switch (INITIALIZATION_STATE) {
    case SUCCESSFUL_INITIALIZATION:
      return StaticLoggerBinder.getSingleton().getLoggerFactory();
    case NOP_FALLBACK_INITIALIZATION:
      return NOP_FALLBACK_FACTORY;
    case FAILED_INITIALIZATION:
      throw new IllegalStateException(UNSUCCESSFUL_INIT_MSG);
    case ONGOING_INITIALIZATION:
      // support re-entrant behavior.
      // See also http://bugzilla.slf4j.org/show_bug.cgi?id=106
      return TEMP_FACTORY;
  }
  throw new IllegalStateException("Unreachable code");	
}
```
上面的代码中，先初始化，然后根据初始化的结果来选择LoggerFactory。进入performInitialization()方法中

``` java
private final static void performInitialization() {
 bind();
  if (INITIALIZATION_STATE == SUCCESSFUL_INITIALIZATION) {
    versionSanityCheck();
  }
}
```
初始化代码中，主要完成slf4j与具体的日志实现绑定的逻辑，进入bind方法中

``` java
Set staticLoggerBinderPathSet = findPossibleStaticLoggerBinderPathSet();
// 多绑定情况打印
reportMultipleBindingAmbiguity(staticLoggerBinderPathSet);
// 进行绑定
StaticLoggerBinder.getSingleton();
INITIALIZATION_STATE = SUCCESSFUL_INITIALIZATION;
// 打印真正的绑定实现
reportActualBinding(staticLoggerBinderPathSet);
emitSubstituteLoggerWarning();
```
在findPossibleStaticLoggerBinderPathSet()中，寻找`org/slf4j/impl/StaticLoggerBinder.class`接口的实现类，而slf4j-log4j21和logback-classic中都有对应的实现类，当两者都存在时，`reportActualBinding`的打印情况如下：

``` shell
SLF4J: Actual binding is of type [ch.qos.logback.classic.util.ContextSelectorStaticBinder]
```
	
`reportMultipleBindingAmbiguity`的打印如下：

``` c
SLF4J: Found binding in [jar:file:/home/admin/msggrab/target/msggrab.war/WEB-INF/lib/logback-classic-1.0.13.jar!/org/slf4j/impl/StaticLoggerBinder.class]
SLF4J: Found binding in [jar:file:/home/admin/msggrab/target/msggrab.war/WEB-INF/lib/slf4j-log4j12-1.7.5.jar!/org/slf4j/impl/StaticLoggerBinder.class]
SLF4J: See http://www.slf4j.org/codes.html#multiple_bindings for an explanation.
```

### 而我之所以打不出日志，是因为最终选择了logback作为了`StaticLoggerBinder.class`的实现，如下图
![debug](/images/debug.jpg)

**可是，为什么当slf4j-log4j12和logback都存在的情况下，会优先使用logback呢？？？**

...






















