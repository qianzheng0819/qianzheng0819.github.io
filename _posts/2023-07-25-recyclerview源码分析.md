---
layout: post
title:  "RecyclerView源码分析"
date:   2023-07-25 10:48:00 +0800
tags:   android View
description:
---

> RecyclerView自从出道也有几年的光景了，大家对于它的赞扬一直络绎不绝，所以对于如此受欢迎的控件我们必须好好的了解一下。本文尝试带领大家更加全面的理解RecyclerView。

## 设计思路

RecyclerView官网给出的定义是`A flexible view for providing a limited window into a large data set.`，也就是在限定的试图内展示大量的数据，来一张通俗明了的图：

![](/assets/images/understand-recycler/o_1av9i0p3s1vlstit1ucr1qea2bs9.png)

RecyclerView的职责就是将Datas中的数据以一定的规则展示在它的上面，但说破天RecyclerView只是一个ViewGroup，它只认识View，不清楚Data数据的具体结构，所以两个陌生人之间想构建通话，我们很容易想到`适配器模式`,因此，RecyclerView需要一个Adapter来与Datas进行交流：

![](/assets/images/understand-recycler/o_1av9ij8ua17en59k1ahg13d1l7c9.png)

如上如所示，RecyclerView表示只会和ViewHolder进行接触，而Adapter的工作就是将Data转换为RecyclerView认识的ViewHolder，因此RecyclerView就间接地认识了Datas。

事情虽然进展愉快，但RecyclerView是个很懒惰的人，尽管Adapter已经将Datas转换为RecyclerView所熟知的View，但RecyclerView并不想自己管理些子View，因此，它雇佣了一个叫做LayoutManager的大祭司来帮其完成布局，现在，图示变成下面这样：

![/assets/images/understand-recycler/o_1av9iv5731k3422htsd14uess1j.png](/assets/images/understand-recycler/o_1av9iv5731k3422htsd14uess1j.png)

如上图所示，LayoutManager协助RecyclerView来完成布局。但LayoutManager这个大祭司也有弱点，就是它只知道如何将一个一个的View布局在RecyclerView上，但它并不懂得如何管理这些View，如果大祭司肆无忌惮的玩弄View的话肯定会出事情，所以，必须有个管理View的护法，它就是Recycler，LayoutManager在需要View的时候回向护法进行索取，当LayoutManager不需要View(试图滑出)的时候，就直接将废弃的View丢给Recycler，图示如下：

![](/assets/images/understand-recycler/o_1av9iujp712ctgik1slp1c7e1cg0e.png)

到了这里，有负责翻译数据的Adapter，有负责布局的LayoutManager，有负责管理View的Recycler，一切都很完美，但RecyclerView乃何等神也，它下令说当子View变动的时候姿态要优雅(动画)，所以用雇佣了一个舞者ItemAnimator，因此，舞者也进入了这个图示:

![](/assets/images/understand-recycler/o_1av9ji0c01s471j1c5t27vrvcge.png)

如上，我们就是从宏观层面来对RecylerView有个大致的了解，可以看到，RecyclerView作为一个View，它只负责接受用户的各种讯息，然后将信息各司其职的分发出去。接下来我们将深入源码，看看RecyclerView都是怎么来操作各个组件工作的。


## 源码分析

整个RecyclerView还是相当复杂的，我画了一个与RecyclerView相关类的脑图：

![](/assets/images/understand-recycler/o_1avejfume5og1i5s17qn11e474c9.png)

可见RecyclerView涉及的类相当多，所以看代码的时候很容易迷失。因此我们需要抽丝剥茧，按照主线来进行分析。

既然RecyclerView是一个View，那无外乎还是measure、layout、draw几个过程，所以我们依次来看一下。


### onMeasure

前面说过，RecyclerView会将测量与布局交给LayoutManager来做，并且LayoutManager有一个叫做mAutoMeasure的属性，这个属性用来控制LayoutManager是否开启自动测量，开启自动测量的话布局就交由RecyclerView使用一套默认的测量机制，否则，自定义的LayoutManager需要重写onMeasure来处理自身的测量工作。RecyclerView目前提供的几种LayoutManager都开启了自动测量，所以这里我们关注一下自动测量部分的逻辑：

