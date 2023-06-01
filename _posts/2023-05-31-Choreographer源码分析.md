---
layout: post
title:  "Choreographer源码分析"
date:   2023-05-31 09:59:00 +0800
categories: android
tags:   Framework
description:
---

前言
-------------------------
源码版本是android4.4.4,该版本已经引入了Choreographer类。稍低版本的源码，主干功能更加清晰，方便阅读。

当我们滚动视图时，scrollView里的scroller类会不断更新mScrollY，然后调用invalidate()刷新，
我们看Choreographer是如何响应应用的请求。

View.invalidate()源码分析
--------------------------
{% highlight java %}
void invalidate(boolean invalidateCache) {
        if (skipInvalidate()) {
            return;
        }
        ......
        final ViewParent p = mParent;
        if (p != null && ai != null) {
            final Rect r = ai.mTmpInvalRect;
            r.set(0, 0, mRight - mLeft, mBottom - mTop);
            // Don't call invalidate -- we don't want to internally scroll
            // our own bounds
            p.invalidateChild(this, r);
        }
      }
    }
{% endhighlight %}

调用了父类ViewParent的invalidateChild()方法，子View的ViewParent就是它的父View即ViewGroup

{% highlight java %}
public final void invalidateChild(View child, final Rect dirty) {

  do {
        ............
        parent = parent.invalidateChildInParent(location, dirty);
        if (view != null) {
            // Account for transform on current parent
            Matrix m = view.getMatrix();
            if (!m.isIdentity()) {
                RectF boundingRect = attachInfo.mTmpTransformRect;
                boundingRect.set(dirty);
                m.mapRect(boundingRect);
                dirty.set((int) (boundingRect.left - 0.5f),
                        (int) (boundingRect.top - 0.5f),
                        (int) (boundingRect.right + 0.5f),
                        (int) (boundingRect.bottom + 0.5f));
            }
          }
      } while (parent != null);
}
{% endhighlight %}

循环调用parent.invalidateChildInParent，最终的parent为ViewRootImpl.
`ViewRootImpl#invalidateChildInParent`
{% highlight java %}
@Override
    public ViewParent invalidateChildInParent(int[] location, Rect dirty) {
        ......
        final float appScale = mAttachInfo.mApplicationScale;
        final boolean intersected = localDirty.intersect(0, 0,
                (int) (mWidth * appScale + 0.5f), (int) (mHeight * appScale + 0.5f));
        if (!intersected) {
            localDirty.setEmpty();
        }
        if (!mWillDrawSoon && (intersected || mIsAnimating)) {
            scheduleTraversals();
        }

        return null;
    }
{% endhighlight %}

可以看到最终调用了scheduleTraversals(),这里就是我们最为熟悉的ViewRootImpl的scheduleTraversals()方法了，
我们对比下android 2.3.7和android 4.4.4这里的源码实现

