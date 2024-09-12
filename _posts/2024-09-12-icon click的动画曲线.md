---
layout: post
title:  "icon click动画分析"
date:   2024-09-12 20:50:00 +0800
tags:   fwk
description:
---

具体流程：
1.点击事件传递给 Launcher：当用户点击 Launcher 中的图标，Launcher 会通过 Intent 请求启动相应的应用或 Activity。

2.启动 Activity 和动画请求：
Launcher 通过 ActivityOptions.makeRemoteAnimation() 创建了一个包含 RemoteAnimationAdapter 的 ActivityOptions，并将其与启动的 Intent 绑定。
这会告诉系统，这个 Activity 的启动动画将由远程动画（Remote Animation）来控制，而不是使用默认的过渡动画。

3.系统处理启动请求：
当系统（ActivityManagerService）接收到启动 Activity 的请求后，它会与 WindowManagerService (WMS) 协作，管理窗口和动画的显示。

4.触发远程动画回调：
WMS 负责窗口的管理和动画控制。在检测到某个 Activity 需要通过远程动画进行过渡时，WMS 会调用绑定的 RemoteAnimationRunner（即 RemoteAnimationRunnerCompat）来执行动画。
在此过程中，WMS 会调用 RemoteAnimationRunnerCompat 的 onAnimationStart() 方法，传递动画相关的 RemoteAnimationTarget，告诉 RemoteAnimationRunner 该如何控制窗口的过渡动画。

5.ActivityTaskManagerService 调用动画相关的 RemoteAnimationController
ActivityTaskManagerService 是启动 Activity 的核心管理器。在启动 Activity 的过程中，如果发现存在远程动画，则会创建一个 RemoteAnimationController 来处理动画。这部分代码可以在 ActivityTaskManagerService.java 文件中找到。

`WindowContainerTransaction.java`
```java
void applyAnimation(...) {
    if (mRemoteAnimationAdapter != null) {
        mAnimationAdapter = new RemoteAnimationController(mContext, mRemoteAnimationAdapter);
    }
}
```

`RemoteAnimationController.java`
```java
    void goodToGo(@WindowManager.TransitionOldType int transit) {
        ProtoLog.d(WM_DEBUG_REMOTE_ANIMATIONS, "goodToGo()");
        if (mCanceled) {
            ProtoLog.d(WM_DEBUG_REMOTE_ANIMATIONS,
                    "goodToGo(): Animation canceled already");
            onAnimationFinished();
            invokeAnimationCancelled("already_cancelled");
            return;
        }

        // Scale the timeout with the animator scale the controlling app is using.
        mHandler.postDelayed(mTimeoutRunnable,
                (long) (TIMEOUT_MS * mService.getCurrentAnimatorScale()));
        mFinishedCallback = new FinishedCallback(this);

        // Create the app targets
        final RemoteAnimationTarget[] appTargets = createAppAnimations();
        if (appTargets.length == 0 && !AppTransition.isKeyguardOccludeTransitOld(transit)) {
            // Keyguard occlude transition can be executed before the occluding activity becomes
            // visible. Even in this case, KeyguardService expects to receive binder call, so we
            // don't cancel remote animation.
            ProtoLog.d(WM_DEBUG_REMOTE_ANIMATIONS,
                    "goodToGo(): No apps to animate, mPendingAnimations=%d",
                    mPendingAnimations.size());
            onAnimationFinished();
            invokeAnimationCancelled("no_app_targets");
            return;
        }

        if (mOnRemoteAnimationReady != null) {
            mOnRemoteAnimationReady.run();
            mOnRemoteAnimationReady = null;
        }

        // Create the remote wallpaper animation targets (if any)
        final RemoteAnimationTarget[] wallpaperTargets = createWallpaperAnimations();

        // Create the remote non app animation targets (if any)
        final RemoteAnimationTarget[] nonAppTargets = createNonAppWindowAnimations(transit);

        mService.mAnimator.addAfterPrepareSurfacesRunnable(() -> {
            try {
                linkToDeathOfRunner();
                ProtoLog.d(WM_DEBUG_REMOTE_ANIMATIONS, "goodToGo(): onAnimationStart,"
                                + " transit=%s, apps=%d, wallpapers=%d, nonApps=%d",
                        AppTransition.appTransitionOldToString(transit), appTargets.length,
                        wallpaperTargets.length, nonAppTargets.length);
                if (AppTransition.isKeyguardOccludeTransitOld(transit)) {
                    EventLogTags.writeWmSetKeyguardOccluded(
                            transit == TRANSIT_OLD_KEYGUARD_UNOCCLUDE ? 0 : 1,
                            1 /* animate */,
                            transit,
                            "onAnimationStart");
                }
                mRemoteAnimationAdapter.getRunner().onAnimationStart(transit, appTargets,
                        wallpaperTargets, nonAppTargets, mFinishedCallback);
            } catch (RemoteException e) {
                Slog.e(TAG, "Failed to start remote animation", e);
                onAnimationFinished();
            }
            if (ProtoLog.isEnabled(WM_DEBUG_REMOTE_ANIMATIONS, LogLevel.DEBUG)) {
                ProtoLog.d(WM_DEBUG_REMOTE_ANIMATIONS, "startAnimation(): Notify animation start:");
                writeStartDebugStatement();
            }
        });
        setRunningRemoteAnimation(true);
    }
```