```
if (mLayout.mAutoMeasure) {
    final int widthMode = MeasureSpec.getMode(widthSpec);
    final int heightMode = MeasureSpec.getMode(heightSpec);
    final boolean skipMeasure = widthMode == MeasureSpec.EXACTLY
            && heightMode == MeasureSpec.EXACTLY;
    mLayout.onMeasure(mRecycler, mState, widthSpec, heightSpec);
    if (skipMeasure || mAdapter == null) {
        return;
    }
    if (mState.mLayoutStep == State.STEP_START) {
        dispatchLayoutStep1();
    }
    mLayout.setMeasureSpecs(widthSpec, heightSpec);
    mState.mIsMeasuring = true;
    dispatchLayoutStep2();
    mLayout.setMeasuredDimensionFromChildren(widthSpec, heightSpec);
    if (mLayout.shouldMeasureTwice()) {
        mLayout.setMeasureSpecs(
                MeasureSpec.makeMeasureSpec(getMeasuredWidth(), MeasureSpec.EXACTLY),
                MeasureSpec.makeMeasureSpec(getMeasuredHeight(), MeasureSpec.EXACTLY));
        mState.mIsMeasuring = true;
        dispatchLayoutStep2();
        // now we can get the width and height from the children.
        mLayout.setMeasuredDimensionFromChildren(widthSpec, heightSpec);
    }
}
```

自动测量的原理如下:

当RecyclerView的宽高都为EXACTLY时，可以直接设置对应的宽高，然后返回，结束测量。

如果宽高的测量规则不是EXACTLY的,则会在onMeasure()中开始布局的处理，这里首先要介绍一个很重要的类：RecyclerView.State ，这个类封装了当前RecyclerView的有用信息。State的一个变量mLayoutStep表示了RecyclerView当前的布局状态，包括STEP_START、STEP_LAYOUT 、 STEP_ANIMATIONS三个，而RecyclerView的布局过程也分为三步，其中，STEP_START表示即将开始布局，需要调用dispatchLayoutStep1来执行第一步布局，接下来，布局状态变为STEP_LAYOUT，表示接下来需要调用dispatchLayoutStep2里进行第二步布局，同理，第二步布局后状态变为STEP_ANIMATIONS，需要执行第三步布局dispatchLayoutStep3。

这三个步骤的工作也各不相同，step1负责记录状态，step2负责布局，step3则与step1进行比较，根据变化来触发动画。

RecyclerView将布局划分的如此细致必然是有其原因的，在开启自动测量模式的情况，RecyclerView是支持WRAP_CONTENT属性的，比如我们可以很容易的在RecyclerView的下面放置其它的View，RecyclerView会根据子View所占大小动态调整自己的大小，这时候，RecyclerView就会将子控件的measure与layout提前到Recycler的onMeasure中，因为它需要确定子空间的大小与位置后，再来设置自己的大小。所以这时候就会在onMeasure中完成step1与step2。否则，就需要在onLayout中去完成整个布局过程。

综上，整个mLayout.mAutoMeasure就是在做前两步的布局，可见RecylerView的measure与layout是紧密相关的，所以我们来赶快瞧一瞧RecyclerView是如何layout的。

### onLayout

我们直接看下onLayout的代码：

```
protected void onLayout(boolean changed, int l, int t, int r, int b) {
    TraceCompat.beginSection(TRACE_ON_LAYOUT_TAG);
    dispatchLayout();
    TraceCompat.endSection();
    mFirstLayoutComplete = true;
}
```

直接追进dispatchLayout：

