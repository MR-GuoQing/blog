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
@Override
   public boolean dispatchTouchEvent(MotionEvent ev) {
    ...

       boolean handled = false;
       if (onFilterTouchEventForSecurity(ev)) {
           final int action = ev.getAction();
           final int actionMasked = action & MotionEvent.ACTION_MASK;

           // 如果事件为down事件
           if (actionMasked == MotionEvent.ACTION_DOWN) {
          //首先清除所有之前事件，重置触摸状态
               cancelAndClearTouchTargets(ev);
               resetTouchState();
           }

           //事件是否被拦截
           final boolean intercepted;
           if (actionMasked == MotionEvent.ACTION_DOWN
                   || mFirstTouchTarget != null) {
                     //事件是否允许被拦截
               final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
               //如果不允许被子view拦截(子view可以通过requestDisallowInterceptTouchEvent方法修改disallowIntercept标志位)则调用onInterceptTouchEvent查看viewgroup是否拦截事件，并保存action事件。event.getAction()值就是在此处设置
               if (!disallowIntercept) {
                   intercepted = onInterceptTouchEvent(ev);
                   ev.setAction(action);  changed
               } else {
                   intercepted = false;
               }
           } else {
               // 如果mFirstTouchTarget位null并且action不是down事件，viewgroup继续拦截此事件，说明view决定拦截触摸事件，则后续move up事件将继续由viewgroup来处理
               intercepted = true;
           }
                           ...
          //触摸目标制空
           TouchTarget newTouchTarget = null;

           //如果viewgroup不拦截触摸事件并且事件没有被取消
           if (!canceled && !intercepted) {
             //所有触摸区域view的数量
             final int childrenCount = mChildrenCount;
             //newTouchTarget在前面已经制空并且触摸的区域的view数量不为0
                    if (newTouchTarget == null && childrenCount != 0) {
                        final float x = ev.getX(actionIndex);
                        final float y = ev.getY(actionIndex);
                        //将触摸区域的view从前到后排序存放在列表中（view的z坐标逆序排序，即最前面的view在列表的末端）
                        final ArrayList<View> preorderedList = buildTouchDispatchChildList();

                        final View[] children = mChildren;
                        final View child = getAndVerifyPreorderedView(
                                   preorderedList, children, childIndex);
                        //逆序遍历
                        for (int i = childrenCount - 1; i >= 0; i--) {
                          ...
                          //dispatchTransformedTouchEvent方法主要用来调用子view的dispatchTouchEvent方法，如果子view拦截了此事件，则遍历之前view的集合找到触摸view，然后通过addTouchTarget将newTouchTarget赋值，mFirstTouchTarget就是在这个方法里被赋值
                           if(dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)) {

                               mLastTouchDownTime = ev.getDownTime();
                               if (preorderedList != null) {

                                   for (int j = 0; j < childrenCount; j++) {
                                       if (children[childIndex] == mChildren[j]) {
                                           mLastTouchDownIndex = j;
                                           break;
                                       }
                                   }
                               } else {
                                   mLastTouchDownIndex = childIndex;
                               }

                               newTouchTarget = addTouchTarget(child,
                               break;
                           }
                       }
                   }

            }

            // 如果没有找到合适处理事件的view将再次调用dispatchTransformedTouchEvent方法，不过这次view是null，所以会调用super.dispatchTouchEvent(event)既view的dispatchTouchEvent方法会调用
            if (mFirstTouchTarget == null) {
                // No touch targets so treat this as an ordinary view.
                handled = dispatchTransformedTouchEvent(ev, canceled, null,
                        TouchTarget.ALL_POINTER_IDS);
            } else {
                // Dispatch to touch targets, excluding the new touch target if we already
                // dispatched to it.  Cancel touch targets if necessary.
                TouchTarget predecessor = null;
                TouchTarget target = mFirstTouchTarget;
                while (target != null) {
                    final TouchTarget next = target.next;
                    if (alreadyDispatchedToNewTouchTarget && target == newTouchTarget) {
                        handled = true;
                    } else {
                        final boolean cancelChild = resetCancelNextUpFlag(target.child)
                                || intercepted;
                        if (dispatchTransformedTouchEvent(ev, cancelChild,
                                target.child, target.pointerIdBits)) {
                            handled = true;
                        }
                        if (cancelChild) {
                            if (predecessor == null) {
                                mFirstTouchTarget = next;
                            } else {
                                predecessor.next = next;
                            }
                            target.recycle();
                            target = next;
                            continue;
                        }
                    }
                    predecessor = target;
                    target = next;
                }
            }
return handled;
}

```
