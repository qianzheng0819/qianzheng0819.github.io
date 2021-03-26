---
layout: post
title:  "ScrollView源码浅谈"
date:   2021-03-8 15:33:00 +0800
categories: android
tags:   View
description:
---

### **前言**
scrollview的核心是Scroller类，还有View的draw()方法里调用的computeScroll()方法。一般在自定义控件里使用Scroller类时，都要配合重写
View的computeScroll()方法。下面我们分析一下ScrollView这个官方自定义控件实现的具体代码。本文源码使用android2.3.7版本，低版本的代码结构简单，方便分析。

### **onInterceptTouchEvent**
ScrollView作为一个viewGroup自然是要重写这个方法来决定自己是否拦截触摸事件啦。先上源码，容易理解的代码就直接在注释里分析啦。
{%highlight java%}
@Override
    public boolean onInterceptTouchEvent(MotionEvent ev) {
        /*
         * This method JUST determines whether we want to intercept the motion.
         * If we return true, onMotionEvent will be called and we do the actual
         * scrolling there.
         */

        /*
        * Shortcut the most recurring case: the user is in the dragging
        * state and he is moving his finger.  We want to intercept this
        * motion.
        */
        final int action = ev.getAction();
        if ((action == MotionEvent.ACTION_MOVE) && (mIsBeingDragged)) {
            return true;
        }

        switch (action & MotionEvent.ACTION_MASK) {
            case MotionEvent.ACTION_MOVE: {
                /*
                 * mIsBeingDragged == false, otherwise the shortcut would have caught it. Check
                 * whether the user has moved far enough from his original down touch.
                 */

                /*
                * Locally do absolute value. mLastMotionY is set to the y value
                * of the down event.
                */
                final int activePointerId = mActivePointerId;
                if (activePointerId == INVALID_POINTER) {
                    // If we don't have a valid id, the touch down wasn't on content.
                    break;
                }

                final int pointerIndex = ev.findPointerIndex(activePointerId);
                final float y = ev.getY(pointerIndex);
                final int yDiff = (int) Math.abs(y - mLastMotionY);
                if (yDiff > mTouchSlop) {
                    mIsBeingDragged = true;
                    mLastMotionY = y;
                }
                break;
            }

            case MotionEvent.ACTION_DOWN: {
                final float y = ev.getY();
                if (!inChild((int) ev.getX(), (int) y)) {
                    mIsBeingDragged = false;
                    break;
                }

                /*
                 * Remember location of down touch.
                 * ACTION_DOWN always refers to pointer index 0.
                 */
                mLastMotionY = y;
                mActivePointerId = ev.getPointerId(0);

                /*
                * If being flinged and user touches the screen, initiate drag;
                * otherwise don't.  mScroller.isFinished should be false when
                * being flinged.
                */
                mIsBeingDragged = !mScroller.isFinished();
                break;
            }

            case MotionEvent.ACTION_CANCEL:
            case MotionEvent.ACTION_UP:
                /* Release the drag */
                mIsBeingDragged = false;
                mActivePointerId = INVALID_POINTER;
                if (mScroller.springBack(mScrollX, mScrollY, 0, 0, 0, getScrollRange())) {
                    invalidate();
                }
                break;
            case MotionEvent.ACTION_POINTER_UP:
                onSecondaryPointerUp(ev);
                break;
        }

        /*
        * The only time we want to intercept motion events is if we are in the
        * drag mode.
        */
        return mIsBeingDragged;
    }
{%endhighlight%}

下面分别对actionDown,actionMove,actionUp这三个事件进行分析。分析就以注释的方式写在代码里吧，也可以结合google源英文注释进行理解。
{%highlight java%}
if ((action == MotionEvent.ACTION_MOVE) && (mIsBeingDragged)) {
            return true;
        }
....