```
void dispatchLayout() {
	...
    mState.mIsMeasuring = false;
    if (mState.mLayoutStep == State.STEP_START) {
        dispatchLayoutStep1();
        mLayout.setExactMeasureSpecsFrom(this);
        dispatchLayoutStep2();
    } else if (mAdapterHelper.hasUpdates() || mLayout.getWidth() != getWidth() ||mLayout.getHeight() != getHeight()) {
        // First 2 steps are done in onMeasure but looks like we have to run again due to
        // changed size.
        mLayout.setExactMeasureSpecsFrom(this);
        dispatchLayoutStep2();
    } else {
        // always make sure we sync them (to ensure mode is exact)
        mLayout.setExactMeasureSpecsFrom(this);
    }
    dispatchLayoutStep3();
}
```
通过查看dispatchLayout的代码正好验证了我们前文关于RecyclerView的layout三步走原则，如果在onMeasure中已经完成了step1与step2，则只会执行step3，否则三步会依次触发。接下来我们一步一步的进行分析

#### dispatchLayoutStep1

```
private void dispatchLayoutStep1(){   
    ...
    if (mState.mRunSimpleAnimations) {
          int count = mChildHelper.getChildCount();
          for (int i = 0; i < count; ++i) {
              final ViewHolder holder = getChildViewHolderInt(mChildHelper.getChildAt(i));
              final ItemHolderInfo animationInfo = mItemAnimator
                      .recordPreLayoutInformation(mState, holder,
                              ItemAnimator.buildAdapterChangeFlagsForAnimations(holder),
                              holder.getUnmodifiedPayloads());
              mViewInfoStore.addToPreLayout(holder, animationInfo);
              ...
          }
    }
    ...
    mState.mLayoutStep = State.STEP_LAYOUT;
}
```

step的第一步目的就是在记录View的状态，首先遍历当前所有的View依次进行处理，mItemAnimator会根据每个View的信息封装成一个ItemHolderInfo，这个ItemHolderInfo中主要包含的就是当前View的位置状态等。然后ItemHolderInfo 就被存入mViewInfoStore中，注意这里调用的是mViewInfoStore的addTo`Pre`Layout方法，我们追进：

```
void addToPreLayout(ViewHolder holder, ItemHolderInfo info) {
    InfoRecord record = mLayoutHolderMap.get(holder);
    if (record == null) {
        record = InfoRecord.obtain();
        mLayoutHolderMap.put(holder, record);
    }
    record.preInfo = info;
    record.flags |= FLAG_PRE;
}
```
addToPreLayout方法中会根据holder来查询InfoRecord信息，如果没有，则生成，然后将info信息赋值给InfoRecord的preInfo变量。最后标记FLAG_PRE信息，如此，完成函数。

所以纵观整个layout的第一步，就是在记录当前的View信息，因为进入第二步后，View的信息就将被改变了。我们来看第二步：


#### dispatchLayoutStep2

```
private void dispatchLayoutStep2() {
    ...
    mLayout.onLayoutChildren(mRecycler, mState);
	 ...
    mState.mLayoutStep = State.STEP_ANIMATIONS;
}
```

layout的第二步主要就是真正的去布局View了，前面也说过，RecyclerView的布局是由LayoutManager负责的，所以第二步的主要工作也都在LayoutManager中，由于每种布局的方式不一样，这里我们以常见的LinearLayoutManager为例。我们看其onLayoutChildren方法：

```
public void onLayoutChildren(RecyclerView.Recycler recycler, RecyclerView.State state) {

        ...

        if (!mAnchorInfo.mValid || mPendingScrollPosition != NO_POSITION ||
                mPendingSavedState != null) {
            updateAnchorInfoForLayout(recycler, state, mAnchorInfo);
        }

        ...

        if (mAnchorInfo.mLayoutFromEnd) {
            firstLayoutDirection = mShouldReverseLayout ? LayoutState.ITEM_DIRECTION_TAIL :
                    LayoutState.ITEM_DIRECTION_HEAD;
        } else {
            firstLayoutDirection = mShouldReverseLayout ? LayoutState.ITEM_DIRECTION_HEAD :
                    LayoutState.ITEM_DIRECTION_TAIL;
        }

        ...

        onAnchorReady(recycler, state, mAnchorInfo, firstLayoutDirection);

        ...

        if (mAnchorInfo.mLayoutFromEnd) {
           ...
        } else {
            // fill towards end
            updateLayoutStateToFillEnd(mAnchorInfo);
            fill(recycler, mLayoutState, state, false);
        	  ...

            // fill towards start
            updateLayoutStateToFillStart(mAnchorInfo);
    		  ...
            fill(recycler, mLayoutState, state, false);
            ...
        }

        ...
    }
```

