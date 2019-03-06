---
layout:     post
title:      View触摸事件分发机制源码解析
subtitle:    ""
date:       2018-1-5 14:20
author:     guoqing
header-img: img/posts/post-bg-android.jpg
catalog: true
tags:
    - Android
    - 事件分发
    - 自定义view
---
### View触摸事件分发机制源码解析
Android中Activity是负责所有与用户交互的事务，所有可视界面的创建都是从Activity开始。因此view的触摸事件的处理也是从Activity开始的，其开始处理触摸事件的方法如下所示：
```java
/**
//from Activity.java
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
                          //首先先从TouchTarget链表中查找是否child已经在链表中，如果存在则表明child已经响应触摸事件，则直接跳出循环，如果不在则开始调用dispatchTransformedTouchEvent方法。
                          newTouchTarget = getTouchTarget(child);
                            if (newTouchTarget != null) {

                                newTouchTarget.pointerIdBits |= idBitsToAssign;
                                break;
                            }
                          //dispatchTransformedTouchEvent方法主要用来调用子view的dispatchTouchEvent方法，如果子view拦截了此事件，则遍历之前view的集合找到触摸view，然后通过addTouchTarget将newTouchTarget赋值然后break跳出循环，alreadyDispatchedToNewTouchTarget,mFirstTouchTarget就是在这个方法里被赋值,
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
                              //找到处理触摸事件的子view则将child添加到链表中并赋值newTouchTarget
                               newTouchTarget = addTouchTarget(child);
                              //alreadyDispatchedToNewTouchTarget标志位为ture
                               alreadyDispatchedToNewTouchTarget = true;
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
              //如果mFirstTouchTarget不为null，则会继续遍历子view并调用子view的dispatchTransformedTouchEvent方法
                TouchTarget predecessor = null;
                TouchTarget target = mFirstTouchTarget;
                while (target != null) {
                    final TouchTarget next = target.next;
                    //链表中的子view已经处理过事件，handle直接为true，否则继续遍历链表的子view并调用dispatchTransformedTouchEvent遍历并调用其子view的didispatchTouchEvent方法
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
以上就是viewgroup的事件分发全过程，首先在action_down事件到来时重置所有的触摸状态，然后如果子类没有拦截viewgroup的事件，则先调用viewgroup的onInterceptTouchEvent方法来查看自己是否拦截触摸事件，如果拦截则调用viewgroup的onTouchEvent方法处理事件。如果不拦截，首先将触摸区域的view存放在列表中，并依次调用子view的dispatchTouchEvent方法，并将响应触摸事件的view存放在TouchTarget链表中，然后会继续遍历链表中的view，将事件传第到子view中。这样就完成了一轮事件分发，下面分析子view对事件分发的处理。
```java
public boolean dispatchTouchEvent(MotionEvent event) {
//noinspection SimplifiableIfStatement
           ListenerInfo li = mListenerInfo;
           if (li != null && li.mOnTouchListener != null
                   && (mViewFlags & ENABLED_MASK) == ENABLED
                   && li.mOnTouchListener.onTouch(this, event)) {
               result = true;
           }

           if (!result && onTouchEvent(event)) {
               result = true;
           }
       }
}
 return result;
```
ListenerInfo是个静态内部类里面封装了点击滑动事件监听接口，如常见的OnClickListener、OnTouchListener等。如果view设置了onTouchListener则会调用OnTouch方法，否则OntouchEvent方法会被调用。由此可以看出View的OnTouch方法的优先级比较高，如果返回true了则OnTouchEvent方法不会被调用。下面来看一下onTouchEvent方法。
```java
    public boolean onTouchEvent(MotionEvent event) {

       final int action = event.getAction();
       //clickable判断view是否可点击
       final boolean clickable = ((viewFlags & CLICKABLE) == CLICKABLE
               || (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE)
               || (viewFlags & CONTEXT_CLICKABLE) == CONTEXT_CLICKABLE;
               //如果view是disable状态，任然返回view本身的clickable状态。所以即使view本身是不可点击的，它仍然消耗事件
          if ((viewFlags & ENABLED_MASK) == DISABLED) {
            if (action == MotionEvent.ACTION_UP && (mPrivateFlags & PFLAG_PRESSED) != 0) {
                setPressed(false);
            }
            mPrivateFlags3 &= ~PFLAG3_FINGER_DOWN;
            // A disabled view that is clickable still consumes the touch
            // events, it just doesn't respond to them.
            return clickable;
        }
        //如果view设置了TouchDelegate，则调用Touchdelegate的onTouchEvent方法。TouchDelegate是用来增大view本身点击范围的一个帮助类。
        if (mTouchDelegate != null) {
            if (mTouchDelegate.onTouchEvent(event)) {
                return true;
            }
        }
        //view的点击事件发生在up事件中，首先view必须是可点击的并且不是长按事件
        if (clickable || (viewFlags & TOOLTIP) == TOOLTIP) {
            switch (action) {
                case MotionEvent.ACTION_UP:

                    boolean prepressed = (mPrivateFlags & PFLAG_PREPRESSED) != 0;
                        ....

                        //mHasPerformedLongPress的赋值是在down事件中的checkForLongClick中。
                        if (!mHasPerformedLongPress && !mIgnoreNextUpEvent) {
                            // This is a tap, so remove the longpress check
                            removeLongPressCallback();

                            // 如果是view的转状态是点击状态
                            if (!focusTaken) {
                              //开始执行点击事件
                                // Use a Runnable and post this rather than calling
                                // performClick directly. This lets other visual state
                                // of the view update before click actions start.
                                if (mPerformClick == null) {
                                    mPerformClick = new PerformClick();
                                }
                                if (!post(mPerformClick)) {
                                    performClick();
                                }
                            }
                        }
                        ...
        case MotionEvent.ACTION_DOWN:
        if (!clickable) {
                       checkForLongClick(0, x, y);
                       break;
                   }
                   ...

```
view在点击和长按状态分别会触发performClick和performLongClick事件，然后在两个方法中分别会调用ListenerInfo的onClick和onLongClick方法。以上就是view触摸事件分发机制源码的大概流程。
