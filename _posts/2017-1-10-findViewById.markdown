---
title: 2017-1-10findViewById
layout:     post
#subtitle:   " 常用链接导航"
date:       2017-1-10 20:00:00
author:     "图乐"
#header-img: ""
catalog: true
tags:
    - android日记
    - Android
    - findViewById
---
欢迎访问[Android日记][1],如有转载请注明Android日记 http://androiddiary.site
2017.1.10 周二 晴 临沂
---

### 写日记

###  从findViewById()说
还记得三年之前学习android就开始使用findViewById，但是用了这么久还没有具体分析过他的实现，今天终于有机会仔细看看。
1. 在Activity中找到findViewById方法
```
/**
 * Finds a view that was identified by the id attribute from the XML that
 * was processed in {@link #onCreate}.
 *
 * @return The view if found or null otherwise.
 */
public View findViewById(@IdRes int id) {
    return getWindow().findViewById(id);
}
```

2. getWindow：
```
public Window getWindow() {
    return mWindow;  
}
```
mWindow 是 private Window mWindow; 是一个Window类型的变量

3. 再看Window类中的findViewById方法
```
/**
 * Finds a view that was identified by the id attribute from the XML that
 * was processed in {@link android.app.Activity#onCreate}.  This will
 * implicitly call {@link #getDecorView} for you, with all of the
 * associated side-effects.
 *
 * @return The view if found or null otherwise.
 */
@Nullablepublic View findViewById(@IdRes int id) {
    return getDecorView().findViewById(id);
}
```
4. 发现getDecorView()是Window中的一个抽象方法。而Window唯一的子类是PhoneWindow.

5. 在PhoneWindow找到getDecorView()方法
```
@Override
public final View getDecorView() {
    if (mDecor == null || mForceDecorInstall) {
        installDecor();
    }
    return mDecor;
}
```
而mDecor是Phone的一个内部类DecorView
```
private final class DecorView extends FrameLayout implements RootViewSurfaceTaker 
```
它继承了FrameLayout ，而FrameLayout 是ViewGroup的子类
6. 最终我们在View类中找到了
```
/**
 * Look for a child view with the given id.  If this view has the given
 * id, return this view.
 *
 * @param id The id to search for.
 * @return The view that has the given id in the hierarchy or null
 */
@Nullablepublic final View findViewById(@IdRes int id) {
    if (id < 0) {
        return null;
    }
    return findViewTraversal(id);
}
```
在View类的findViewById中调用的方法findViewTraversal只在ViewGroup里面被重写（Override）了。
我们先看一下ViewGroup中的实现（因为mDecor是ViewGroup的子类会调用ViewGroup的方法）
```
@Override
protected View findViewTraversal(@IdRes int id) {
    if (id == mID) {
        return this;
    }
    final View[] where = mChildren;
    final int len = mChildrenCount;
    for (int i = 0; i < len; i++) {
        View v = where[i];
        if ((v.mPrivateFlags & PFLAG_IS_ROOT_NAMESPACE) == 0) {
            v = v.findViewById(id);
            if (v != null) {
                return v;
            }
        }
    }
    return null;
}
```
简单的说如果当前ViewGroup（或者其子类的）mID == id 为true，也就找到了要查找了View，如果不相等则会去遍历ViewGroup的子View数组，如果在子View的找到就返回；如果遍历完所有的子View都没有找到则返回null。这里的遍历类似于树的先序遍历，
我们在看一下在View类中的实现
```
protected View findViewTraversal(@IdRes int id) {
    if (id == mID) {
        return this;
    }
    return null;
}
```
View类似于树的叶子节点，它没有子布局（节点），所以他的遍历很简单，就是简单比较id是否相等，相等就返回当前View，不相等则返回null;
---
###  简单总结
使用findViewById时，最终会调用ViewGroup中的findViewTraversal，这个方法会遍历所有的子View，形成一个递归查询，找到最末端（View中）。如果找到就会返回这个View并停止查询，如果没找到就会返回为null。在确定要查找的那个View在某个View中的时候，我们调用那个View.findViewById()方法，会减少查询的循环次数，提高效率。