整个onLayoutChildren过程还是很复杂的，这里我尽量省略了一些与流程关系不大的细节处理代码。整个onLayoutChildren过程可以大致整理如下：

- 找到anchor点
- 根据anchor一直向前布局，直至填充满anchor点前面的所有区域
- 根据anchor一直向后布局，直至填充满anchor点后面的所有区域

anchor点的寻找是由updateAnchorInfoForLayout函数负责的：

```
private void updateAnchorInfoForLayout(RecyclerView.Recycler recycler, RecyclerView.State state,
                                       AnchorInfo anchorInfo) {
    ...
    if (updateAnchorFromChildren(recycler, state, anchorInfo)) {
        return;
    }
    ...
    anchorInfo.assignCoordinateFromPadding();
    anchorInfo.mPosition = mStackFromEnd ? state.getItemCount() - 1 : 0;
}
```

函数内首先通过子view来获取anchor，如果没有获取到，就根据就取头/尾点来作为anchor。所以这里我们主要关注updateAnchorFromChildren函数：

```
private boolean updateAnchorFromChildren(RecyclerView.Recycler recycler,
                                         RecyclerView.State state, AnchorInfo anchorInfo) {
    if (getChildCount() == 0) {
        return false;
    }
    final View focused = getFocusedChild();
    if (focused != null && anchorInfo.isViewValidAsAnchor(focused, state)) {
        anchorInfo.assignFromViewAndKeepVisibleRect(focused);
        return true;
    }
    if (mLastStackFromEnd != mStackFromEnd) {
        return false;
    }
    View referenceChild = anchorInfo.mLayoutFromEnd
            ? findReferenceChildClosestToEnd(recycler, state)
            : findReferenceChildClosestToStart(recycler, state);
    if (referenceChild != null) {
        anchorInfo.assignFromView(referenceChild);
        ...
        return true;
    }
    return false;
}
```

updateAnchorFromChildren内部做的事情也很容易理解，首先寻找被focus的child，找到的话以此child作为anchor，否则根据布局的方向寻找最合适的child来作为anchor，如果找到则将child的信息赋值给anchorInfo，其实anchorInfo主要记录的信息就是view的物理位置与在adapter中的位置。找到后返回true，否则返回false则交由上一步的函数做处理。

综上，刚刚的所追踪的代码都是在寻找anchor点。在我们寻找后，LinearLayoutManager还给了我们更改anchor的时机，就是`onAnchorReady `函数，我们可以继承LinearLayoutManager 来重写onAnchorReady方法，就可以实现某些特定的功能，比如进入RecyclerView时定位在某一项等等。

总之，我们现在找到了anchor信息，接下来就是根据anchor来布局了。无论从上到下还是从下到上布局，都调用的是fill方法，我们进入fill方法来查看一番：

```
int fill(RecyclerView.Recycler recycler, LayoutState layoutState,
        RecyclerView.State state, boolean stopOnFocusable) {
    final int start = layoutState.mAvailable;
    if (layoutState.mScrollingOffset != LayoutState.SCROLLING_OFFSET_NaN) {
        recycleByLayoutState(recycler, layoutState);
    }
    int remainingSpace = layoutState.mAvailable + layoutState.mExtra;
    LayoutChunkResult layoutChunkResult = mLayoutChunkResult;
    while ((layoutState.mInfinite || remainingSpace > 0) && layoutState.hasMore(state)) {
        layoutChunk(recycler, state, layoutState, layoutChunkResult);
        ...
    }
    return start - layoutState.mAvailable;
 }
```
这里同样省略了很多代码，我们关注重点：

首先比较重要的函数是recycleByLayoutState，这个函数就厉害了，它会根据当前信息对不需要的View进行回收：

