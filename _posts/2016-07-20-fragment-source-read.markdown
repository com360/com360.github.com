---
layout: post
title:  "Fragment源码分析"
date:   2016-07-20 21:35:31 +0800
categories: jekyll update
---
## Fragment源码分析

> 在此博客之前，最好使用过Fragment或者对Fragment有一定的了解。
> 如果对Fragment不太了解，请参看官方使用帮助 [Building a Dynamic UI with Fragments](https://developer.android.com/training/basics/fragments/index.html ) 和 API guide [Fragment](https://developer.android.com/guide/components/fragments.html)。


### Fragment的生命周期

首先让大家先看一个Fragment的生命周期与Activity生命周期的关系。
此图来自于[Fragment完整的生命周期](https://github.com/xxv/android-lifecycle/blob/master/complete_android_fragment_lifecycle.png)

![Fragment完整的生命周期]({{ site.url }}/assets/complete_android_fragment_lifecycle.png)

从上图可以看出，Fragment的生命周期相当的复杂，在使用的过程中会有很多难以预料的问题。所以才有了 [Square的工程师为什么反对使用Fragment](https://corner.squareup.com/2014/10/advocating-against-android-fragments.html)；并且他们使用了MVP的结构，把Fragment的逻辑做到最轻，甚至不使用Fragment，View层放在自定义View里面，逻辑放在Presenter里面，这样可以把View进行组件化，也便于对Presenter进行单元测试。


### Fragment相关的核心类

Fragment相关的核心类，主要功能是负责Fragment的生命周期。

请看下面这张类关系图（画的不太规范）

![Fragment核心类关系图]({{ site.url }}/assets/fragment_most_classes.jpg)

* FragmentActivity: Fragment视图展示的容器，用于展示Fragment中的mView;还通过调用mFragments成员的方法把Activity的生命周期回调传递出去。

* FragmentController：FragmentActivity中的mFragments成员，负责把Activity的生命周期传递给自己的成员mHost。

* FragmentHostCallback： FragmentController中的mHost成员，负责把Activity的生命周期传递给自己的成员mFragmentManager。

* FragmentManager: FragmentManager会根据Activity的生命周期进行处理Fragment的生命周期，实现类是FragmentManagerImpl。

* FragmentTransaction：Fragment操作的事物对象，可以批量的对Fragment进行add, remove, replace, hien, show等操作，内部会创建一个循环链表，最终的Fragment生命周期的调用还是由FragmentManager进行管理。


除了核心流程之外还有数据的异常回复和保存等，暂时不在此讨论。

>需要注意的地方：
>
> 如果Activity没有执行到onResume()，Fragment的生命周期必须和Fragemnt保持对应关系；
>
> 如果Activity已经执行onResume(),处于活动状态，新add或者replace进去的会有自己的生命周期状态，从onCreate()一直执行到onResume()；


此分析基于com.android.support:support-v4:23.2.1