case MotionEvent.ACTION_MOVE: {
  /*
   * mIsBeingDragged == false, otherwise the shortcut would have caught it. Check
   * whether the user has moved far enough from his original down touch.
   */

  /*
  * Locally do absolute value. mLastMotionY is set to the y value
  * of the down event.
  */
  final int activePointerId = mActivePointerId;
  if (activePointerId == INVALID_POINTER) {
      // If we don't have a valid id, the touch down wasn't on content.
      break;
  }

  final int pointerIndex = ev.findPointerIndex(activePointerId);
  final float y = ev.getY(pointerIndex);
  final int yDiff = (int) Math.abs(y - mLastMotionY);
  if (yDiff > mTouchSlop) {
      mIsBeingDragged = true;
      mLastMotionY = y;
  }
  break;
}
{%endhighlight%}
如果处于drag模式，我们对于actionMove进行拦截，方便传递给onTouchEvent()来进行滚动。
如果还未处于drag模式，但是yDiff > mTouchSlop，这个时候可以认为drag模式被触发了。宏观来说就是用户拖动了一段距离。TOUCH_SLOP是在
ViewConfigration类里定义，是16px。drag模式触发后，我们也要开始拦截actionMove事件。接下来看actionDown事件
{%highlight java%}
case MotionEvent.ACTION_DOWN: {
                final float y = ev.getY();
                if (!inChild((int) ev.getX(), (int) y)) {
                    mIsBeingDragged = false;
                    break;
                }

                /*
                 * Remember location of down touch.
                 * ACTION_DOWN always refers to pointer index 0.
                 */
                mLastMotionY = y;
                mActivePointerId = ev.getPointerId(0);

                /*
                * If being flinged and user touches the screen, initiate drag;
                * otherwise don't.  mScroller.isFinished should be false when
                * being flinged.
                */
                mIsBeingDragged = !mScroller.isFinished();
                break;
            }
{%endhighlight%}
点击位置不在子控件里面，mIsBeingDragged = false，不拦截该事件。如果当前处于fling模式，那么mIsBeingDragged = !mScroller.isFinished()
返回为true,即拦截该事件，想必是为fling模式下触摸停止的功能做铺垫，后续的代码里我们会验证这个猜想。接下来看actionUp事件
{%highlight java%}
case MotionEvent.ACTION_UP:
                /* Release the drag */
                mIsBeingDragged = false;
                mActivePointerId = INVALID_POINTER;
                if (mScroller.springBack(mScrollX, mScrollY, 0, 0, 0, getScrollRange())) {
                    invalidate();
                }
                break;
{%endhighlight%}
对于actionUp事件，直接mIsBeingDragged = false，即脱离drag模式。也不进行拦截。mScroller.springBack是针对滑动回弹的情形，在没有回弹的
情况下是返回false的，既不会有重绘操作。常规情况下，actionUp事件就是脱离drag模式，并清空激活的触摸点mActivePointerId。

分析了拦截方法，接下来就得看onTouchEvent方法了，在我的“也谈Android事件分发”文章中已经详细的描述了事件分发的原理。