```
private void recycleByLayoutState(RecyclerView.Recycler recycler, LayoutState layoutState) {
     if (layoutState.mLayoutDirection == LayoutState.LAYOUT_START) {
			...
     } else {
         recycleViewsFromStart(recycler, layoutState.mScrollingOffset);
     }
 }

```
我们继续追进recycleViewsFromStart：

```
private void recycleViewsFromStart(RecyclerView.Recycler recycler, int dt) {
    final int limit = dt;
    final int childCount = getChildCount();
    if (mShouldReverseLayout) {
        ...
    } else {
        for (int i = 0; i < childCount; i++) {
            View child = getChildAt(i);
            if (mOrientationHelper.getDecoratedEnd(child) > limit
                    || mOrientationHelper.getTransformedEndWithDecoration(child) > limit) {
                recycleChildren(recycler, 0, i);
                return;
            }
        }
    }
}
```
这个函数的作用就是遍历所有的子View，找出逃离边界的View进行回收，回收函数我们锁定在recycleChildren里，而这个函数最后又会调到removeAndRecycleViewAt：

```
public void removeAndRecycleViewAt(int index, Recycler recycler) {
    final View view = getChildAt(index);
    removeViewAt(index);
    recycler.recycleView(view);
}
```

这个函数首先调用removeViewAt函数，这个函数的作用是将View从RecyclerView中移除， 紧接着我们看到，是recycler执行了view的回收逻辑。这里我们暂且打住，关于recycler我们会单独进行说明，这里我们只需要理解，在fill函数的一开始会去回收逃离出屏幕的view。我们再次回到fill函数，关注这里：

```
 while ((layoutState.mInfinite || remainingSpace > 0) && layoutState.hasMore(state)) {
      layoutChunk(recycler, state, layoutState, layoutChunkResult);
        ...
 }
```

这段代码很容易理解，只要还有剩余空间，就会执行layoutChunk方法：

```
void layoutChunk(RecyclerView.Recycler recycler, RecyclerView.State state,
        LayoutState layoutState, LayoutChunkResult result) {
    View view = layoutState.next(recycler);
    ...
    LayoutParams params = (LayoutParams) view.getLayoutParams();
    if (layoutState.mScrapList == null) {
        if (mShouldReverseLayout == (layoutState.mLayoutDirection
                == LayoutState.LAYOUT_START)) {
            addView(view);
        } else {
            addView(view, 0);
        }
    } else {
       ...
    }
    ...
    layoutDecoratedWithMargins(view, left, top, right, bottom);
    ...
}
```
我们首先看到，layoutState的next方法返回了一个View，凭空变出一个View，好神奇，追进去看一下：

```
View next(RecyclerView.Recycler recycler) {
    ...
    final View view = recycler.getViewForPosition(mCurrentPosition);
    return view;
}
```

可见，view的获取逻辑也由recycler来负责，所以，这里我们同样打住，只需要清楚recycler可以根据位置返回一个view即可。

再回到layoutChunk看一下对刚刚生成的view作何处理：


```
if (mShouldReverseLayout == (layoutState.mLayoutDirection
            == LayoutState.LAYOUT_START)) {
        addView(view);
    } else {
        addView(view, 0);
}
```
很明显调用了addView方法，虽然这个方法是LayoutManager的，但这个方法最终会多次辗转调用到RecyclerView的addView方法，将view添加在RecyclerView中。

综上，我们就梳理了整个第二步布局的过程，此过程完成了子View的测量与布局，任务还是相当繁重。
#### dispatchLayoutStep3

接下来，就到了布局的最后一步了，我们直接看下dispatchLayoutStep3方法：

```
private void dispatchLayoutStep3() {
    mState.mLayoutStep = State.STEP_START;
    if (mState.mRunSimpleAnimations) {
        for (int i = mChildHelper.getChildCount() - 1; i >= 0; i--) {
            ...
            final ItemHolderInfo animationInfo = mItemAnimator
                    .recordPostLayoutInformation(mState, holder);
                mViewInfoStore.addToPostLayout(holder, animationInfo);
        }

        mViewInfoStore.process(mViewInfoProcessCallback);
    }
    ...
}
```

