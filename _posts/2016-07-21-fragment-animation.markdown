---
layout: post
title:  "Fragment动画分析"
date:   2016-07-21 21:35:31 +0800
categories: jekyll update
---
## Fragment 动画分析

> 在此博客之前，最好使用过Fragment或者对Fragment有一定的了解。
> 如果对Fragment不太了解，请参看官方使用帮助 [Building a Dynamic UI with Fragments](https://developer.android.com/training/basics/fragments/index.html ) 和 API guide [Fragment](https://developer.android.com/guide/components/fragments.html)。


### support-v4 中的 Fragment 动画分析

#### 如何使得Framgent之间有转场动画

> 此分析基于com.android.support:support-v4:23.2.1

  要想Fragment之间的跳转有转场动画的用户比较简单：
  
```java
getSupportFragmentManager().beginTransaction()
// Replace the default fragment animations with animator resources
// representing rotations when switching to the back of the card,as
// well as animator resources representing rotations when flipping
// back to the front (e.g. when the system Back button is pressed).
.setCustomAnimations(
        R.animator.slide_right_in,
        R.animator.slide_right_out,
        R.animator.slide_left_in,
        R.animator.slide_left_out)
.replace(R.id.container, new NextFragment())

// Add this transaction to the back stack, allowing users to press
// Back to get to the front of the card.
.addToBackStack(null)

// Commit the transaction.
.commit();

```

动画只需要在xml文件中定义即可。

v4中的Fragment到底支持什么动画类型？

在FragmentManager中的FragmentImpl类中的void moveToState(Fragment f, int newState, int transit, int transitionStyle,
            boolean keepActive)方法中的一个代码片段：
            
```java

f.mContainer = container;
f.mView = f.performCreateView(f.getLayoutInflater(
        f.mSavedFragmentState), container, f.mSavedFragmentState);
if (f.mView != null) {
    f.mInnerView = f.mView;
    if (Build.VERSION.SDK_INT >= 11) {
        ViewCompat.setSaveFromParentEnabled(f.mView, false);
    } else {
        f.mView = NoSaveStateFrameLayout.wrap(f.mView);
    }
    if (container != null) {
        Animation anim = loadAnimation(f, transit, true,
                                            transitionStyle);
        if (anim != null) {
            setHWLayerAnimListenerIfAlpha(f.mView, anim);
            f.mView.startAnimation(anim);
        }
        container.addView(f.mView);
    }
    if (f.mHidden) f.mView.setVisibility(View.GONE);
        f.onViewCreated(f.mView, f.mSavedFragmentState);
    } else {
        f.mInnerView = null;
    }
}

```         

加载动画对象的关键代码：

```java
Animation anim = loadAnimation(f, transit, true, transitionStyle);	
```

看下loadAnimation(f, transit, true, transitionStyle)方法：

```java
	Animation loadAnimation(Fragment fragment, int transit, boolean enter,
            int transitionStyle) {
    Animation animObj = fragment.onCreateAnimation(transit, enter,
            fragment.mNextAnim);
    if (animObj != null) {
        return animObj;
    }
        
    if (fragment.mNextAnim != 0) {
        Animation anim = AnimationUtils.loadAnimation(mHost.getContext(), fragment.mNextAnim);
        if (anim != null) {
            return anim;
        }
    }
        
    if (transit == 0) {
        return null;
    }
        
    int styleIndex = transitToStyleIndex(transit, enter);
    if (styleIndex < 0) {
        return null;
    }

    switch (styleIndex) {
        case ANIM_STYLE_OPEN_ENTER:
            return makeOpenCloseAnimation(mHost.getContext(), 1.125f, 1.0f, 0, 1);
        case ANIM_STYLE_OPEN_EXIT:
            return makeOpenCloseAnimation(mHost.getContext(), 1.0f, .975f, 1, 0);
        case ANIM_STYLE_CLOSE_ENTER:
            return makeOpenCloseAnimation(mHost.getContext(), .975f, 1.0f, 0, 1);
        case ANIM_STYLE_CLOSE_EXIT:
            return makeOpenCloseAnimation(mHost.getContext(), 1.0f, 1.075f, 1, 0);
        case ANIM_STYLE_FADE_ENTER:
            return makeFadeAnimation(mHost.getContext(), 0, 1);
        case ANIM_STYLE_FADE_EXIT:
            return makeFadeAnimation(mHost.getContext(), 1, 0);
    }
        
    if (transitionStyle == 0 && mHost.onHasWindowAnimations()) {
        transitionStyle = mHost.onGetWindowAnimations();
    }
    if (transitionStyle == 0) {
       return null;
    }
        
    //TypedArray attrs = mActivity.obtainStyledAttributes(transitionStyle,
    //        com.android.internal.R.styleable.FragmentAnimation);
    //int anim = attrs.getResourceId(styleIndex, 0);
    //attrs.recycle();
        
    //if (anim == 0) {
    //    return null;
    //}
        
    //return AnimatorInflater.loadAnimator(mActivity, anim);
    return null;
```

Fragment的这个方法默认实现为空fragment.onCreateAnimation(transit, enter, fragment.mNextAnim);
如果实现为空的话，返回null，会执行到

Animation anim = AnimationUtils.loadAnimation(mHost.getContext(), fragment.mNextAnim);


如果此方法仍然返回null，会执行到

```java
	int styleIndex = transitToStyleIndex(transit, enter);
    if (styleIndex < 0) {
        return null;
    }

    switch (styleIndex) {
        case ANIM_STYLE_OPEN_ENTER:
            return makeOpenCloseAnimation(mHost.getContext(), 1.125f, 1.0f, 0, 1);
        case ANIM_STYLE_OPEN_EXIT:
            return makeOpenCloseAnimation(mHost.getContext(), 1.0f, .975f, 1, 0);
        case ANIM_STYLE_CLOSE_ENTER:
            return makeOpenCloseAnimation(mHost.getContext(), .975f, 1.0f, 0, 1);
        case ANIM_STYLE_CLOSE_EXIT:
            return makeOpenCloseAnimation(mHost.getContext(), 1.0f, 1.075f, 1, 0);
        case ANIM_STYLE_FADE_ENTER:
            return makeFadeAnimation(mHost.getContext(), 0, 1);
        case ANIM_STYLE_FADE_EXIT:
            return makeFadeAnimation(mHost.getContext(), 1, 0);
    }
```
通过getSupportFragmentManager().beginTransaction().setTransitionStyle(@StyleRes int styleRes)设置的。


#### 使用Fragment中遇到的问题

java.lang.RuntimeException: Unknown animation name: objectAnimator

此问题是因为在定义动画的xml的使用使用了<objectAnimator>标签，v4中的Fragment不支持属性动画。

因为V4是支持api 4以上的手机，属性动画api 11才支持。v4中并没有加入属性动画的支持。所以才会报错。

```java
	if (fragment.mNextAnim != 0) {
        Animation anim = AnimationUtils.loadAnimation(mHost.getContext(), fragment.mNextAnim);
        if (anim != null) {
            return anim;
        }
    }
```

最终为会执行到AnimationUtils的createAnimationFromXml方法

```java
private static Animation createAnimationFromXml(Context c, XmlPullParser parser,
            AnimationSet parent, AttributeSet attrs) throws XmlPullParserException, IOException {

        Animation anim = null;

        // Make sure we are on a start tag.
        int type;
        int depth = parser.getDepth();

        while (((type=parser.next()) != XmlPullParser.END_TAG || parser.getDepth() > depth)
               && type != XmlPullParser.END_DOCUMENT) {

            if (type != XmlPullParser.START_TAG) {
                continue;
            }

            String  name = parser.getName();

            if (name.equals("set")) {
                anim = new AnimationSet(c, attrs);
                createAnimationFromXml(c, parser, (AnimationSet)anim, attrs);
            } else if (name.equals("alpha")) {
                anim = new AlphaAnimation(c, attrs);
            } else if (name.equals("scale")) {
                anim = new ScaleAnimation(c, attrs);
            }  else if (name.equals("rotate")) {
                anim = new RotateAnimation(c, attrs);
            }  else if (name.equals("translate")) {
                anim = new TranslateAnimation(c, attrs);
            } else {
                throw new RuntimeException("Unknown animation name: " + parser.getName());
            }

            if (parent != null) {
                parent.addAnimation(anim);
            }
        }

        return anim;

    }
```

也就是说，只支持<set> <alpha> <scale> <rotate> <translate> 标签

有的开发者在使用属性动画的时候遇到如下问题：

java.lang.RuntimeException: Unknown animation name: objectAnimator


比如：[stackoverflow中的这个问题](http://stackoverflow.com/questions/16688900/fragment-unknown-animation-name-objectanimator)

```java

05-22 11:32:34.706: E/AndroidRuntime(6801): FATAL EXCEPTION: main
05-22 11:32:34.706: E/AndroidRuntime(6801): java.lang.RuntimeException: Unknown animation name: objectAnimator
05-22 11:32:34.706: E/AndroidRuntime(6801):     at android.view.animation.AnimationUtils.createAnimationFromXml(AnimationUtils.java:124)
05-22 11:32:34.706: E/AndroidRuntime(6801):     at android.view.animation.AnimationUtils.createAnimationFromXml(AnimationUtils.java:114)
05-22 11:32:34.706: E/AndroidRuntime(6801):     at android.view.animation.AnimationUtils.createAnimationFromXml(AnimationUtils.java:91)
05-22 11:32:34.706: E/AndroidRuntime(6801):     at android.view.animation.AnimationUtils.loadAnimation(AnimationUtils.java:72)
05-22 11:32:34.706: E/AndroidRuntime(6801):     at android.support.v4.app.FragmentManagerImpl.loadAnimation(FragmentManager.java:710)
05-22 11:32:34.706: E/AndroidRuntime(6801):     at android.support.v4.app.FragmentManagerImpl.hideFragment(FragmentManager.java:1187)
05-22 11:32:34.706: E/AndroidRuntime(6801):     at android.support.v4.app.BackStackRecord.run(BackStackRecord.java:610)
05-22 11:32:34.706: E/AndroidRuntime(6801):     at android.support.v4.app.FragmentManagerImpl.execPendingActions(FragmentManager.java:1431)
05-22 11:32:34.706: E/AndroidRuntime(6801):     at android.support.v4.app.FragmentManagerImpl$1.run(FragmentManager.java:420)
05-22 11:32:34.706: E/AndroidRuntime(6801):     at android.os.Handler.handleCallback(Handler.java:725)
05-22 11:32:34.706: E/AndroidRuntime(6801):     at android.os.Handler.dispatchMessage(Handler.java:92)
05-22 11:32:34.706: E/AndroidRuntime(6801):     at android.os.Looper.loop(Looper.java:137)
05-22 11:32:34.706: E/AndroidRuntime(6801):     at android.app.ActivityThread.main(ActivityThread.java:5041)
05-22 11:32:34.706: E/AndroidRuntime(6801):     at java.lang.reflect.Method.invokeNative(Native Method)
05-22 11:32:34.706: E/AndroidRuntime(6801):     at java.lang.reflect.Method.invoke(Method.java:511)
05-22 11:32:34.706: E/AndroidRuntime(6801):     at com.android.internal.os.ZygoteInit$MethodAndArgsCaller.run(ZygoteInit.java:793)
05-22 11:32:34.706: E/AndroidRuntime(6801):     at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:560)
05-22 11:32:34.706: E/AndroidRuntime(6801):     at dalvik.system.NativeStart.main(Native Method)

```

#### 如何使用属性动画
 
  修改v4中的实现，github中有一个实现可以参考：[kedzie/Support_v4_NineOldAndroids](https://github.com/kedzie/Support_v4_NineOldAndroids)

#### 不使用属性动画，使用自定义复杂动画

从上面的分析可以看出：

```java
		
Animation animObj = fragment.onCreateAnimation(transit, enter, fragment.mNextAnim);
        
if (animObj != null) {
    return animObj;
}
```

只要实现Fragment的onCreateAnimation(transit, enter, fragment.mNextAnim)方法，返回一个Animation对象即可。
通过enter和fragment.mNextAnim构造Animation对象。

我在线上项目中就是使用此方法实现了一些3D动画的效果。


### Framework 中的 Fragment 动画分析

使用方法参考官方文档[Displaying Card Flip Animations](https://developer.android.com/training/animation/cardflip.html)

加载动画的方法有所简化：

```java
f.mContainer = container;
.mView = f.performCreateView(f.getLayoutInflater(
        f.mSavedFragmentState), container, f.mSavedFragmentState);
if (f.mView != null) {
    f.mView.setSaveFromParentEnabled(false);
    if (container != null) {
    Animator anim = loadAnimator(f, transit, true,
                transitionStyle);
    if (anim != null) {
        anim.setTarget(f.mView);
        setHWLayerAnimListenerIfAlpha(f.mView, anim);
                anim.start();
    }
    container.addView(f.mView);
}
if (f.mHidden) f.mView.setVisibility(View.GONE);
    f.onViewCreated(f.mView, f.mSavedFragmentState);
}
```

```java

	Animator loadAnimator(Fragment fragment, int transit, boolean enter,
            int transitionStyle) {
        Animator animObj = fragment.onCreateAnimator(transit, enter,
                fragment.mNextAnim);
        if (animObj != null) {
            return animObj;
        }
        
        if (fragment.mNextAnim != 0) {
            Animator anim = AnimatorInflater.loadAnimator(mHost.getContext(), fragment.mNextAnim);
            if (anim != null) {
                return anim;
            }
        }
        
        if (transit == 0) {
            return null;
        }
        
        int styleIndex = transitToStyleIndex(transit, enter);
        if (styleIndex < 0) {
            return null;
        }
        
        if (transitionStyle == 0 && mHost.onHasWindowAnimations()) {
            transitionStyle = mHost.onGetWindowAnimations();
        }
        if (transitionStyle == 0) {
            return null;
        }
        
        TypedArray attrs = mHost.getContext().obtainStyledAttributes(transitionStyle,
                com.android.internal.R.styleable.FragmentAnimation);
        int anim = attrs.getResourceId(styleIndex, 0);
        attrs.recycle();
        
        if (anim == 0) {
            return null;
        }
        
        return AnimatorInflater.loadAnimator(mHost.getContext(), anim);
    }
```

会执行到这里：

```java
private static Animator createAnimatorFromXml(Resources res, Theme theme, XmlPullParser parser,
            AttributeSet attrs, AnimatorSet parent, int sequenceOrdering, float pixelSize)
            throws XmlPullParserException, IOException {
        Animator anim = null;
        ArrayList<Animator> childAnims = null;

        // Make sure we are on a start tag.
        int type;
        int depth = parser.getDepth();

        while (((type = parser.next()) != XmlPullParser.END_TAG || parser.getDepth() > depth)
                && type != XmlPullParser.END_DOCUMENT) {

            if (type != XmlPullParser.START_TAG) {
                continue;
            }

            String name = parser.getName();
            boolean gotValues = false;

            if (name.equals("objectAnimator")) {
                anim = loadObjectAnimator(res, theme, attrs, pixelSize);
            } else if (name.equals("animator")) {
                anim = loadAnimator(res, theme, attrs, null, pixelSize);
            } else if (name.equals("set")) {
                anim = new AnimatorSet();
                TypedArray a;
                if (theme != null) {
                    a = theme.obtainStyledAttributes(attrs, R.styleable.AnimatorSet, 0, 0);
                } else {
                    a = res.obtainAttributes(attrs, R.styleable.AnimatorSet);
                }
                anim.appendChangingConfigurations(a.getChangingConfigurations());
                int ordering = a.getInt(R.styleable.AnimatorSet_ordering, TOGETHER);
                createAnimatorFromXml(res, theme, parser, attrs, (AnimatorSet) anim, ordering,
                        pixelSize);
                a.recycle();
            } else if (name.equals("propertyValuesHolder")) {
                PropertyValuesHolder[] values = loadValues(res, theme, parser,
                        Xml.asAttributeSet(parser));
                if (values != null && anim != null && (anim instanceof ValueAnimator)) {
                    ((ValueAnimator) anim).setValues(values);
                }
                gotValues = true;
            } else {
                throw new RuntimeException("Unknown animator name: " + parser.getName());
            }

            if (parent != null && !gotValues) {
                if (childAnims == null) {
                    childAnims = new ArrayList<Animator>();
                }
                childAnims.add(anim);
            }
        }
        if (parent != null && childAnims != null) {
            Animator[] animsArray = new Animator[childAnims.size()];
            int index = 0;
            for (Animator a : childAnims) {
                animsArray[index++] = a;
            }
            if (sequenceOrdering == TOGETHER) {
                parent.playTogether(animsArray);
            } else {
                parent.playSequentially(animsArray);
            }
        }
        return anim;
    }

```

它使用了AnimatorInflater.loadAnimator(mHost.getContext(), fragment.mNextAnim)加载动画：

返回的是Animator对象，而不是Animation。所以Framework中是完美支持属性动画的。

其实v4中自己可以扩展一下，可以同时支持Animation和Animator，现在的应用都一般只支持4.0以上，所以还是比较容易扩展的。