对于onTouchEvent方法我们主要分析actionMove事件，其他的事件源码读者有空可以自己阅读。
{%highlight java%}
case MotionEvent.ACTION_MOVE:
    if (mIsBeingDragged) {
        // Scroll to follow the motion event
        final int activePointerIndex = ev.findPointerIndex(mActivePointerId); // 找到有效接触点
        final float y = ev.getY(activePointerIndex);
        final int deltaY = (int) (mLastMotionY - y);
        mLastMotionY = y;

        final int oldX = mScrollX;
        final int oldY = mScrollY;
        final int range = getScrollRange();
        if (overScrollBy(0, deltaY, 0, mScrollY, 0, range,
                0, mOverscrollDistance, true)) {
            // Break our velocity if we hit a scroll barrier.
            mVelocityTracker.clear();
        }
        onScrollChanged(mScrollX, mScrollY, oldX, oldY);
    ....
{%endhighlight%}
关键方法是overScrollBy(),overScrollBy是View类的方法。
{%highlight java%}
protected boolean overScrollBy(int deltaX, int deltaY,
            int scrollX, int scrollY,
            int scrollRangeX, int scrollRangeY,
            int maxOverScrollX, int maxOverScrollY,
            boolean isTouchEvent) {
        final int overScrollMode = mOverScrollMode;
        final boolean canScrollHorizontal =
                computeHorizontalScrollRange() > computeHorizontalScrollExtent();
        final boolean canScrollVertical =
                computeVerticalScrollRange() > computeVerticalScrollExtent();
        final boolean overScrollHorizontal = overScrollMode == OVER_SCROLL_ALWAYS ||
                (overScrollMode == OVER_SCROLL_IF_CONTENT_SCROLLS && canScrollHorizontal);
        final boolean overScrollVertical = overScrollMode == OVER_SCROLL_ALWAYS ||
                (overScrollMode == OVER_SCROLL_IF_CONTENT_SCROLLS && canScrollVertical);

        int newScrollX = scrollX + deltaX;
        if (!overScrollHorizontal) {
            maxOverScrollX = 0;
        }

        int newScrollY = scrollY + deltaY;
        if (!overScrollVertical) {
            maxOverScrollY = 0;
        }

        // Clamp values if at the limits and record
        final int left = -maxOverScrollX;
        final int right = maxOverScrollX + scrollRangeX;
        final int top = -maxOverScrollY;
        final int bottom = maxOverScrollY + scrollRangeY;

        boolean clampedX = false;
        if (newScrollX > right) {
            newScrollX = right;
            clampedX = true;
        } else if (newScrollX < left) {
            newScrollX = left;
            clampedX = true;
        }

        boolean clampedY = false;
        if (newScrollY > bottom) {
            newScrollY = bottom;
            clampedY = true;
        } else if (newScrollY < top) {
            newScrollY = top;
            clampedY = true;
        }

        onOverScrolled(newScrollX, newScrollY, clampedX, clampedY);

        return clampedX || clampedY;
    }
{%endhighlight%}
overScrollBy其实就是检查你有没有一滑到底（边界）。但是其中关键是他调用了onOverScrolled。
view的onOverScrolled是一个空的实现，那我们去看下ScrollView控件是如何实现的
{%highlight java%}
@Override
  protected void onOverScrolled(int scrollX, int scrollY,
          boolean clampedX, boolean clampedY) {
      // Treat animating scrolls differently; see #computeScroll() for why.
      if (!mScroller.isFinished()) {
          mScrollX = scrollX;
          mScrollY = scrollY;
          if (clampedY) {
              mScroller.springBack(mScrollX, mScrollY, 0, 0, 0, getScrollRange());
          }
      } else {
          super.scrollTo(scrollX, scrollY);
      }
      awakenScrollBars();
  }
{%endhighlight%}
mScroller.isFinished()其实标记的就是fling模式，后续我们可以验证这个观点。显然我们在处理actionMove事件，是处于drag模式，而非
fling模式的。所以，其实是走的super.scrollTo(scrollX, scrollY)分支了。看下view.scrollTo方法
{%highlight java%}
public void scrollTo(int x, int y) {
        if (mScrollX != x || mScrollY != y) {
            int oldX = mScrollX;
            int oldY = mScrollY;
            mScrollX = x;
            mScrollY = y;
            onScrollChanged(mScrollX, mScrollY, oldX, oldY);
            if (!awakenScrollBars()) {
                invalidate();
            }
        }
    }
{%endhighlight%}
代码非常的简单，但是属于核心代码。作用就是更新mScrollY,然后通过invalidate()进行重绘。而所谓view的滚动，它的原理就是更新mScrollY然后
刷新页面。

由于actionMove事件都是连续的，而且每次y轴的变化量不大，所以在drag模式下,不会突然一下滚动到目标位置。但是scrollTo也是View类的一个
公开api,当目标位置和当前位置差值较大时，会突然滚动过去。往往我们需要有一个缓缓滚动过去的动画，这个时候ScrollView的特殊Api smoothScrollTo
就派上用场了。     

smoothScrollTo的核心类就是Scroller类了，也是接下来重点要分析的一个类。  
先看看ScrollView的smoothScrollTo方法
{%highlight java%}
public final void smoothScrollBy(int dx, int dy) {
    if (getChildCount() == 0) {
        // Nothing to do.
        return;
    }
    long duration = AnimationUtils.currentAnimationTimeMillis() - mLastScroll;
    if (duration > ANIMATED_SCROLL_GAP) {
        final int height = getHeight() - mPaddingBottom - mPaddingTop;
        final int bottom = getChildAt(0).getHeight();
        final int maxY = Math.max(0, bottom - height);
        final int scrollY = mScrollY;
        dy = Math.max(0, Math.min(scrollY + dy, maxY)) - scrollY;

        mScroller.startScroll(mScrollX, scrollY, 0, dy);
        invalidate();
    } else {
        if (!mScroller.isFinished()) {
            mScroller.abortAnimation();
        }
        scrollBy(dx, dy);
    }
    mLastScroll = AnimationUtils.currentAnimationTimeMillis();
}
{%endhighlight%}
核心方法只有两句
{%highlight java%}
  mScroller.startScroll(mScrollX, scrollY, 0, dy);
  invalidate();
{%endhighlight%}
先来看mScroller.startScroll做了什么
{%highlight java%}
public void startScroll(int startX, int startY, int dx, int dy, int duration) {
    mMode = SCROLL_MODE;
    mScrollerX.startScroll(startX, dx, duration);
    mScrollerY.startScroll(startY, dy, duration);
}

// OverScroller内部类MagneticOverScroller的方法
void startScroll(int start, int distance, int duration) {
    mFinished = false;

    mStart = start;
    mFinal = start + distance;

    mStartTime = AnimationUtils.currentAnimationTimeMillis();
    mDuration = duration;

    // Unused
    mDeceleration = 0.0f;
    mVelocity = 0;
}
{%endhighlight%}
这里面好像没干什么事情，无非记录了下开始的位置，结束的位置，动画开始的事件，动画持续的时间等属性。   
那么实现动画滚动的代码到底在哪呢。其实这里需要你对View类的绘制源码有一定的了解，需要有一个对源码的熟悉度，我就直接上源码吧   
ViewGroup的drawChild方法里
{%highlight java%}
 protected boolean drawChild(Canvas canvas, View child, long drawingTime) {
  ....
  // Sets the flag as early as possible to allow draw() implementations
      // to call invalidate() successfully when doing animations
      child.mPrivateFlags |= DRAWN;

      if (!concatMatrix && canvas.quickReject(cl, ct, cr, cb, Canvas.EdgeType.BW) &&
              (child.mPrivateFlags & DRAW_ANIMATION) == 0) {
          return more;
      }

      child.computeScroll();

      final int sx = child.mScrollX;
      final int sy = child.mScrollY;
  ....
}
{%endhighlight%}
核心就是child.computeScroll()这个方法了。后续可以对view的绘制体系做一篇源码分析，主要是涉及的内容太多了。  
也就是每次重绘时，viewGroup在绘制child时都会计算child的scroll值。那么接下来就简单了，我们看下ScrollView是如何实现computeScroll方法吧
{%highlight java%}
@Override
    public void computeScroll() {
        if (mScroller.computeScrollOffset()) {
            // This is called at drawing time by ViewGroup.  We don't want to
            // re-show the scrollbars at this point, which scrollTo will do,
            // so we replicate most of scrollTo here.
            //
            //         It's a little odd to call onScrollChanged from inside the drawing.
            //
            //         It is, except when you remember that computeScroll() is used to
            //         animate scrolling. So unless we want to defer the onScrollChanged()
            //         until the end of the animated scrolling, we don't really have a
            //         choice here.
            //
            //         I agree.  The alternative, which I think would be worse, is to post
            //         something and tell the subclasses later.  This is bad because there
            //         will be a window where mScrollX/Y is different from what the app
            //         thinks it is.
            //
            int oldX = mScrollX;
            int oldY = mScrollY;
            int x = mScroller.getCurrX();
            int y = mScroller.getCurrY();

            if (oldX != x || oldY != y) {
                overScrollBy(x - oldX, y - oldY, oldX, oldY, 0, getScrollRange(),
                        0, mOverflingDistance, false);
                onScrollChanged(mScrollX, mScrollY, oldX, oldY);

                final int range = getScrollRange();
                final int overscrollMode = getOverScrollMode();
                if (overscrollMode == OVER_SCROLL_ALWAYS ||
                        (overscrollMode == OVER_SCROLL_IF_CONTENT_SCROLLS && range > 0)) {
                    if (y < 0 && oldY >= 0) {
                        mEdgeGlowTop.onAbsorb((int) mScroller.getCurrVelocity());
                    } else if (y > range && oldY <= range) {
                        mEdgeGlowBottom.onAbsorb((int) mScroller.getCurrVelocity());
                    }
                }
            }
            awakenScrollBars();

            // Keep on drawing until the animation has finished.
            postInvalidate();
        }
    }
{%endhighlight%}
核心代码也不过两句，overScrollBy和postInvalidate()。无需分析了。重点关注下mScroller.computeScrollOffset()
{%highlight java%}
public boolean computeScrollOffset() {
        if (isFinished()) {
            return false;
        }

        switch (mMode) {
            case SCROLL_MODE:
                long time = AnimationUtils.currentAnimationTimeMillis();
                // Any scroller can be used for time, since they were started
                // together in scroll mode. We use X here.
                final long elapsedTime = time - mScrollerX.mStartTime;

                final int duration = mScrollerX.mDuration;
                if (elapsedTime < duration) {
                    float q = (float) (elapsedTime) / duration;

                    if (mInterpolator == null)
                        q = Scroller.viscousFluid(q);
                    else
                        q = mInterpolator.getInterpolation(q);

                    mScrollerX.updateScroll(q);
                    mScrollerY.updateScroll(q);
                } else {
                    abortAnimation();
                }
                break;

            case FLING_MODE:
                if (!mScrollerX.mFinished) {
                    if (!mScrollerX.update()) {
                        if (!mScrollerX.continueWhenFinished()) {
                            mScrollerX.finish();
                        }
                    }
                }

                if (!mScrollerY.mFinished) {
                    if (!mScrollerY.update()) {
                        if (!mScrollerY.continueWhenFinished()) {
                            mScrollerY.finish();
                        }
                    }
                }

                break;
        }

        return true;
    }
{%endhighlight%}
mMode只有在用到mScrooler类的时候才会有值，所以像前面提到的onTouchEvent里面，mMode是没有值的，computeScrollOffset()是会直接返回true的。   
为什么mMode有值很重要呢，因为就是mScrollerY.updateScroll(q)实现了动态改变滑动的目标值，即我们的smoothScroll的动效。

那么到现在呢，对于ScrollView和Scroller类如何实现滑动，无论是走触摸事件，还是调用api smoothScrollTo。我们已经讲的非常清晰了。这篇博客就
到此为止了。