这一步是与第一步呼应的的，此时由于子View都已完成布局，所以子View的信息都发生了变化。我们会看到第一步出现的mViewInfoStore和mItemAnimator再次登场，这次mItemAnimator调用的是recordPostLayoutInformation方法，而mViewInfoStore调用的是addToPostLayout方法，还记得刚刚我强调的吗，之前是pre，也就是真正布局之前的状态，而现在要记录布局之后的状态，我们追进addToPostLayout：

```
void addToPostLayout(ViewHolder holder, ItemHolderInfo info) {
    InfoRecord record = mLayoutHolderMap.get(holder);
    if (record == null) {
        record = InfoRecord.obtain();
        mLayoutHolderMap.put(holder, record);
    }
    record.postInfo = info;
    record.flags |= FLAG_POST;
}
```

和第一步的addToPreLayout类似，不过这次info信息被赋值给了record的postInfo变量，这样，一个record中就包含了布局前后view的状态。

最后，mViewInfoStore调用了process方法，这个方法就是根据mViewInfoStore中的View信息，来执行动画逻辑，这又是一个可以展看很多的点，这里不做探讨，感兴趣的可以深入的看一下，会对动画流程有更直观的体会。


### 缓存逻辑

前面的章节对于Recycler这个类相关的操作我们都直接进行了忽略，这里我们好好的来看下RecylerView是如何工作的。

与ListView不同，RecyclerView的缓存是分为多级的，但其实整个的缓存逻辑还是很容易理解的，我们首先看一下刚刚获取View的方法getViewForPosition:

```
View getViewForPosition(int position, boolean dryRun) {
            boolean fromScrap = false;
            ViewHolder holder = null;
            if (mState.isPreLayout()) {
                holder = getChangedScrapViewForPosition(position);
                fromScrap = holder != null;
            }
            if (holder == null) {
                holder = getScrapViewForPosition(position, INVALID_TYPE, dryRun);
               ...
            }
            if (holder == null) {
                final int offsetPosition = mAdapterHelper.findPositionOffset(position);

                final int type = mAdapter.getItemViewType(offsetPosition);
                if (mAdapter.hasStableIds()) {
                    holder = getScrapViewForId(mAdapter.getItemId(offsetPosition), type, dryRun);
                }
                if (holder == null && mViewCacheExtension != null) {
                    final View view = mViewCacheExtension
                            .getViewForPositionAndType(this, position, type);
                   ...
                }
                if (holder == null) { // fallback to recycler
                    holder = getRecycledViewPool().getRecycledView(type);
                    if (holder != null) {
                        holder.resetInternal();
                        if (FORCE_INVALIDATE_DISPLAY_LIST) {
                            invalidateDisplayListInt(holder);
                        }
                    }
                }
                if (holder == null) {
                    holder = mAdapter.createViewHolder(RecyclerView.this, type);
                }
            }

           //生成LayoutParams的代码 ...
            return holder.itemView;
        }
}
```
获取View的逻辑可以整理成如下：

- 搜索mChangedScrap，如果找到则返回相应holder。
- 搜索mAttachedScrap与mCachedViews，如果找到且holder有效则返回相应holder。
- 如果设置了mViewCacheExtension，对其调用getViewForPositionAndType方法进行获取，若该方法返回结果则生成对应的holder。
- 搜索mRecyclerPool，如果找到到则返回相应holder
- 如果上述过程都没有找到对应的holder，则执行我们熟悉的Adapter.createViewHolder()，创建新的holder实例

综上，只要数据合法，该方法最终肯定会返回符合条件的View。这里可能大家比较关注的是这么多级别的缓存到底有什么区别，这个问题可能大家看完回收View的代码会有更深的理解：

```
 void recycleViewHolderInternal(ViewHolder holder) {
 ...
            if (holder.isRecyclable()) {
                if (!holder.hasAnyOfTheFlags(ViewHolder.FLAG_INVALID | ViewHolder.FLAG_REMOVED
                        | ViewHolder.FLAG_UPDATE)) {
                    int cachedViewSize = mCachedViews.size();
                    if (cachedViewSize >= mViewCacheMax && cachedViewSize > 0) {
                        recycleCachedViewAt(0);
                        cachedViewSize --;
                    }
                    if (cachedViewSize < mViewCacheMax) {
                        mCachedViews.add(holder);
                        cached = true;
                    }
                }
                if (!cached) {
                    addViewHolderToRecycledViewPool(holder);
                    recycled = true;
                }
            }

 ...
}
```