`android 2.3.7`
{% highlight java %}
  public void scheduleTraversals() {
      if (!mTraversalScheduled) {
          mTraversalScheduled = true;
          sendEmptyMessage(DO_TRAVERSAL);
      }
  }

  @Override
    public void handleMessage(Message msg) {
        switch (msg.what) {
        .......
        case DO_TRAVERSAL:
            if (mProfile) {
                Debug.startMethodTracing("ViewRoot");
            }

            performTraversals();

            if (mProfile) {
                Debug.stopMethodTracing();
                mProfile = false;
            }
            break;
{% endhighlight %}

`android 4.4.4`
{% highlight java %}
void scheduleTraversals() {
    if (!mTraversalScheduled) {
        mTraversalScheduled = true;
        mTraversalBarrier = mHandler.getLooper().postSyncBarrier();
        mChoreographer.postCallback(
                Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
        scheduleConsumeBatchedInput();
    }
}
{% endhighlight %}

Choreographer是在Android4.1版本引入。它的引入，主要是配合 Vsync，给上层 App 的渲染提供一个稳定的 Message 处理的时机。

Choreographer源码解析
----------------------
#### Choreographer 的单例初始化
{% highlight java %}
// Thread local storage for the choreographer.
private static final ThreadLocal<Choreographer> sThreadInstance =
        new ThreadLocal<Choreographer>() {
    @Override
    protected Choreographer initialValue() {
        // 获取当前线程的 Looper
        Looper looper = Looper.myLooper();
        ......
        // 构造 Choreographer 对象
        Choreographer choreographer = new Choreographer(looper, VSYNC_SOURCE_APP);
        if (looper == Looper.getMainLooper()) {
            mMainInstance = choreographer;
        }
        return choreographer;
    }
};
{% endhighlight %}

#### Choreographer 的构造函数
{% highlight java %}
private Choreographer(Looper looper, int vsyncSource) {
    mLooper = looper;
    // 1. 初始化 FrameHandler
    mHandler = new FrameHandler(looper);
    // 2. 初始化 DisplayEventReceiver
    mDisplayEventReceiver = USE_VSYNC
            ? new FrameDisplayEventReceiver(looper, vsyncSource)
            : null;
    mLastFrameTimeNanos = Long.MIN_VALUE;
    mFrameIntervalNanos = (long)(1000000000 / getRefreshRate());
    //3. 初始化 CallbacksQueues
    mCallbackQueues = new CallbackQueue[CALLBACK_LAST + 1];
    for (int i = 0; i <= CALLBACK_LAST; i++) {
        mCallbackQueues[i] = new CallbackQueue();
    }
    ......
}
{% endhighlight %}

#### FrameHandler
{% highlight java %}
private final class FrameHandler extends Handler {
    ......
    public void handleMessage(Message msg) {
        switch (msg.what) {
            case MSG_DO_FRAME://开始渲染下一帧的操作
                doFrame(System.nanoTime(), 0);
                break;
            case MSG_DO_SCHEDULE_VSYNC://请求 Vsync
                doScheduleVsync();
                break;
            case MSG_DO_SCHEDULE_CALLBACK://处理 Callback
                doScheduleCallback(msg.arg1);
                break;
        }
    }
}
{% endhighlight %}

#### Choreographer 初始化链
在 Activity 启动过程，执行完 onResume 后，会调用 Activity.makeVisible()，然后再调用到 addView()，
层层调用会进入如下方法
{% highlight java %}
ActivityThread.handleResumeActivity(IBinder, boolean, boolean, String) (android.app)
-->WindowManagerImpl.addView(View, LayoutParams) (android.view)
  -->WindowManagerGlobal.addView(View, LayoutParams, Display, Window) (android.view)
    -->ViewRootImpl.ViewRootImpl(Context, Display) (android.view)
    public ViewRootImpl(Context context, Display display) {
        ......
        mChoreographer = Choreographer.getInstance();
        ......
    }
{% endhighlight %}

Choreographer源码调用的时序图
------------------------------
![p]({{ site.baseurl }}/assets/images/2023-pic/p1.jpg)

#### Step3.postSyncBarrier
{% highlight java %}
void scheduleTraversals() {
    if (!mTraversalScheduled) {
        mTraversalScheduled = true;
        mTraversalBarrier = mHandler.getLooper().postSyncBarrier();
        mChoreographer.postCallback(
                Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
        scheduleConsumeBatchedInput();
    }
}
{% endhighlight %}

Handler的消息分3种，普通消息，异步消息和屏障消息

普通消息又叫做同步消息，屏障消息是屏障同步消息

`设置了同步屏障之后，Handler只会处理异步消息。再换句话说，同步屏障为Handler消息机制增加了一种简单的优先级机制，
异步消息的优先级要高于同步消息。`

源码数据结构并不复杂，具体分析可参考[博客](https://www.jianshu.com/p/ed318296f95f)

#### Step5,6
{% highlight java %}
void scheduleTraversals() {
    if (!mTraversalScheduled) {
        mTraversalScheduled = true;
        mTraversalBarrier = mHandler.getLooper().postSyncBarrier();
        mChoreographer.postCallback(
                Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
        scheduleConsumeBatchedInput();
    }
}

public void postCallback(int callbackType, Runnable action, Object token) {
        postCallbackDelayed(callbackType, action, token, 0);
    }

public void postCallbackDelayed(int callbackType,
          Runnable action, Object token, long delayMillis) {
      if (action == null) {
          throw new IllegalArgumentException("action must not be null");
      }
      if (callbackType < 0 || callbackType > CALLBACK_LAST) {
          throw new IllegalArgumentException("callbackType is invalid");
      }

      postCallbackDelayedInternal(callbackType, action, token, delayMillis);
  }

private void postCallbackDelayedInternal(int callbackType,
          Object action, Object token, long delayMillis) {
      if (DEBUG) {
          Log.d(TAG, "PostCallback: type=" + callbackType
                  + ", action=" + action + ", token=" + token
                  + ", delayMillis=" + delayMillis);
      }

      synchronized (mLock) {
          final long now = SystemClock.uptimeMillis();
          final long dueTime = now + delayMillis;
          mCallbackQueues[callbackType].addCallbackLocked(dueTime, action, token);

          if (dueTime <= now) {
              scheduleFrameLocked(now);
          } else {
              Message msg = mHandler.obtainMessage(MSG_DO_SCHEDULE_CALLBACK, action);
              msg.arg1 = callbackType;
              msg.setAsynchronous(true);
              mHandler.sendMessageAtTime(msg, dueTime);
          }
      }
  }
{% endhighlight %}
delayMillis = 0,所以dueTime = now,执行scheduleFrameLocked(now)

#### Step 7,8,9
{% highlight java %}
private void scheduleFrameLocked(long now) {
    if (!mFrameScheduled) {
        mFrameScheduled = true;
        if (USE_VSYNC) {
            if (DEBUG) {
                Log.d(TAG, "Scheduling next frame on vsync.");
            }

            // If running on the Looper thread, then schedule the vsync immediately,
            // otherwise post a message to schedule the vsync from the UI thread
            // as soon as possible.
            if (isRunningOnLooperThreadLocked()) {
                scheduleVsyncLocked();
            } else {
                Message msg = mHandler.obtainMessage(MSG_DO_SCHEDULE_VSYNC);
                msg.setAsynchronous(true);
                mHandler.sendMessageAtFrontOfQueue(msg);
            }
        } else {
            final long nextFrameTime = Math.max(
                    mLastFrameTimeNanos / NANOS_PER_MS + sFrameDelay, now);
            if (DEBUG) {
                Log.d(TAG, "Scheduling next frame in " + (nextFrameTime - now) + " ms.");
            }
            Message msg = mHandler.obtainMessage(MSG_DO_FRAME);
            msg.setAsynchronous(true);
            mHandler.sendMessageAtTime(msg, nextFrameTime);
        }
    }
}

....
private void scheduleVsyncLocked() {
        mDisplayEventReceiver.scheduleVsync();
    }

....

/**
  DisplayEventReceiver.class
  Schedules a single vertical sync pulse to be delivered when the next
  display frame begins.
 **/
public void scheduleVsync() {
    if (mReceiverPtr == 0) {
        Log.w(TAG, "Attempted to schedule a vertical sync pulse but the display event "
                + "receiver has already been disposed.");
    } else {
        nativeScheduleVsync(mReceiverPtr);
    }
}
{% endhighlight %}

USE_VSYNC为true,且是在ui线程，执行scheduleVsyncLocked()，然后依次执行mDisplayEventReceiver.scheduleVsync()。

这里的作用就是请求VSync-App信号。

ViewRootImpl的scheduleTraversals方法，简单来说`注册了Choreographer.CALLBACK_TRAVERSAL类型回调，并请求系统的VSync信号`

#### Step12
{% highlight java %}
@Override
public void onVsync(long timestampNanos, int builtInDisplayId, int frame) {
    // Post the vsync event to the Handler.
    // The idea is to prevent incoming vsync events from completely starving
    // the message queue.  If there are no messages in the queue with timestamps
    // earlier than the frame time, then the vsync event will be processed immediately.
    // Otherwise, messages that predate the vsync event will be handled first.
    long now = System.nanoTime();
    if (timestampNanos > now) {
        Log.w(TAG, "Frame time is " + ((timestampNanos - now) * 0.000001f)
                + " ms in the future!  Check that graphics HAL is generating vsync "
                + "timestamps using the correct timebase.");
        timestampNanos = now;
    }

    if (mHavePendingVsync) {
        Log.w(TAG, "Already have a pending vsync event.  There should only be "
                + "one at a time.");
    } else {
        mHavePendingVsync = true;
    }

    mTimestampNanos = timestampNanos;
    mFrame = frame;
    Message msg = Message.obtain(mHandler, this);
    msg.setAsynchronous(true);
    mHandler.sendMessageAtTime(msg, timestampNanos / NANOS_PER_MS);
}

@Override
public void run() {
    mHavePendingVsync = false;
    doFrame(mTimestampNanos, mFrame);
}
{% endhighlight %}
向handler发送了异步消息。handler处理改消息时，执行doFrame

#### Step15 doFrame
{% highlight java %}
void doFrame(long frameTimeNanos, int frame) {
        final long startNanos;
        synchronized (mLock) {
            if (!mFrameScheduled) {
                return; // no work to do
            }

            startNanos = System.nanoTime();
            final long jitterNanos = startNanos - frameTimeNanos;
            if (jitterNanos >= mFrameIntervalNanos) {
                final long skippedFrames = jitterNanos / mFrameIntervalNanos;
                if (skippedFrames >= SKIPPED_FRAME_WARNING_LIMIT) {
                    Log.i(TAG, "Skipped " + skippedFrames + " frames!  "
                            + "The application may be doing too much work on its main thread.");
                }
                final long lastFrameOffset = jitterNanos % mFrameIntervalNanos;
                if (DEBUG) {
                    Log.d(TAG, "Missed vsync by " + (jitterNanos * 0.000001f) + " ms "
                            + "which is more than the frame interval of "
                            + (mFrameIntervalNanos * 0.000001f) + " ms!  "
                            + "Skipping " + skippedFrames + " frames and setting frame "
                            + "time to " + (lastFrameOffset * 0.000001f) + " ms in the past.");
                }
                frameTimeNanos = startNanos - lastFrameOffset;
            }

            if (frameTimeNanos < mLastFrameTimeNanos) {
                if (DEBUG) {
                    Log.d(TAG, "Frame time appears to be going backwards.  May be due to a "
                            + "previously skipped frame.  Waiting for next vsync.");
                }
                scheduleVsyncLocked();
                return;
            }

            mFrameScheduled = false;
            mLastFrameTimeNanos = frameTimeNanos;
        }

        doCallbacks(Choreographer.CALLBACK_INPUT, frameTimeNanos);
        doCallbacks(Choreographer.CALLBACK_ANIMATION, frameTimeNanos);
        doCallbacks(Choreographer.CALLBACK_TRAVERSAL, frameTimeNanos);
    }
{% endhighlight %}

doFrame 函数主要做下面几件事

- 计算掉帧逻辑
- 记录帧绘制信息
- 执行 CALLBACK_INPUT、CALLBACK_ANIMATION、CALLBACK_INSETS_ANIMATION、CALLBACK_TRAVERSAL、CALLBACK_COMMIT

{% highlight java %}
final class TraversalRunnable implements Runnable {
  @Override
  public void run() {
      doTraversal();
  }
}
{% endhighlight %}
最后回调viewRootImpl之前传入的runnable TraversalRunnable,并执行我们熟悉的doTraversal()方法
