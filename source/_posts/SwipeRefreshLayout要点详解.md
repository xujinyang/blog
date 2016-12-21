---
title: SwipeRefreshLayout要点详解
date: 2016-06-19 14:02:14
tags: Android
---

> SwipRefreshLayout是google提供的support v4包下面的下拉刷新控件，他继承自ViewGroup,内部可以放几乎所有的滚动控件。This layout should be made the parent of the view that will be refreshed as a result of the gesture and can only support one direct child.

本文不涉及到具体的使用，因为这个控件已经烂大街了，在很多标榜material design设计的app中都标配这个活泼的小球，在这样的情况下，出现了向美团，京东等，下拉出现更有趣动画的实现，比如

![美团的下拉效果](https://camo.githubusercontent.com/568acac94970a1b71140832d377a3dd4912ebf9c/687474703a2f2f696d672e626c6f672e6373646e2e6e65742f3230313531303330323234313334353736)

recruit-lifestyle/WaveSwipeRefreshLayout水滴下拉刷新...

![](https://github.com/recruit-lifestyle/WaveSwipeRefreshLayout/raw/master/sc/animation.gif)

WaveRefreshForAndroid这个是基于Android-PullToRefresh修改的而成的水波纹下拉刷新...可能作者主攻ios,所以ios的效果看起来好看点WaveRefresh...
![](https://github.com/alienjun/WaveRefreshForAndroid/raw/master/Sceenshots/screenshot2.gif)

我们会发现，他们好像各有各的炫酷狂拽吊炸天，但是有好像和亲兄弟一样，都是通过下拉动作，触发一系列的动画和动作。
下面就以他们的爹爹SwipRefreshLayout来分析他们是如何实现的，了解了原来，想在SwipRefreshLayout上定制一个自己的下拉库也就易如反掌了。

## SwipRefreshLayout绘制

再炫的控件也是要继承基础的View，既然是继承ViewGroup，那么他肯定要实现下面俩个方法
- onMeasure
- onLayout

找到对应的代码

### onMeasure
其实SwipRefreshLayout中只有来个子View，一个是类似listview，还有个是它自己添加的circleView，在onMeasure中计算childView的测量值以及模式，以及设置自己的宽和高,

```
 @Override
    public void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        super.onMeasure(widthMeasureSpec, heightMeasureSpec);
      	//判断内部控件是否是空，如果是空,就去找到，这里其实就是找mTarget=listview
        if (mTarget == null) {
            ensureTarget();
        }
        if (mTarget == null) {
            return;
        }
      //测量listview大小,去掉padding
        mTarget.measure(MeasureSpec.makeMeasureSpec(
                getMeasuredWidth() - getPaddingLeft() - getPaddingRight(),
                MeasureSpec.EXACTLY), MeasureSpec.makeMeasureSpec(
                getMeasuredHeight() - getPaddingTop() - getPaddingBottom(), MeasureSpec.EXACTLY));
         //测量小圆球大小,绝对大小
       //mCircleHeight = mCircleWidth = (int) (CIRCLE_DIAMETER_LARGE * metrics.density);
     	//mCircleHeight = mCircleWidth = (int) (CIRCLE_DIAMETER * metrics.density);
        mCircleView.measure(MeasureSpec.makeMeasureSpec(mCircleWidth, MeasureSpec.EXACTLY),
                MeasureSpec.makeMeasureSpec(mCircleHeight, MeasureSpec.EXACTLY));
        //mCurrentTargetOffsetTop 小圆球的上边Y轴坐标
        if (!mUsingCustomStart && !mOriginalOffsetCalculated) {
            mOriginalOffsetCalculated = true;
            mCurrentTargetOffsetTop = mOriginalOffsetTop = -mCircleView.getMeasuredHeight();
        }
        mCircleViewIndex = -1;
        // Get the index of the circleview.
        for (int index = 0; index < getChildCount(); index++) {
            if (getChildAt(index) == mCircleView) {
                mCircleViewIndex = index;
                break;
            }
        }
    }

```
### onLayout
在onLayout中对子view固定位置，忽然想到，下拉的时候也就是位置在改变，所以每次重新绘制的时候，onlayout中肯定有一个变量，在决定着下拉的高度，仔细看下，果然有个变量mCurrentTargetOffsetTop，如果我们找到了这个变量的变化的触发方法，也就是找到了下拉刷新核心秘密。
```
    @Override
    protected void onLayout(boolean changed, int left, int top, int right, int bottom) {
  		//获取自己的宽高
        final int width = getMeasuredWidth();
        final int height = getMeasuredHeight();
        if (getChildCount() == 0) {
            return;
        }
        //同样的代码，确定内部的view
        if (mTarget == null) {
            ensureTarget();
        }
        if (mTarget == null) {
            return;
        }
      	//这里感觉有点多余，childRight不就是width-getPaddingRight()么，老外的想法也真奇怪
        final View child = mTarget;
        final int childLeft = getPaddingLeft();
        final int childTop = getPaddingTop();
        final int childWidth = width - getPaddingLeft() - getPaddingRight();
        final int childHeight = height - getPaddingTop() - getPaddingBottom();
        child.layout(childLeft, childTop, childLeft + childWidth, childTop + childHeight);
        int circleWidth = mCircleView.getMeasuredWidth();
        int circleHeight = mCircleView.getMeasuredHeight();
      	//关键的一个参数，mCurrentTargetOffsetTop决定着小圆球下拉过程中的高度
        mCircleView.layout((width / 2 - circleWidth / 2), mCurrentTargetOffsetTop,
                (width / 2 + circleWidth / 2), mCurrentTargetOffsetTop + circleHeight);
    }

```

盗图来显示
![](https://camo.githubusercontent.com/4a98c58e7a20a4103eae24d67027472816be1e4b/68747470733a2f2f646e2d636f64696e672d6e65742d70726f64756374696f6e2d70702e71626f782e6d652f38646636643435382d373030622d346563352d623733312d6336623863333463646464632e706e67)

## View的事件分发

View的绘制过程中俩个最重要的方法找到了，那么下面就是下拉过程中的事件分发了。先介绍一下事件分发最关键的几个点
- dispatchTouchEvent(MotionEvent ev)
- onInterceptTouchEvent(MotionEvent ev)
- onTouchEvent(MotionEvent ev)
这三者的关系我之前一直搞不清，最后在任玉刚的《Android开发艺术探索中》在看到下面这段代码才算明白，这是一段伟大的代码,简单几行，事件传递的奥义表达的淋漓尽致 。

```
public boolean dispatchTouchEvent(MotionEvent ev){
	boolean consume=false;
        if(onInterceptTouchEvent(ev)){
            consume=onTouchEvent(ev);
        }else{
            consume=child.dispatchTouchEvnet(ev);
        }
  return consume;
}
```
一个滑动事件传过来，首先SwipRefreshLayout在onInterceptTouchEvent中决定要不要拦截当前事件，如果不拦截就分发给listview，如果拦截那么它的onTouchEvent会处理对应的事件.

### onInterceptTouchEvent


```
 public boolean onInterceptTouchEvent(MotionEvent ev) {
        ensureTarget();
        final int action = MotionEventCompat.getActionMasked(ev);
     	....
    	一些边际情况判断
    	....	

        switch (action) {
            case MotionEvent.ACTION_DOWN:
                setTargetOffsetTopAndBottom(mOriginalOffsetTop - mCircleView.getTop(), true);
              	//多指触摸的时候，获取第一个
                mActivePointerId = MotionEventCompat.getPointerId(ev, 0);
                mIsBeingDragged = false;
                final float initialDownY = getMotionEventY(ev, mActivePointerId);
                if (initialDownY == -1) {
                    return false;
                }
				//记录下按下的位置，老套路了
                mInitialDownY = initialDownY;
                break;

            case MotionEvent.ACTION_MOVE:
                final float y = getMotionEventY(ev, mActivePointerId);
                if (y == -1) {
                    return false;
                }
				//最后的位置，和按下的位置获取滑动距离yDiff
                final float yDiff = y - mInitialDownY;
				//滑动距离大于最小滑动距离时候拦截这个事件
                if (yDiff > mTouchSlop && !mIsBeingDragged) {
                    mInitialMotionY = mInitialDownY + mTouchSlop;
                    mIsBeingDragged = true;
                    mProgress.setAlpha(STARTING_PROGRESS_ALPHA);
                }
                break;

        }

        return mIsBeingDragged;
    }

```
### onTouchEvent

```
 public boolean onTouchEvent(MotionEvent ev) {
      
		。。。
        各种情况判断
        。。。
        switch (action) {
            case MotionEvent.ACTION_MOVE: {
                final float y = MotionEventCompat.getY(ev, pointerIndex);
              	//关键的来了，move事件传递到这里，当前Y-初始位置，再乘以阻尼系数.5f，得到一个距离overscrollTop，传到了moveSpinner中，那moveSpinner(overscrollTop)肯定就是触发滑动的关键方法了
                final float overscrollTop = (y - mInitialMotionY) * DRAG_RATE;
                if (mIsBeingDragged) {
                    if (overscrollTop > 0) {
                        moveSpinner(overscrollTop);
                    } else {
                        return false;
                    }
                }
                break;
            }
            case MotionEvent.ACTION_UP: {
                pointerIndex = MotionEventCompat.findPointerIndex(ev, mActivePointerId);
                if (pointerIndex < 0) {
                    Log.e(LOG_TAG, "Got ACTION_UP event but don't have an active pointer id.");
                    return false;
                }

                final float y = MotionEventCompat.getY(ev, pointerIndex);
                final float overscrollTop = (y - mInitialMotionY) * DRAG_RATE;
                mIsBeingDragged = false;
              	//手指抬起来的时候，在finishSpinner中判断要触发刷新onRefresh还是显示个动画就弹回去
                finishSpinner(overscrollTop);
                mActivePointerId = INVALID_POINTER;
                return false;
            }
        }

        return true;
    }

```

### moveSpinner
这个方法，在各种计算之后，设置mCircleView的scale和alpha，然后又设置了圆球中间的mProgress的角度，并没有更新mCurrentTargetOffsetTop ,最后调用了setTargetOffsetTopAndBottom方法，接着看
```
   private void moveSpinner(float overscrollTop) {
       
        // where 1.0f is a full circle
        if (mCircleView.getVisibility() != View.VISIBLE) {
            mCircleView.setVisibility(View.VISIBLE);
        }
        if (!mScale) {
            ViewCompat.setScaleX(mCircleView, 1f);
            ViewCompat.setScaleY(mCircleView, 1f);
        }
       
        float strokeStart = adjustedPercent * .8f;
        mProgress.setStartEndTrim(0f, Math.min(MAX_PROGRESS_ANGLE, strokeStart));
        mProgress.setArrowScale(Math.min(1f, adjustedPercent));

        float rotation = (-0.25f + .4f * adjustedPercent + tensionPercent * 2) * .5f;
        mProgress.setProgressRotation(rotation);
        setTargetOffsetTopAndBottom(targetY - mCurrentTargetOffsetTop, true /* requires update */);
    }
```

### setTargetOffsetTopAndBottom
将mCircleView 显示出来，设置offset，也就是会触发mCircleView的ondraw，然后mCurrentTargetOffsetTop变量再次被赋值，如果api<11的时候手动触发刷新,这样下一次SwipRefreshLayout执行 onMeasure和onLayout的时候，就知道circleview在哪里，多大。

```
 private void setTargetOffsetTopAndBottom(int offset, boolean requiresUpdate) {
        mCircleView.bringToFront();
        mCircleView.offsetTopAndBottom(offset);
        mCurrentTargetOffsetTop = mCircleView.getTop();
        if (requiresUpdate && android.os.Build.VERSION.SDK_INT < 11) {
            invalidate();
        }
    }


```

MotionEvent重复下去，mCurrentTargetOffsetTop不断更新位置，SwipRefreshLayout不断的draw，小圆球跟着手指移动的动画就完成了。

那么如果要自己定制这样的动画，怎么做？

首先流程不变，circleView要换成自己要的View，moveSpinner方法要大改，子view根据overscrollTop，计算出百分百，阻尼，进度来等等数据一步一步的设置，ACTION_UP的时候，还有改造finishSpinner设置手指抬起的时候，View的显示逻辑.这样简单的定制就完成了。