View的回收并不像View的创建那么复杂，这里只涉及了两层缓存mCachedViews与mRecyclerPool，mCachedViews相当于一个先进先出的数据结构，每当有新的View需要缓存时都会将新的View存入mCachedViews，而mCachedViews则会移除头元素，并将头元素放入mRecyclerPool，所以mCachedViews相当于一级缓存，mRecyclerPool则相当于二级缓存，并且mRecyclerPool是可以多个RecyclerView共享的，这在类似于多Tab的新闻类应用会有很大的用处，因为多个Tab下的多个RecyclerView可以共用一个二级缓存。减少内存开销。

如此，就是对RecyclerView的缓存逻辑的简要分析。

## 与AdapterView比较

谈到RecyclerView，总避免不了与它的前辈AdapterView家族进行一撕，这里我整理了一下RecylerView与AdapterView的各自特点：

![](/assets/images/understand-recycler/o_1av9k9g98p5d1s6u14n91m3n14ue9.png)

前面四点两位都提供了各自的实现，但也各有各自的特点：

- 点击事件

     ListView原生提供Item单击、长按的事件, 而RecyclerView则需要使用onTouchListener，相对自己实现会比较复杂。

- 分割线

   ListView可以很轻松的设置divider属性来显示Item之间的分割线，RecyclerView需要我们自己实现ItemDecoration，前者使用简单，后者可定制性强。

- 布局类型

  AdapterView提供了ListView与GridView两种类型，分别对应流式布局与网格式布局。RecyclerView提供了LinearLayoutManager、GridLayoutManager与之抗衡，相对而言，使用RecyclerView来进行更换布局方式更为轻松。只需要更换一个变量即可，而对于AdapterView而言则是需要更换一个View了。

- 缓存方式

    ListView使用了一个名为RecyclerBin的类负责试图的缓存，而Recycler则使用Recycler来进行缓存，原理上两者基本一致。

在对比了几个相近点之外，我们再分别来看一下两者的不同点，先看RecyclerView：

- 局部刷新

    这是一个很有用的功能，在ListView中我们想局部刷新某个Item需要自己来编写刷新逻辑，而在RecyclerView中我们可以通过`notifyItemChanged(position)`来刷新单个Item，甚至可以通过`notifyItemChanged(position, payload)`来传入一个payload信息来刷新单个Item中的特定内容。

- 动画

    作为视觉动物，我相信很多人喜欢RecylerView都和它简单的动画API有关，因为之前对ListView做动画比较困难，并且不舒服。

- 嵌套布局

   嵌套布局也是最近比较火的一个概念，RecyclerView实现了`NestedScrollingChild`接口，使得它可以和一些嵌套组件很好的工作。

我们再来看ListView原生独有的几个特点：

- 头部与尾部的支持

   ListView原生支持添加头部与尾部，虽然RecyclerView可以通过定义不同的Type来做支持，但实际应用中，如果封装的不好，是很容易出问题的，因为Adapter中的数据位置与物理数据位置发生了偏移。

- 多选

  支持多选、单选也是ListView的一大长处，其实如果要我们自己在RecyclerView中去做支持还是需要不少代码量的。

- 多数据源的支持

   ListView提供了CursorAdapter、ArrayAdapter，可以让我们很方便的从数据库或者数组中获取数据，这在测试的时候很有用。

