---
layout: post
title: "Android 触摸事件机制(四) ViewGroup中触摸事件详解"
description: "android"
category: android
tags: [android]
date: 2015-01-04 09:01
---


> 本文将对ViewGroup中触摸事件相关的内容进行介绍。ViewGroup继承于View，所以说，ViewGroup中对触摸事件的处理，很多都继承于View。但是，ViewGroup又有自己对触摸事件的特定处理。本文重点介绍的是dispatchTouchEvent()；理解ViewGroup的dispatchTouchEvent()接口是理解Android触摸事件传递机制的关机。

> 注意：本文是基于Android 4.4.2版本进行介绍的！

> **目录**  
> **1**. [ViewGroup中触摸事件的概述](#anchor1)  
> **2**. [ViewGroup中触摸事件的源码解析](#anchor2)  
> **2.1**. [ViewGroup中的dispatchTouchEvent](#anchor2_1)  
> **2.2**. [ViewGroup中的onTouchEvent](#anchor2_2)  



<a name="anchor1"></a>
# 1. ViewGroup中触摸事件的概述

ViewGroup继承于View，它中对触摸事件的处理，很多都继承于View。但是，ViewGroup又有自己对触摸事件的特定处理。  
(01) ViewGroup重载了dispatchTouchEvent()接口。  
(02) ViewGroup新增了onInterceptTouchEvent()接口。


<a name="anchor2"></a>
# 2. ViewGroup中触摸事件的源码解析

<a name="anchor2_1"></a>
## 2.1 ViewGroup中的dispatchTouchEvent

    @Override
    public boolean dispatchTouchEvent(MotionEvent ev) {
        // mInputEventConsistencyVerifier是调试用的，不会理会
        if (mInputEventConsistencyVerifier != null) {
            mInputEventConsistencyVerifier.onTouchEvent(ev, 1);
        }

        // 第1步：是否要分发该触摸事件
        //
        // onFilterTouchEventForSecurity()表示是否要分发该触摸事件。 
        // 如果该View不是位于顶部，并且有设置属性使该View不在顶部时不响应触摸事件，则不分发该触摸事件，即返回false。
        // 否则，则对触摸事件进行分发，即返回true。
        boolean handled = false;
        if (onFilterTouchEventForSecurity(ev)) {
            final int action = ev.getAction();
            final int actionMasked = action & MotionEvent.ACTION_MASK;

            // 第2步：检测是否需要清空目标和状态
            //
            // 如果是ACTION_DOWN(即按下事件)，则清空之前的触摸事件处理目标和状态。
            // 这里的情况状态包括：
            // (01) 清空mFirstTouchTarget链表，并设置mFirstTouchTarget为null。
            //      mFirstTouchTarget是"接受触摸事件的View"所组成的单链表
            // (02) 清空mGroupFlags的FLAG_DISALLOW_INTERCEPT标记
            //      如果设置了FLAG_DISALLOW_INTERCEPT，则不允许ViewGroup对触摸事件进行拦截。
            // (03) 清空mPrivateFlags的PFLAG_CANCEL_NEXT_UP_EVEN标记
            if (actionMasked == MotionEvent.ACTION_DOWN) {
                cancelAndClearTouchTargets(ev);
                resetTouchState();
            }    

            // 第3步：检查当前ViewGroup是否想要拦截触摸事件
            // 
            // 是的话，设置intercepted为true；否则intercepted为false。
            // 如果是"按下事件(ACTION_DOWN)" 或者 mFirstTouchTarget不为null；就执行if代码块里面的内容。
            // 否则的话，设置intercepted为true。
            final boolean intercepted;
            if (actionMasked == MotionEvent.ACTION_DOWN || mFirstTouchTarget != null) {
                // 检查禁止拦截标记：FLAG_DISALLOW_INTERCEPT
                // 如果调用了requestDisallowInterceptTouchEvent()标记的话，则FLAG_DISALLOW_INTERCEPT会为true。
                // 例如，ViewPager在处理触摸事件的时候，就会调用requestDisallowInterceptTouchEvent()
                //     ，禁止它的父类对触摸事件进行拦截
                final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
                if (!disallowIntercept) {
                    // 如果禁止拦截标记为false的话，则调用onInterceptTouchEvent()；并返回拦截状态。
                    intercepted = onInterceptTouchEvent(ev);
                    ev.setAction(action); // restore action in case it was changed
                } else {
                    intercepted = false;
                }    
            } else {
                intercepted = true;
            }    

            // 第4步：检查当前的触摸事件是否被取消
            // 
            // (01) 对于ACTION_DOWN而言，mPrivateFlags的PFLAG_CANCEL_NEXT_UP_EVENT位肯定是0；因此，canceled=false。
            // (02) 当前的View或ViewGroup要被从父View中detach时，PFLAG_CANCEL_NEXT_UP_EVENT就会被设为true；
            //      此时，它就不再接受触摸事情。
            final boolean canceled = resetCancelNextUpFlag(this)
                    || actionMasked == MotionEvent.ACTION_CANCEL;

            // 第5步：将触摸事件分发给"当前ViewGroup的子View和子ViewGroup"
            // 
            // 如果触摸"没有被取消"，同时也"没有被拦截"的话，则将触摸事件分发给它的子View和子ViewGroup。  
            //     如果当前ViewGroup的孩子有接受触摸事件的话，则将该孩子添加到mFirstTouchTarget链表中。
            final boolean split = (mGroupFlags & FLAG_SPLIT_MOTION_EVENTS) != 0;
            TouchTarget newTouchTarget = null;
            boolean alreadyDispatchedToNewTouchTarget = false;
            if (!canceled && !intercepted) {
                if (actionMasked == MotionEvent.ACTION_DOWN
                        || (split && actionMasked == MotionEvent.ACTION_POINTER_DOWN)
                        || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {
                    // 这是获取触摸事件的序号 以及 触摸事件的id信息。
                    // (01) 对于ACTION_DOWN，actionIndex肯定是0
                    // (02) 而getPointerId()是获取的该触摸事件的id，并将该id信息保存到idBitsToAssign中。
                    //    这个触摸事件的id是为多指触摸而添加的；对于单指触摸，getActionIndex()返回的肯定是0；
                    //    而对于多指触摸，第一个手指的id是0，第二个手指的id是1，第三个手指的id是2，...依次类推。
                    final int actionIndex = ev.getActionIndex();
                    final int idBitsToAssign = split ? 1 << ev.getPointerId(actionIndex)
                            : TouchTarget.ALL_POINTER_IDS;

                    // 清空这个手指之前的TouchTarget链表。
                    // 一个TouchTarget，相当于一个可以被触摸的对象；它中记录了接受触摸事件的View
                    removePointersFromTouchTargets(idBitsToAssign);

                    // 获取该ViewGroup包含的View和ViewGroup的数目，
                    // 然后递归遍历ViewGroup的孩子，对触摸事件进行分发。
                    // 递归遍历ViewGroup的孩子：是指对于当前ViewGroup的所有孩子，都会逐个遍历，并分发触摸事件；
                    //   对于逐个遍历到的每一个孩子，若该孩子是ViewGroup类型的话，则会递归到调用该孩子的孩子，...
                    final int childrenCount = mChildrenCount;
                    if (newTouchTarget == null && childrenCount != 0) {
                        final float x = ev.getX(actionIndex);
                        final float y = ev.getY(actionIndex);
                        final View[] children = mChildren;

                        final boolean customOrder = isChildrenDrawingOrderEnabled();
                        for (int i = childrenCount - 1; i >= 0; i--) {
                            final int childIndex = customOrder ?
                                    getChildDrawingOrder(childrenCount, i) : i;
                            final View child = children[childIndex];
                            // 如果child可以接受触摸事件，
                            // 并且触摸坐标(x,y)在child的可视范围之内的话；
                            // 则继续往下执行。否则，调用continue。
                            // child可接受触摸事件：是指child的是可见的(VISIBLE)；或者虽然不可见，但是位于动画状态。
                            if (!canViewReceivePointerEvents(child)
                                    || !isTransformedTouchPointInView(x, y, child, null)) {
                                continue;
                            }

                            // getTouchTarget()的作用是查找child是否存在于mFirstTouchTarget的单链表中。
                            // 是的话，返回对应的TouchTarget对象；否则，返回null。
                            newTouchTarget = getTouchTarget(child);
                            if (newTouchTarget != null) {
                                newTouchTarget.pointerIdBits |= idBitsToAssign;
                                break;
                            }

                            // 重置child的mPrivateFlags变量中的PFLAG_CANCEL_NEXT_UP_EVENT位。
                            resetCancelNextUpFlag(child);

                            // 调用dispatchTransformedTouchEvent()将触摸事件分发给child。
                            if (dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)) {
                                // 如果child能够接受该触摸事件，即child消费或者拦截了该触摸事件的话；
                                // 则调用addTouchTarget()将child添加到mFirstTouchTarget链表的表头，并返回表头对应的TouchTarget
                                // 同时还设置alreadyDispatchedToNewTouchTarget为true。
                                mLastTouchDownTime = ev.getDownTime();
                                mLastTouchDownIndex = childIndex;
                                mLastTouchDownX = ev.getX();
                                mLastTouchDownY = ev.getY();
                                newTouchTarget = addTouchTarget(child, idBitsToAssign);
                                alreadyDispatchedToNewTouchTarget = true;
                                break;
                            }
                        }
                    }

                    // 如果newTouchTarget为null，并且mFirstTouchTarget不为null；
                    // 则设置newTouchTarget为mFirstTouchTarget链表中第一个不为空的节点。
                    if (newTouchTarget == null && mFirstTouchTarget != null) {
                        // Did not find a child to receive the event.
                        // Assign the pointer to the least recently added target.
                        newTouchTarget = mFirstTouchTarget;
                        while (newTouchTarget.next != null) {
                            newTouchTarget = newTouchTarget.next;
                        }
                        newTouchTarget.pointerIdBits |= idBitsToAssign;
                    }
                }
            }

            // 第6步：进一步的对触摸事件进行分发
            // 
            // (01) 如果mFirstTouchTarget为null，意味着还没有任何View来接受该触摸事件；
            //   此时，将当前ViewGroup看作一个View；
            //   将会调用"当前的ViewGroup的父类View的dispatchTouchEvent()"对触摸事件进行分发处理。
            //   即，会将触摸事件交给当前ViewGroup的onTouch(), onTouchEvent()进行处理。
            // (02) 如果mFirstTouchTarget不为null，意味着有ViewGroup的子View或子ViewGroup中，
            //   有可以接受触摸事件的。那么，就将触摸事件分发给这些可以接受触摸事件的子View或子ViewGroup。
            if (mFirstTouchTarget == null) {
                // 注意：这里的第3个参数是null
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

            // 第7步：再次检查取消标记，并进行相应的处理
            // 
            // Update list of touch targets for pointer up or cancel, if needed.
            if (canceled
                    || actionMasked == MotionEvent.ACTION_UP
                    || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {
                resetTouchState();
            } else if (split && actionMasked == MotionEvent.ACTION_POINTER_UP) {
                final int actionIndex = ev.getActionIndex();
                final int idBitsToRemove = 1 << ev.getPointerId(actionIndex);
                removePointersFromTouchTargets(idBitsToRemove);
            }
        }

        // mInputEventConsistencyVerifier是调试用的，不会理会
        if (!handled && mInputEventConsistencyVerifier != null) {
            mInputEventConsistencyVerifier.onUnhandledEvent(ev, 1);
        }
        return handled;
    }

说明：该代码定义在frameworks/base/core/java/android/view/ViewGroup.java中。流程比较复杂，但文章已经给出了非常详细的注释，相信根据注释应该能读懂。遇到不懂或有疑惑的地方，还需阅读源码才是！  

注意：**第5步，即ViewGroup尝试将触摸事件分发给它的孩子。这只有在ACTION_DOWN的时候才发生。如果它的孩子接受了触摸事件，则会调用addTouchTarget()将该孩子添加到mFirstTouchTarget链表中。  在ACTION_DOWN之后，传递ACTION_MOVE或ACTION_UP时，ViewGroup不会再执行第5步；而是在第6步中，直接遍历mFirstTouchTarget链表，查找之前接受ACTION_DOWN的孩子，并将触摸事件分配给这些孩子。**  
**也就是说，如果ViewGroup的某个孩子没有接受ACTION_DOWN事件；那么，ACTION_MOVE和ACTION_UP等事件也一定不会分发给这个孩子！**





### 2.1.1 ViewGroup中的dispatchTransformedTouchEvent()

    private boolean dispatchTransformedTouchEvent(MotionEvent event, boolean cancel,
            View child, int desiredPointerIdBits) {
        final boolean handled;

        // 检测是否需要发送ACTION_CANCEL。
        // 如果cancel为true 或者 action是ACTION_CANCEL;
        // 则设置消息为ACTION_CANCEL，并将ACTION_CANCEL消息分发给对应的对象，并返回。
        // (01) 如果child是空，则将ACTION_CANCEL消息分发给当前ViewGroup；
        //      只不过会将ViewGroup看作它的父类View，调用View的dispatchTouchEvent()接口。
        // (02) 如果child不是空，调用child的dispatchTouchEvent()。
        final int oldAction = event.getAction();
        if (cancel || oldAction == MotionEvent.ACTION_CANCEL) {
            event.setAction(MotionEvent.ACTION_CANCEL);
            if (child == null) {
                handled = super.dispatchTouchEvent(event);
            } else {
                handled = child.dispatchTouchEvent(event);
            }
            event.setAction(oldAction);
            return handled;
        }

        // 计算触摸事件的id信息
        final int oldPointerIdBits = event.getPointerIdBits();
        final int newPointerIdBits = oldPointerIdBits & desiredPointerIdBits;

        // 如果新的id信息为0，则返回false。
        if (newPointerIdBits == 0) {
            return false;
        }


        // 如果计算得到的前后触摸事件id信息相同，则执行不需要重新计算MotionEvent，直接执行if语句块进行消费分发；
        // 否则，就重新计算MotionEvent之后，再进行消息分发。
        final MotionEvent transformedEvent;
        if (newPointerIdBits == oldPointerIdBits) {
            if (child == null || child.hasIdentityMatrix()) {
                // (01) 如果child是空，则将ViewGroup看作它的父类View，调用View的dispatchTouchEvent()接口。
                // (02) 如果child不是空，调用child的dispatchTouchEvent()。
                if (child == null) {
                    handled = super.dispatchTouchEvent(event);
                } else {
                    final float offsetX = mScrollX - child.mLeft;
                    final float offsetY = mScrollY - child.mTop;
                    event.offsetLocation(offsetX, offsetY);

                    handled = child.dispatchTouchEvent(event);

                    event.offsetLocation(-offsetX, -offsetY);
                }
                return handled;
            }
            transformedEvent = MotionEvent.obtain(event);
        } else {
            transformedEvent = event.split(newPointerIdBits);
        }

        // (01) 如果child是空，则将ViewGroup看作它的父类View，调用View的dispatchTouchEvent()接口。
        // (02) 如果child不是空，调用child的dispatchTouchEvent()。
        if (child == null) {
            handled = super.dispatchTouchEvent(transformedEvent);
        } else {
            final float offsetX = mScrollX - child.mLeft;
            final float offsetY = mScrollY - child.mTop;
            transformedEvent.offsetLocation(offsetX, offsetY);
            if (! child.hasIdentityMatrix()) {
                transformedEvent.transform(child.getInverseMatrix());
            }

            handled = child.dispatchTouchEvent(transformedEvent);
        }

        // Done.
        transformedEvent.recycle();
        return handled;
    }

说明：dispatchTransformedTouchEvent()会对触摸事件进行重新打包后再分发。  
如果它的第三个参数child是null，则会将触摸消息分发给ViewGroup自己，只不过此时是将ViewGroup看作一个View，即调用View的dispatchTouchEvent()进行消息分发。而View的dispatchTouchEvent()在前面一篇文章中已经消息介绍过了，它会触摸事件分发给onTouch(), onTouchEvent()进行处理。
如果它的第三个参数child不是null，则会调用child.dispatchTouchEvent()进行消息分发。而如果这个child是ViewGroup对象的话，它则又会递归的将消息分发给它的孩子。


### 2.1.2 ViewGroup中的cancelAndClearTouchTargets()

    private TouchTarget mFirstTouchTarget;

    private void cancelAndClearTouchTargets(MotionEvent event) {
        // 如果mFirstTouchTarget链表不为空，则清空该链表
        if (mFirstTouchTarget != null) {
            ...

            // 遍历mFirstTouchTarget链表，请清空每一个TouchTarget中View的PFLAG_CANCEL_NEXT_UP_EVENT位
            for (TouchTarget target = mFirstTouchTarget; target != null; target = target.next) {
                resetCancelNextUpFlag(target.child);
                ...
            }
            // 清空TouchTarget链表本身，并设置mFirstTouchTarget为null
            clearTouchTargets();

            ...
        }
    }

    private static boolean resetCancelNextUpFlag(View view) {
        // 清空mPrivateFlags中的PFLAG_CANCEL_NEXT_UP_EVENT位
        if ((view.mPrivateFlags & PFLAG_CANCEL_NEXT_UP_EVENT) != 0) {
            view.mPrivateFlags &= ~PFLAG_CANCEL_NEXT_UP_EVENT;
            return true;
        }
        return false;
    }

    private void clearTouchTargets() {
        TouchTarget target = mFirstTouchTarget;
        // 清空mFirstTouchTarget链表，并设置mFirstTouchTarget为null
        if (target != null) {
            do {
                TouchTarget next = target.next;
                target.recycle();
                target = next;
            } while (target != null);
            mFirstTouchTarget = null;
        }
    }

说明：cancelAndClearTouchTargets()的作用和明显。就是清空mFirstTouchTarget链表中每一个View的PFLAG_CANCEL_NEXT_UP_EVENT标记；然后清空mFirstTouchTarget链表，并设置mFirstTouchTarget为null。  
mFirstTouchTarget是TouchTarget类的成员。TouchTarget是ViewGroup的内部类，一个TouchTarget对象可以视为一个被触摸对象；即，在ViewGroup中，就通过TouchTarget表示一个可以接受触摸事件的对象。


    private static final class TouchTarget {
        ...

        // 被触摸的View
        public View child;

        // pointerIdBits是记录触摸事件的id信息(对于多指触摸而言)
        public int pointerIdBits;

        // TouchTarget指向的下一个节点
        public TouchTarget next;


        private TouchTarget() {
        }

        ...
    }


### 2.1.3 ViewGroup中的resetTouchState()

    private void resetTouchState() {
        clearTouchTargets();
        resetCancelNextUpFlag(this);
        mGroupFlags &= ~FLAG_DISALLOW_INTERCEPT;
    }

说明：resetTouchState()是清空当前ViewGroup的点击状态。



### 2.1.4 ViewGroup中的removePointersFromTouchTargets()

    private void removePointersFromTouchTargets(int pointerIdBits) {
        TouchTarget predecessor = null;
        TouchTarget target = mFirstTouchTarget;
        while (target != null) {
            final TouchTarget next = target.next;
            if ((target.pointerIdBits & pointerIdBits) != 0) {
                target.pointerIdBits &= ~pointerIdBits;
                if (target.pointerIdBits == 0) {
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

说明：理解removePointersFromTouchTargets()的关机，是理解ev.getPointerId()。而getPointerId()是获取的该触摸事件的id。对于多指触摸，第一个手指的id是0，第二个手指的id是1，第三个手指的id是2，...依次类推。  
而pointerIdBits就是记录的id信息的。第一个手指的pointerIdBits是0x0，第二个手指的的pointerIdBits是0x1，第三个手指的pointerIdBits是0x2，第四个手指的pointerIdBits是0x4，...所有手指的pointerIdBits是0xffffffff。  
理解了触摸id之后，再看看removePointersFromTouchTargets()就非常容易理解了。它是从mFirstTouchTarget链表中逐个遍历，清空pointerIdBits；如果清空pointerIdBits之后，TouchTarget的pointerIdBits为0，则将该节点从链表中删除。



### 2.1.5 ViewGroup中的canViewReceivePointerEvents()

    private static boolean canViewReceivePointerEvents(View child) {
        return (child.mViewFlags & VISIBILITY_MASK) == VISIBLE
                || child.getAnimation() != null;
    }

说明：canViewReceivePointerEvents()是判断child是否可以接受触摸事件。如果child是VISIBLE；或者child是非VISIBLE，但是它处于动画状态；这两种状态都可以接受触摸事件。


### 2.1.6 ViewGroup中的isTransformedTouchPointInView()

    protected boolean isTransformedTouchPointInView(float x, float y, View child,
            PointF outLocalPoint) {
        float localX = x + mScrollX - child.mLeft;
        float localY = y + mScrollY - child.mTop;
        if (! child.hasIdentityMatrix() && mAttachInfo != null) {
            final float[] localXY = mAttachInfo.mTmpTransformLocation;
            localXY[0] = localX;
            localXY[1] = localY;
            child.getInverseMatrix().mapPoints(localXY);
            localX = localXY[0];
            localY = localXY[1];
        }
        final boolean isInView = child.pointInView(localX, localY);
        ...
        return isInView;
    }

说明：isTransformedTouchPointInView()是判断(x,y)是否在child中。



### 2.1.7 ViewGroup中的getTouchTarget()

    private TouchTarget getTouchTarget(View child) {
        for (TouchTarget target = mFirstTouchTarget; target != null; target = target.next) {
            if (target.child == child) {
                return target;
            }
        }
        return null;
    }

说明：getTouchTarget()的作用是查找child是否存在于mFirstTouchTarget的单链表中。是的话，返回对应的TouchTarget对象；否则，返回null。



<a name="anchor2_2"></a>
## 2.2 ViewGroup中的onTouchEvent

ViewGroup没有覆盖onTouchEvent()。因此，调用ViewGroup的onTouchEvent()的话；实际上调用的是它的父类View的onTouchEvent()。



<a name="anchor2_3"></a>
## 2.3 ViewGroup中的onInterceptTouchEvent

    public boolean onInterceptTouchEvent(MotionEvent ev) {
        return false;
    }   

说明：ViewGroup的onInterceptTouchEvent()默认返回false。而且什么都不会执行！  
一般来说，若我们自定义ViewGroup时，需要拦截触摸消息；就可以覆盖onInterceptTouchEvent()来进行。



ViewGroup中关于触摸事件的代码就分析至此。总的来说：  
(01) **ViewGroup中的dispatchTouchEvent()会将触摸事件进行递归遍历传递。ViewGroup会遍历它的所有孩子，对每个孩子都递归的调用dispatchTouchEvent()来分发触摸事件。**
(02) **如果ViewGroup的某个孩子没有接受(消费或者拦截)ACTION_DOWN事件；那么，ACTION_MOVE和ACTION_UP等事件也一定不会分发给这个孩子！**
(03) **ViewGroup的onInterceptTouchEvent()默认返回false。**
