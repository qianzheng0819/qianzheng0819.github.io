---
layout: post
title:  "NestedScrollView源码分析"
date:   2023-09-14 15:28:00 +0800
tags:   android View
description:
---

布局结构
---------------

![p](/assets/images/2023-pic/p5.png)  


演示效果
------------
![p](/assets/images/2023-pic/p4.gif)  


源码解析
----------------

{% highlight java %}
public boolean onInterceptTouchEvent(MotionEvent ev) {
        

    switch (action & MotionEvent.ACTION_MASK) {
        
        case MotionEvent.ACTION_DOWN: {
            final int y = (int) ev.getY();
            if (!inChild((int) ev.getX(), y)) {
                mIsBeingDragged = false;
                recycleVelocityTracker();
                break;
            }

            /*
                * Remember location of down touch.
                * ACTION_DOWN always refers to pointer index 0.
                */
            mLastMotionY = y;
            mActivePointerId = ev.getPointerId(0);

            initOrResetVelocityTracker();
            mVelocityTracker.addMovement(ev);
            /*
                * If being flinged and user touches the screen, initiate drag;
                * otherwise don't. mScroller.isFinished should be false when
                * being flinged. We need to call computeScrollOffset() first so that
                * isFinished() is correct.
            */
            mScroller.computeScrollOffset();
            mIsBeingDragged = !mScroller.isFinished();
            startNestedScroll(ViewCompat.SCROLL_AXIS_VERTICAL, ViewCompat.TYPE_TOUCH);
            break;
        }

           
        return mIsBeingDragged;
    }
{% endhighlight %}

先看看viewGroup拦截事件,最后`return mIsBeingDragged`。mScroller.isFinished()为false时代表fling模式，可以分析得出正常情况下mIsBeingDragged为false，即nsv(NestedScrollView，以下都简称nsv)并不会拦截down事件。   

down事件处理调用了一个很重要的函数`startNestedScroll(ViewCompat.SCROLL_AXIS_VERTICAL, ViewCompat.TYPE_TOUCH)`


{% highlight java %}
@Override
public boolean startNestedScroll(int axes, int type) {
    return mChildHelper.startNestedScroll(axes, type);
}

public boolean startNestedScroll(@ScrollAxis int axes, @NestedScrollType int type) {
        if (hasNestedScrollingParent(type)) {
            // Already in progress
            return true;
        }
        if (isNestedScrollingEnabled()) {
            ViewParent p = mView.getParent();
            View child = mView;
            while (p != null) {
                if (ViewParentCompat.onStartNestedScroll(p, child, mView, axes, type)) {
                    setNestedScrollingParentForType(type, p);
                    ViewParentCompat.onNestedScrollAccepted(p, child, mView, axes, type);
                    return true;
                }
                if (p instanceof View) {
                    child = (View) p;
                }
                p = p.getParent();
            }
        }
        return false;
    }
{% endhighlight %}

mChildHelper是nsv作为嵌套滑动的子控件实现接口NestedScrollingChild的代理实现类   

`startNestedScroll`实际的作用有两个：
- 给子nsv找到它的父nsv，并保存到childHelper里
- 设置父nsv为嵌套滑动模式，即它的垂直或水平轴支持嵌套滑动

接着看nsv对move事件的拦截情况
{% highlight java %}
case MotionEvent.ACTION_MOVE: {
    final int y = (int) ev.getY(pointerIndex);
    final int yDiff = Math.abs(y - mLastMotionY);
    if (yDiff > mTouchSlop
            && (getNestedScrollAxes() & ViewCompat.SCROLL_AXIS_VERTICAL) == 0) {
        mIsBeingDragged = true;
        mLastMotionY = y;
        initVelocityTrackerIfNotExists();
        mVelocityTracker.addMovement(ev);
        mNestedYOffset = 0;
        final ViewParent parent = getParent();
        if (parent != null) {
            parent.requestDisallowInterceptTouchEvent(true);
        }
    }
    break;
}
{% endhighlight %}

`getNestedScrollAxes() & ViewCompat.SCROLL_AXIS_VERTICAL) == 0`，上面说到了父控件的滚动轴支持嵌套滑动。    

所以父nsv不拦截move事件，而子nsv会拦截move事件。    

所有的move事件最终都由子nsv消费。

{% highligh java %}
case MotionEvent.ACTION_MOVE:
    final int activePointerIndex = ev.findPointerIndex(mActivePointerId);
    if (activePointerIndex == -1) {
        Log.e(TAG, "Invalid pointerId=" + mActivePointerId + " in onTouchEvent");
        break;
    }

    final int y = (int) ev.getY(activePointerIndex);
    int deltaY = mLastMotionY - y;
    if (!mIsBeingDragged && Math.abs(deltaY) > mTouchSlop) {
        final ViewParent parent = getParent();
        if (parent != null) {
            parent.requestDisallowInterceptTouchEvent(true);
        }
        mIsBeingDragged = true;
        if (deltaY > 0) {
            deltaY -= mTouchSlop;
        } else {
            deltaY += mTouchSlop;
        }
    }
    if (mIsBeingDragged) {
        // Start with nested pre scrolling
        if (dispatchNestedPreScroll(0, deltaY, mScrollConsumed, mScrollOffset,
                ViewCompat.TYPE_TOUCH)) {
            deltaY -= mScrollConsumed[1];
            mNestedYOffset += mScrollOffset[1];
        }

        // Scroll to follow the motion event
        mLastMotionY = y - mScrollOffset[1];

        final int oldY = getScrollY();
        final int range = getScrollRange();
        final int overscrollMode = getOverScrollMode();
        boolean canOverscroll = overscrollMode == View.OVER_SCROLL_ALWAYS
                || (overscrollMode == View.OVER_SCROLL_IF_CONTENT_SCROLLS && range > 0);

        // Calling overScrollByCompat will call onOverScrolled, which
        // calls onScrollChanged if applicable.
        if (overScrollByCompat(0, deltaY, 0, getScrollY(), 0, range, 0,
                0, true) && !hasNestedScrollingParent(ViewCompat.TYPE_TOUCH)) {
            // Break our velocity if we hit a scroll barrier.
            mVelocityTracker.clear();
        }

        final int scrolledDeltaY = getScrollY() - oldY;
        final int unconsumedY = deltaY - scrolledDeltaY;

        mScrollConsumed[1] = 0;

        dispatchNestedScroll(0, scrolledDeltaY, 0, unconsumedY, mScrollOffset,
                ViewCompat.TYPE_TOUCH, mScrollConsumed);

        mLastMotionY -= mScrollOffset[1];
        mNestedYOffset += mScrollOffset[1];

        if (canOverscroll) {
            deltaY -= mScrollConsumed[1];
            ensureGlows();
            final int pulledToY = oldY + deltaY;
            if (pulledToY < 0) {
                EdgeEffectCompat.onPull(mEdgeGlowTop, (float) deltaY / getHeight(),
                        ev.getX(activePointerIndex) / getWidth());
                if (!mEdgeGlowBottom.isFinished()) {
                    mEdgeGlowBottom.onRelease();
                }
            } else if (pulledToY > range) {
                EdgeEffectCompat.onPull(mEdgeGlowBottom, (float) deltaY / getHeight(),
                        1.f - ev.getX(activePointerIndex)
                                / getWidth());
                if (!mEdgeGlowTop.isFinished()) {
                    mEdgeGlowTop.onRelease();
                }
            }
            if (mEdgeGlowTop != null
                    && (!mEdgeGlowTop.isFinished() || !mEdgeGlowBottom.isFinished())) {
                ViewCompat.postInvalidateOnAnimation(this);
            }
        }
    }
    break;
{% endhighlight %}

调用`overScrollByCompat`前会去调用`dispatchNestedPreScroll`，我们的父nsv在preScroll里是空操作。    

接着调用`dispatchNestedScroll`让父nsv消费掉剩余的scrollY；

{% highlight java %}
private boolean dispatchNestedScrollInternal(int dxConsumed, int dyConsumed,
            int dxUnconsumed, int dyUnconsumed, @Nullable int[] offsetInWindow,
            @NestedScrollType int type, @Nullable int[] consumed) {
        if (isNestedScrollingEnabled()) {
            final ViewParent parent = getNestedScrollingParentForType(type);
            if (parent == null) {
                return false;
            }

            if (dxConsumed != 0 || dyConsumed != 0 || dxUnconsumed != 0 || dyUnconsumed != 0) {
                int startX = 0;
                int startY = 0;
                if (offsetInWindow != null) {
                    mView.getLocationInWindow(offsetInWindow);
                    startX = offsetInWindow[0];
                    startY = offsetInWindow[1];
                }

                if (consumed == null) {
                    consumed = getTempNestedScrollConsumed();
                    consumed[0] = 0;
                    consumed[1] = 0;
                }

                ViewParentCompat.onNestedScroll(parent, mView,
                        dxConsumed, dyConsumed, dxUnconsumed, dyUnconsumed, type, consumed);

                if (offsetInWindow != null) {
                    mView.getLocationInWindow(offsetInWindow);
                    offsetInWindow[0] -= startX;
                    offsetInWindow[1] -= startY;
                }
                return true;
            } else if (offsetInWindow != null) {
                // No motion, no dispatch. Keep offsetInWindow up to date.
                offsetInWindow[0] = 0;
                offsetInWindow[1] = 0;
            }
        }
        return false;
    }
{% endhighlight %}

`ViewParentCompat.onNestedScroll`就是父nsv滚动的代码