`mService.mAnimator.addAfterPrepareSurfacesRunnable()`从字面意思可以大概猜测，在Surface Prepare好
后添加回调，回调中会执行`mRemoteAnimationAdapter.getRunner().onAnimationStart`。这样即是从fwk层回调到了应用层。

通过分析`WindowAnimator.addAfterPrepareSurfacesRunnable`方法，可以大概知道其流程。

```java
 private void animate(long frameTimeNs) {
        if (!mInitialized) {
            return;
        }

        // Schedule next frame already such that back-pressure happens continuously.
        scheduleAnimation();

        final RootWindowContainer root = mService.mRoot;
        final boolean useShellTransition = root.mTransitionController.isShellTransitionsEnabled();
        final int animationFlags = useShellTransition ? CHILDREN : (TRANSITION | CHILDREN);
        boolean rootAnimating = false;
        mCurrentTime = frameTimeNs / TimeUtils.NANOS_PER_MS;
        mBulkUpdateParams = 0;
        root.mOrientationChangeComplete = true;
        if (DEBUG_WINDOW_TRACE) {
            Slog.i(TAG, "!!! animate: entry time=" + mCurrentTime);
        }

        ProtoLog.i(WM_SHOW_TRANSACTIONS, ">>> OPEN TRANSACTION animate");
        try {
            // Remove all deferred displays, tasks, and activities.
            root.handleCompleteDeferredRemoval();

            final AccessibilityController accessibilityController =
                    mService.mAccessibilityController;
            final int numDisplays = root.getChildCount();
            for (int i = 0; i < numDisplays; i++) {
                final DisplayContent dc = root.getChildAt(i);
                // Update animations of all applications, including those associated with
                // exiting/removed apps.
                dc.updateWindowsForAnimator();
                dc.prepareSurfaces();
            }

            for (int i = 0; i < numDisplays; i++) {
                final DisplayContent dc = root.getChildAt(i);

                if (!useShellTransition) {
                    dc.checkAppWindowsReadyToShow();
                }
                if (accessibilityController.hasCallbacks()) {
                    accessibilityController
                            .recomputeMagnifiedRegionAndDrawMagnifiedRegionBorderIfNeeded(
                                    dc.mDisplayId);
                }

                if (dc.isAnimating(animationFlags, ANIMATION_TYPE_ALL)) {
                    rootAnimating = true;
                    if (!dc.mLastContainsRunningSurfaceAnimator) {
                        dc.mLastContainsRunningSurfaceAnimator = true;
                        dc.enableHighFrameRate(true);
                    }
                } else if (dc.mLastContainsRunningSurfaceAnimator) {
                    dc.mLastContainsRunningSurfaceAnimator = false;
                    dc.enableHighFrameRate(false);
                }
                mTransaction.merge(dc.getPendingTransaction());
            }

            cancelAnimation();

            if (mService.mWatermark != null) {
                mService.mWatermark.drawIfNeeded();
            }

        } catch (RuntimeException e) {
            Slog.wtf(TAG, "Unhandled exception in Window Manager", e);
        }

        final boolean hasPendingLayoutChanges = root.hasPendingLayoutChanges(this);
        final boolean doRequest = (mBulkUpdateParams != 0 || root.mOrientationChangeComplete)
                && root.copyAnimToLayoutParams();
        if (hasPendingLayoutChanges || doRequest) {
            mService.mWindowPlacerLocked.requestTraversal();
        }

        if (rootAnimating && !mLastRootAnimating) {
            Trace.asyncTraceBegin(Trace.TRACE_TAG_WINDOW_MANAGER, "animating", 0);
        }
        if (!rootAnimating && mLastRootAnimating) {
            mService.mWindowPlacerLocked.requestTraversal();
            Trace.asyncTraceEnd(Trace.TRACE_TAG_WINDOW_MANAGER, "animating", 0);
        }
        mLastRootAnimating = rootAnimating;

        // APP_TRANSITION, SCREEN_ROTATION, TYPE_RECENTS are handled by shell transition.
        if (!useShellTransition) {
            updateRunningExpensiveAnimationsLegacy();
        }

        Trace.traceBegin(Trace.TRACE_TAG_WINDOW_MANAGER, "applyTransaction");
        mTransaction.apply();
        Trace.traceEnd(Trace.TRACE_TAG_WINDOW_MANAGER);
        mService.mWindowTracing.logState("WindowAnimator");
        ProtoLog.i(WM_SHOW_TRANSACTIONS, "<<< CLOSE TRANSACTION animate");

        mService.mAtmService.mTaskOrganizerController.dispatchPendingEvents();
        executeAfterPrepareSurfacesRunnables();

        if (DEBUG_WINDOW_TRACE) {
            Slog.i(TAG, "!!! animate: exit"
                    + " mBulkUpdateParams=" + Integer.toHexString(mBulkUpdateParams)
                    + " hasPendingLayoutChanges=" + hasPendingLayoutChanges);
        }
    }


void addAfterPrepareSurfacesRunnable(Runnable r) {
        // If runnables are already being handled in executeAfterPrepareSurfacesRunnable, then just
        // immediately execute the runnable passed in.
        if (mInExecuteAfterPrepareSurfacesRunnables) {
            r.run();
            return;
        }

        mAfterPrepareSurfacesRunnables.add(r);
        scheduleAnimation();
    }


void scheduleAnimation() {
        if (!mAnimationFrameCallbackScheduled) {
            mAnimationFrameCallbackScheduled = true;
            mChoreographer.postFrameCallback(mAnimationFrameCallback);
        }
    }


WindowAnimator(final WindowManagerService service) {
        mService = service;
        mContext = service.mContext;
        mPolicy = service.mPolicy;
        mTransaction = service.mTransactionFactory.get();
        service.mAnimationHandler.runWithScissors(
                () -> mChoreographer = Choreographer.getSfInstance(), 0 /* timeout */);

        mAnimationFrameCallback = frameTimeNs -> {
            synchronized (mService.mGlobalLock) {
                mAnimationFrameCallbackScheduled = false;
                animate(frameTimeNs);
                if (mNotifyWhenNoAnimation && !mLastRootAnimating) {
                    mService.mGlobalLock.notifyAll();
                }
            }
        };
    }

private void cancelAnimation() {
        if (mAnimationFrameCallbackScheduled) {
            mAnimationFrameCallbackScheduled = false;
            mChoreographer.removeFrameCallback(mAnimationFrameCallback);
        }
    }

```

`animate()`方法很有欺骗性，其实它并没有做动画动作，因为在它的后程调用了cancelAnimation。
所以这一段代码其实就是给 mChoreographer注册动画回调来获取应用的第一帧。

第一帧获取后，就开始调用addAfterPrepareSurfacesRunnable中的回调，就是我们关键的
`mRemoteAnimationAdapter.getRunner().onAnimationStart`方法。