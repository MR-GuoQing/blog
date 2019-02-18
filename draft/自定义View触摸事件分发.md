---
layout:     post
title:      View触摸事件分发机制源码解析
subtitle:    ""
date:       2018-12-5 14:20
author:     guoqing
header-img: img/posts/post-bg-hacker.jpg
catalog: true
tags:
    - Android
    - Android 自定义View 事件分发
---
### View触摸事件分发机制源码解析
Android中Activity是负责所有与用户交互的事务，所有可视界面的创建都是从Activity开始。因此view的触摸事件的处理也是从Activity开始的，其开始处理触摸事件的方法如下所示：
```java
/**
//from Activity.java

    * Called to process touch screen events.  You can override this to
    * intercept all touch screen events before they are dispatched to the
    * window.  Be sure to call this implementation for touch screen events
    * that should be handled normally.
    *
    * @param ev The touch screen event.
    *
    * @return boolean Return true if this event was consumed.
    */
   public boolean dispatchTouchEvent(MotionEvent ev) {
       if (ev.getAction() == MotionEvent.ACTION_DOWN) {
           onUserInteraction();
       }
       if (getWindow().superDispatchTouchEvent(ev)) {
           return true;
       }
       return onTouchEvent(ev);
   }

```
dispatchTouchEvent方法中所有的down事件都会先由onUserInteraction()处理，方法如下所示：
```java
//from Activity.java
/**
     * Called whenever a key, touch, or trackball event is dispatched to the
     * activity.  Implement this method if you wish to know that the user has
     * interacted with the device in some way while your activity is running.
     * This callback and {@link #onUserLeaveHint} are intended to help
     * activities manage status bar notifications intelligently; specifically,
     * for helping activities determine the proper time to cancel a notfication.
     *
     * <p>All calls to your activity's {@link #onUserLeaveHint} callback will
     * be accompanied by calls to {@link #onUserInteraction}.  This
     * ensures that your activity will be told of relevant user activity such
     * as pulling down the notification pane and touching an item there.
     *
     * <p>Note that this callback will be invoked for the touch down action
     * that begins a touch gesture, but may not be invoked for the touch-moved
     * and touch-up actions that follow.
     *
     * @see #onUserLeaveHint()
     */
    public void onUserInteraction() {
    }

```
从注释中可以看出onUserInteraction需要由用户自己实现，其目的是得到所有在activity运行过程中的的交互事件，比如可以管理通知栏的通知。
接着会调用getWindow().superDispatchTouchEvent(ev)方法，而跟进源码会发现getWindow方法返回的对象是Window。
```java
/**
   * Retrieve the current {@link android.view.Window} for the activity.
   * This can be used to directly access parts of the Window API that
   * are not available through Activity/Screen.
   *
   * @return Window The current window, or null if the activity is not
   *         visual.
   */
  public Window getWindow() {
      return mWindow;
  }

   private Window mWindow;
```
而Window是个抽象类，其实现类为PhoneWindow，因此superDispatchTouchEvent方法的具体实现在phonewindow中，继续跟进源码如下：
```java
//from PhoneWindow.java
// This is the top-level view of the window, containing the window decor.
private DecorView mDecor;

@Override
  public boolean superDispatchTouchEvent(MotionEvent event) {
      return mDecor.superDispatchTouchEvent(event);
  }

  //from DecorView.java
  public class DecorView extends FrameLayout

  public boolean superDispatchTouchEvent(MotionEvent event) {
      return super.dispatchTouchEvent(event);
  }
  ```  

  可以看出方法是由phonewindow中decorview调用的，而decorview继承自FrameLayout，FrameLayout继承自ViewGroup，即super.dispatchTouchEvent方法的为ViewGroup的
dispatchTouchEvent方法。源码如下：
```java

```