综上，我们会发现RecycerView的最大特点就是灵活，正因为这种灵活，因此会牺牲了某些便利性。而AdapterView相对来讲就比较刻板，但它原生为我们提供了很多有用的方法来便于我们快速开发。ListView并不像当年的ActivityGroup，在Fragment出来后就被标记为Deprecated。两者目前还是一种互补的关系，起码在短时间内RecyclerView还并不能完全替代AdapterView，个人感觉原因有两个，一是目前太多的应用使用了ListView，并且ListView向RecyclerView转变也没有无损的方法。第二点，比如我就是想添加个头部，每个item带个点击事件这类简单的需求，ListView完全可以很轻松的胜任，没必要舍近求远来使用RecyclerView。因此，在实际应用中选择更适合自己的就好。

当然，从Google最近几次的更新来看，RecyclerView的进化还是很迅速的，而ListView则几乎没什么变动，所以RecyclerView绝对是大大的潜力股呀。



## 设计精巧的类

> 在翻看RecycleView源码的过程中也遇见了许多之前没有注意过的类，这些类都可以复用在我们的日常工作当中。这里列举出其中具有代表性的几位。

### Bucket

如果一个对象有大量的是与非的状态需要表示，通常我们会使用BitMask
技术来节省内存，在 Java 中，一个 byte 类型，有 8 位（bit），可以表达 8 个不同的状态，而 int 类型，则有 32 位，可以表达 32 种状态。再比如Long类型，有64位，则可以表达64中状态。一般情况下使用一个Long已经足够我们使用了。但如果有不设上限的状态需要我们表示呢？

在ChildHelper里有一个静态内部类Bucket，基本源码如下：

```
    static class Bucket {

        final static int BITS_PER_WORD = Long.SIZE;

        final static long LAST_BIT = 1L << (Long.SIZE - 1);

        long mData = 0;

        Bucket next;

        void set(int index) {
            if (index >= BITS_PER_WORD) {
                ensureNext();
                next.set(index - BITS_PER_WORD);
            } else {
                mData |= 1L << index;
            }
        }
      ...
    }
```
可以看到，Bucket是一个链表结构，当index大于64的时候，它便会去下一个Bucket去寻找，所以，Bucket可以不设上限的表示状态。


### Pools

熟悉Message回收机制的朋友可能了解，在使用Message对象时最好通过`Message.obtain()`方法来获取，这样可以在很多情况下避免创建新对象。在使用完之后调用`message.recycle()`来回收消息。

谷歌为这种机制也提供了抽象的实现,就是位于v4包下Pools类, 内部接口Pool提供了`acquire`与`release`两个方法,不过需要注意的是这个`acquire`方法可能返回空,毕竟Pools不是业务类,它不应该清楚对象的具体创建逻辑.

还有一点是Pools与Message类的实现机制不同,每个Message对象内部都持有一个引用下一个message的指针,相当于一个链表结构,而Pool的实现类`SimplePool`中使用的是数组.

Pool机制在 RecycleView 中有如下几处应用：

* RecycleView将item的增删改封装为`UpdateOp`类。

* `ViewInfoStore`类中静态内部类`InfoRecord`。

## 缺点

RecyclerView也不是万能的，它的灵活性也是有一定限制的，比如我就遇到了一不是很好解决的问题：


 Recyler的缓存级别是一个Item的整个View，而我们没办法自定义缓存级别，这样说比较抽象，举个例子，我的某些Item的某个子View加载很耗时，所以我希望我在上下滑动的时候，Item的其它View是可以被回收利用的，但这个加载很耗时的View是不要重复使用的。即我希望用空间换取时间来获取滑动的流畅性。当然，这样的需求不常见，RecyclerView也不能很好的满足这一点。


## 总结

RecyclerView也应该算作一个明星控件了，自从其诞生开始就备受欢迎，仔细的学习也能让我们在工作中更容易的、更恰当的使用。本文也只是分析了RecyclerView的一部分，关于动画、滑动、嵌套滑动等等还需要大家自行去研究。

![](/assets/images/understand-recycler/o_1avel5oho3vhmvmtg1o0v1e8a9.gif)

## 参考
[A First Glance at Android’s RecyclerView](http://www.grokkingandroid.com/first-glance-androids-recyclerview/){:rel="nofollow"}

[Using the RecyclerView](https://guides.codepath.com/android/using-the-recyclerview#create-the-recyclerview-within-layout){:rel="nofollow"}
