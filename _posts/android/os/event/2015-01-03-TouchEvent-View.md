---
layout: post
title: "Android 触摸事件机制(三) View中触摸事件详解"
description: "android"
category: android
tags: [android]
date: 2015-01-03 09:01
---


> 本文将对View中触摸事件相关的内容进行介绍。重点介绍的是dispatchTouchEvent(), onTouchEvent()这两个API以及OnTouchListener接口。

> 注意：本文是基于Android 4.4.2版本进行介绍的！

> **目录**  
> **1**. [View中触摸事件的概述](#anchor1)  
> **2**. [View中触摸事件的源码解析](#anchor2)  
> **2.1**. [View中的dispatchTouchEvent](#anchor2_1)  
> **2.2**. [View中的onTouchEvent](#anchor2_2)  



<a name="anchor1"></a>
# 1. View中触摸事件的概述

  View中与触摸事件相关的内容可以分为两部分。

**第一部分** dispatchTouchEvent()和onTouchEvent()这两个API

dispatchTouchEvent()是传递触摸事件的API，而onTouchEvent()则是View处理触摸事件的API。  
View中dispatchTouchEvent()将事件传递给"自己的onTouch()", "自己的onTouchEvent()"进行处理。 onTouch()是OnTouchListener接口中API，属于View提供的，让用户自己处理触摸事件的接口。而onTouchEvent()是Android系统提供的，用于处理触摸事件的接口；在onTouchEvent()中会进行一系列的动作，例如获取焦点、设置按下状态，调用onClick()等。

**第二部分** OnTouchListener, OnClickListener, OnLongClickListener等接口

这部分主要是接口。但本文主要介绍的是OnTouchListener接口中的onTouch()。为什么呢？  
这是因为，onTouch()与onTouchEvent()都是用户处理触摸事件的API。  
但不同的是：(01), onTouch()是View提供给用户，让用户自己处理触摸事件的接口。而onTouchEvent()是Android系统自己实现的接口。(02)，onTouch()的优先级比onTouchEvent()的优先级更高。dispatchTouchEvent()中分发事件的时候，会先将事件分配给onTouch()进行处理，然后才分配给onTouchEvent()进行处理。  如果onTouch()对触摸事件进行了处理，并且返回true；那么，该触摸事件就不会分配在分配给onTouchEvent()进行处理了。只有当onTouch()没有处理，或者处理了但返回false时，才会分配给onTouchEvent()进行处理。

    public interface OnTouchListener {
        boolean onTouch(View v, MotionEvent event);
    }


<a name="anchor2"></a>
# 2. View中触摸事件的源码解析

<a name="anchor2_1"></a>
## 2.1 View中的dispatchTouchEvent

    public boolean dispatchTouchEvent(MotionEvent event) {
        if (mInputEventConsistencyVerifier != null) {
            mInputEventConsistencyVerifier.onTouchEvent(event, 0);
        }

        // 如果该View被遮蔽，并且该View在被遮蔽时不响应点击事件；
        // 此时，返回false；不会执行onTouch()或onTouchEvent()，即过滤调用该点击事件。
        // 否则，返回true。
        // 被遮蔽的意思是：该View不是位于顶部，有其他的View在它之上。
        if (onFilterTouchEventForSecurity(event)) {
            //noinspection SimplifiableIfStatement
            ListenerInfo li = mListenerInfo;
            if (li != null && li.mOnTouchListener != null && (mViewFlags & ENABLED_MASK) == ENABLED
                    && li.mOnTouchListener.onTouch(this, event)) {
                return true;
            }

            if (onTouchEvent(event)) {
                return true;
            }
        }

        if (mInputEventConsistencyVerifier != null) {
            mInputEventConsistencyVerifier.onUnhandledEvent(event, 0);
        }
        return false;
    }

说明：该代码定义在frameworks/base/core/java/android/view/View.java中。  
(01) mInputEventConsistencyVerifier是调试用的，这里不用理会。重点看onFilterTouchEventForSecurity()部分。  
(02) onFilterTouchEventForSecurity()表示是否要分发该触摸事件；如果该View不是位于顶部，并且有设置属性使该View不在顶部时不响应触摸事件，则不分发该触摸事件，即不会执行onTouch()与onTouchEvent()。 否则的话，则将事件分发给onTouch(), onTouchEvent()进行处理。
(03) 如果将事件进行分发的话，会先尝试分发给onTouch()；然后才分发给onTouchEvent()。

### 2.1.1 View中的onFilterTouchEventForSecurity()

    public boolean onFilterTouchEventForSecurity(MotionEvent event) {
        //noinspection RedundantIfStatement
        if ((mViewFlags & FILTER_TOUCHES_WHEN_OBSCURED) != 0
                && (event.getFlags() & MotionEvent.FLAG_WINDOW_IS_OBSCURED) != 0) {
            // Window is obscured, drop this touch.
            return false;
        }
        return true;
    }

说明：onFilterTouchEventForSecurity()返回true，表示可以分发该触摸事件；否则，不能分发该触摸事件。不能分发事件的情况，只有mViewFlags&FILTER_TOUCHES_WHEN_OBSCURED!=0，并且event.getFlags()&MotionEvent.FLAG_WINDOW_IS_OBSCURED!=0同时成立。  
(01) FILTER_TOUCHES_WHEN_OBSCURED是android:filterTouchesWhenObscured属性所对应的位。android:filterTouchesWhenObscured是true的话，则表示其他视图在该视图之上，导致该视图被隐藏时，该视图就不再响应触摸事件。  
(02) MotionEvent.FLAG_WINDOW_IS_OBSCURED为true的话，则表示该视图的窗口是被隐藏的。  




<a name="anchor2_2"></a>
## 2.2 View中的onTouchEvent

    public boolean onTouchEvent(MotionEvent event) {
        final int viewFlags = mViewFlags;

        // 如果View被禁用的话，则返回它是否可以点击。
        if ((viewFlags & ENABLED_MASK) == DISABLED) {
            if (event.getAction() == MotionEvent.ACTION_UP && (mPrivateFlags & PFLAG_PRESSED) != 0) {
                setPressed(false);
            }
            return (((viewFlags & CLICKABLE) == CLICKABLE ||
                    (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE));
        }

        // 如果该View的mTouchDelegate不为null的话，将触摸消息分发给mTouchDelegate。
        // mTouchDelegate的默认值是null。
        if (mTouchDelegate != null) {
            if (mTouchDelegate.onTouchEvent(event)) {
                return true;
            }
        }

        // 如果View可以被点击的话，则执行if里面的内容。
        // 这其中涉及到的主要是获取焦点，设置按下状态，触发onClick(), onLongClick()事件等等。
        if (((viewFlags & CLICKABLE) == CLICKABLE ||
                (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE)) {
            switch (event.getAction()) {
                case MotionEvent.ACTION_UP:
                    boolean prepressed = (mPrivateFlags & PFLAG_PREPRESSED) != 0;
                    if ((mPrivateFlags & PFLAG_PRESSED) != 0 || prepressed) {
                        // take focus if we don't have it already and we should in
                        // touch mode.
                        boolean focusTaken = false;
                        if (isFocusable() && isFocusableInTouchMode() && !isFocused()) {
                            focusTaken = requestFocus();
                        }

                        if (prepressed) {
                            // The button is being released before we actually
                            // showed it as pressed.  Make it show the pressed
                            // state now (before scheduling the click) to ensure
                            // the user sees it.
                            setPressed(true);
                       }


                        if (!mHasPerformedLongPress) {
                            // This is a tap, so remove the longpress check
                            removeLongPressCallback();

                            // Only perform take click actions if we were in the pressed state
                            if (!focusTaken) {
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

                        if (mUnsetPressedState == null) {
                            mUnsetPressedState = new UnsetPressedState();
                        }

                        if (prepressed) {
                            postDelayed(mUnsetPressedState,
                                    ViewConfiguration.getPressedStateDuration());
                        } else if (!post(mUnsetPressedState)) {
                            // If the post failed, unpress right now
                            mUnsetPressedState.run();
                        }
                        removeTapCallback();
                    }
                    break;

                case MotionEvent.ACTION_DOWN:
                    mHasPerformedLongPress = false;

                    if (performButtonActionOnTouchDown(event)) {
                        break;
                    }

                    // Walk up the hierarchy to determine if we're inside a scrolling container.
                    boolean isInScrollingContainer = isInScrollingContainer();
                    // For views inside a scrolling container, delay the pressed feedback for
                    // a short period in case this is a scroll.
                    if (isInScrollingContainer) {
                        mPrivateFlags |= PFLAG_PREPRESSED;
                        if (mPendingCheckForTap == null) {
                            mPendingCheckForTap = new CheckForTap();
                        }
                        postDelayed(mPendingCheckForTap, ViewConfiguration.getTapTimeout());
                    } else {
                        // Not inside a scrolling container, so show the feedback right away
                        setPressed(true);
                        checkForLongClick(0);
                    }
                    break;

                case MotionEvent.ACTION_CANCEL:
                    setPressed(false);
                    removeTapCallback();
                    removeLongPressCallback();
                    break;

                case MotionEvent.ACTION_MOVE:
                    final int x = (int) event.getX();
                    final int y = (int) event.getY();

                    // Be lenient about moving outside of buttons
                    if (!pointInView(x, y, mTouchSlop)) {
                        // Outside button
                        removeTapCallback();
                        if ((mPrivateFlags & PFLAG_PRESSED) != 0) {
                            // Remove any future long press/tap checks
                            removeLongPressCallback();

                            setPressed(false);
                        }
                    }
                    break;
            }
            return true;
        }

        return false;
    }

说明：onTouchEvent()是Android系统实现的View对触摸事件的处理。  
(01) 如果View被禁用的话，则返回它是否可以点击。当我们调用了setEnabled(false)时，View就被禁用了；默认情况下，View是可用的。当调用setClickable(true)或者android:clickable为true时，View就是可点击状态；默认情况下，View是不可点击的。  
(02) 如果该View的mTouchDelegate不为null的话，将触摸消息分发给mTouchDelegate。例如，假设有两个视图v1和v2，它们的布局相互之间不重叠；如果设置了v1.setTouchDelegate(v2)的话，v1的触摸事件就会分发给v2。 注意：mTouchDelegate的默认值是null。  
(03) 如果View可以被点击的话，则执行if里面的内容。if里面涉及的内容很多，这里与本文的主题关联不大，暂且不表；如果要细将的话，估计得好几篇文章。例如，setPressed()是设置View的按下状态，如果用户有设置View在不同状态的图片时，setPressed()时会导致View的图片的更新。



View中关于触摸事件的代码就分析至此。总的来说：  
(01) **View中的dispatchTouchEvent()会将事件传递给"自己的onTouch()", "自己的onTouchEvent()"进行处理。而且onTouch()的优先级比onTouchEvent()的优先级要高。**
(02) **onTouch()与onTouchEvent()都是View中用户处理触摸事件的API。onTouch是OnTouchListener接口中的函数，OnTouchListener接口需要用户自己实现。onTouchEvent()是View自带的接口，Android系统提供了默认的实现；当然，用户可以重载该API。**  
(03) **onTouch()与onTouchEvent()有两个不同之处：(01), onTouch()是View提供给用户，让用户自己处理触摸事件的接口。而onTouchEvent()是Android系统自己实现的接口。(02)，onTouch()的优先级比onTouchEvent()的优先级更高。dispatchTouchEvent()中分发事件的时候，会先将事件分配给onTouch()进行处理，然后才分配给onTouchEvent()进行处理。  如果onTouch()对触摸事件进行了处理，并且返回true；那么，该触摸事件就不会分配在分配给onTouchEvent()进行处理了。只有当onTouch()没有处理，或者处理了但返回false时，才会分配给onTouchEvent()进行处理。**
