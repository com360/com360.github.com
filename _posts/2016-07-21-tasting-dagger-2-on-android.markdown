---
layout: post
title:  "dagger2 解析(翻译)"
date:   2016-07-21 21:35:31 +0800
categories: jekyll update
---
## dagger2 解析

[原文地址](http://fernandocejas.com/2015/04/11/tasting-dagger-2-on-android/)

Hi, 是时候决定去写关于最新一周所做的事情的博客了。希望去讨论一些关于我使用[Dagger2](http://google.github.io/dagger/)的一些经验。不过，我首先需要简单扼要的解释一下依赖注入在android开发中的重要性。

在阅读本片博客之前，假设你已经了解一些关于[依赖注入](https://en.wikipedia.org/wiki/Dependency_injection)的基本知识，比如像[Dagger](http://square.github.io/dagger/) \ [Guice](https://github.com/google/guice)这样的依赖注入工具。如果没有的话，最好先阅读一下这篇[优秀的文章](http://antonioleiva.com/dependency-injection-android-dagger-part-1/)http://antonioleiva.com/dependency-injection-android-dagger-part-1/。让我们甩开膀子开干吧！

### 为什么要使用依赖注入

首先我们已经用了很长时间的控制反转原则，程序的执行基于程序执行时的对象图（对象关系图），这种动态程序执行流程成为可能是因为通过抽象实现的。这种运行时绑定是基于依赖反转或者服务定位者（service locator）实现的。

依赖注入能给我们带来很多好处：

* 依赖可以通过注入或者外部配置，这样可以做到组件重用。

* 当协作对象以抽象的方式注入时，我们只需要更换实现类，为不用大面积改动代码。因此，对象实例实现了解耦和分离。

* 依赖可以被注入到一个组件中，使得在测试的时候可以注入mock数据，方便测试。

另一方面，我们可以控制创建的对象的使用范围，在我看来是相当的牛B。对象的创建和生命周期你不需要关心，它们由依赖注入框架管理。

![]({{ site.url }}/assets/dependency_inversion1.png)

### JSR-330 是什么

[java的依赖注入](https://jcp.org/en/jsr/detail?id=330)定义了一套使用注解注入对象的标准,这套标准为了最大可能使得java代码可被重用，测试和维护。

Dagger1 和 Dagger2，还有Guice，都是基于这套标准实现的。提供了持续的和标准的方式实现依赖注入。

### Dagger1

我会很快的过一遍Dagger1，因为不是本篇文章的主题。Dagger1是现在android上最流行的依赖注入框架。Square受到Guice的启发而创建了Dagger。

Dagger1的基本原理：

* 多个注入点：依赖，通过injected

* 多种绑定方法：依赖，通过provided

* 多个modules：实现某种功能的绑定集合

* 多个对象图： 实现同一个领域的modules集合

Dagger1编译器指定绑定关系，也使用反射，但只用于构建组件关系。Dagger在运行时指出所有组件是否合适，因此会有一定开销：有时是低效的，也不易于调试。

### Dagger 2

[Dagger2](http://google.github.io/dagger/) 从Dagger1 fork出来的一个分支，当前版本是2.0，由google进行开发。它受到AutoValue 项目的启发（https://github.com/google/auto，如果想随时写equals和hashcode方法，将会很有用）。

刚开始Dagger2的思想是，使用代码自动生成解决依赖问题，就像我们手写代码一样。

和之前的版本相比，两个版本有很多相似之处，也有很重要的不同需要注意一下：

* 没有使用任何反射： 关系校验，配置和先决条件都是在编译时完成。

* 方便调式和完整跟踪：提供和生产完整详实的调用栈。

* 更优的性能： google提供的数据提高13%的性能。

* 代码覆盖：就像手写代码一样使用方法调度。

当然，有这些出色的特征需要付出一定代价的，导致灵活度降低了：举个例子，因为没有使用反射，所以缺乏动态性。

### 深入分析

了解依赖注入的基本概念，是理解Dagger2的重要条件。如果不理解也没关系，后面会有例子说明。

* @Inject ： 使用这个注解表达我们的依赖请求。换句话说，使用这个注解告诉Dagger，它注解的类或者属性希望参与依赖注入。因此，Daager会构建注解的类对象去完成它的依赖。
 
* @Module : Module 是那些方法提供依赖的类，我可以定义一个类，使用 @Module 进行注解，因此，在创建对象的时候Dagger 将会知道去哪里去寻找对象的依赖。Module 的一个重要特征是，它们设计的时候被切分，然后再组装到一起。

* @Provide ： 在module内部，我在定义的方法上使用此注解，它告诉Dagger 我希望如何构建和提供需要的依赖。

* @Component: Components是注入器，它是连接 @Inject 和 @Module的桥梁。Component仅仅是提供你定义的所有对象的实例，比如，我定义的一个接口使用 @Component进行注解，并列出所有组成他的 @Modules，如果任何一个Module缺失，在编译时会报错。所有的component 能够通过module检测依赖范围。

* @Scope ：

* @Qualifier:

### 无代码无真相

现在已经说了太多的理论上的东西，需要实践Dagger2了。首先在程序的build.gradle文件中增加Dagger的依赖。

build.gradle

***********************************

<pre>
<code>

apply plugin: 'com.neenbedankt.android-apt'

buildscript {
  repositories {
    jcenter()
  }
  dependencies {
    classpath 'com.neenbedankt.gradle.plugins:android-apt:1.4'
  }
}

android {
  ...
}

dependencies {
  apt 'com.google.dagger:dagger-compiler:2.0'
  compile 'com.google.dagger:dagger:2.0'
  provided 'javax.annotation:jsr250-api:1.0' 
  
  ...
}
</code>
</pre>

我们可以看到，我增加了javax 注解库，编译和运行时的库，还有apt 插件。不加入的话，Dagger将无法正常工作。

### 例子

几个月前我写了一篇关于[如何在android上实现 uncle bob的 clean architecture](http://fernandocejas.com/2014/09/03/architecting-android-the-clean-way/)的文章，我非常强烈建议你先阅读这篇文章，这非常有利于理解我在这里所做的事。当时，我面临解决一个关于依赖对象的问题，就像下面这样，重点看注释：

UserDetailsFragment.java

**********************************
<pre>
<code>

@Override void initializePresenter() {
    // 所有的这些初始化都可以通过使用依赖注入框架，但是这个里只是一个例子而已。（大写的无奈）
    
    ThreadExecutor threadExecutor = JobExecutor.getInstance();
    PostExecutionThread postExecutionThread = UIThread.getInstance();
 
    JsonSerializer userCacheSerializer = new JsonSerializer();
    UserCache userCache = UserCacheImpl.getInstance(getActivity(), userCacheSerializer,
        FileManager.getInstance(), threadExecutor);
    UserDataStoreFactory userDataStoreFactory =
        new UserDataStoreFactory(this.getContext(), userCache);
    UserEntityDataMapper userEntityDataMapper = new UserEntityDataMapper();
    UserRepository userRepository = UserDataRepository.getInstance(userDataStoreFactory,
        userEntityDataMapper);
 
    GetUserDetailsUseCase getUserDetailsUseCase = new GetUserDetailsUseCaseImpl(userRepository,
        threadExecutor, postExecutionThread);
    UserModelDataMapper userModelDataMapper = new UserModelDataMapper();
 
    this.userDetailsPresenter =
        new UserDetailsPresenter(this, getUserDetailsUseCase, userModelDataMapper);
  }

</code>
</pre>

正如你所看到的，使用依赖注入框架可以解决这个问题。我们要摆脱上面的代码：这个类不需要知道对象如何创建和如何提供依赖的。

如何去实现呢？ 当然使用Dagger2的功能了... 下图是依赖注入的关系图：

![]({{ site.url }}/assets/composed_dagger_graph1.png)

咱们需要分解一下这张图并解释每个部分。

Application Component: component 的生命周期就是application的生命周期. 它同时注入AndroidApplication 和 BaseActivity 类。

ApplicationComponent.java

*****************************

<pre>
<code>
@Singleton // Constraints this component to one-per-application or unscoped bindings.
@Component(modules = ApplicationModule.class)
public interface ApplicationComponent {
  void inject(BaseActivity baseActivity);
 
  //Exposed to sub-graphs.
  Context context();
  ThreadExecutor threadExecutor();
  PostExecutionThread postExecutionThread();
  UserRepository userRepository();
}
</code>
</pre>

可以看出，为此 Component 使用 @Singleton 注解, 限制了 Component和 application一对一的关系。


