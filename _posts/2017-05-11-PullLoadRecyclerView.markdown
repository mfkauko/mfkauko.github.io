---
layout: post
title:  "PullLoadRecyclerView"
date:   2017-05-11 17:23:23 +0800
categories: Android
tag: Android开发
---

本文主要是实现下拉刷新和上拉加载更多。项目中本来使用了PullableListView实现了类似的功能，但是RecyclerView一直还没有被运用到项目中，同时为了更加熟悉RecyclerView相关知识，所以就用RecyclerView也实现了类似功能。  
## NestedScrolling相关知识
NestedScrolling提供了一套父View和子View滑动交互机制。要完成这样的交互，父View需要实现NestedScrollingParent接口，子View需要实现NestedScrollingChild接口。  

### 实现NestedScrollingChild
如果有一个可以滑动的View，需要被用来作为嵌入滑动的子View，就必须实现本接口。在此View中，包含一个NestedScrollingChildHelper辅助类。NestedScrollingChild接口的实现，基本上就是调用Helper类的对应的函数即可。因为Helper类中已经实现好了child和parent之间的交互逻辑。原来的View的处理Touch事件，并实现滑动的逻辑大体上不需要改变。  
需要做的是，如果要准备开始滑动了，需要告诉parent，你要准备进入滑动状态了，调用startNestedScroll。在滑动之前，先问一下你的parent是否需要滑动，也就是调用dispatchNestedPreScroll。如果父类滑动了一定距离，你需要重新计算一下父类滑动后剩下给你的滑动距离余量。然后，你自己进行余下的滑动。最后，如果滑动距离还有剩余，你就再问一下，parent是否需要再继续滑动你剩下的距离，也就是调用dispatchNestedScroll。

### 实现NestedScrollParent
同样也有一个NestedScrollParentHelper辅助类来实现与child交互的逻辑。滑动动作是child主动发起，parent收到滑动回调并作出响应。  
从上面的child分析可知，滑动开始调用startNestedScroll，parent收到onStartNestedScroll回调，决定是否需要配合child一起进行处理滑动，如果需要配合，还会回调onNestedScrollAccepted。  
每次滑动前，child先询问parent是否需要滑动，即dispatchNestedPreScroll，这就回调到parent的onNestedPreScroll，parent可以在这个回调中“劫持”掉child的滑动，也就是先于child滑动。  
child滑动以后，会调用onNestedScroll。回调到parent的onNestedScroll，这里就是child滑动后，剩下的给parent处理，也就是后于child滑动。
最后滑动结束，调用onStopNestedScroll表示本次处理结束。

除了上面关于Scroll相关的调用和回调，还有Fling相关的调用和回调，处理逻辑基本一致。


### 方法介绍
- onStartNestedScroll  
一定要按照自己的需求返回true，该方法决定了当前控件是否能接收到其内部View（并非是直接子View）滑动时的参数；假设你只涉及到纵向滑动，可以根据nestedScrollAxes这个参数来进行纵向判断。
- onNestedPreScroll  
会传入内部View移动的dx，dy。如果你消耗一定的dx，dy，就通过最后一个参数consumed进行指定。比如我要消耗一半的dy，就可以写consumed[1]=dy/2
- onNestedFling  
可以捕获对内部view的fling事件，如果return true则表示拦截掉内部view的事件。
- onNestedScrollView  
内部view滑动结束后会回调这个方法，根据dxUnconsumed或者dyUnconsumed，父view需要进行处理。
- onStopNestedScroll  
内部view滑动结束后进行的处理。

## 实现
首先定义一个实现NestedScrollingParent的LinearLayout--PullLoadRecyclerViewLayout，因为RecyclerView本身实现了NestedScrolllingChild，关于滑动的处理我们可以直接在PullLoadRecyclerViewLayout中进行。  
Tips：这里假设都已经了解了NestedScrolling的相关知识，如果不熟悉，可以直接参考[hongyang大神的这篇文章](http://blog.csdn.net/lmj623565791/article/details/52204039)。  

所以这里我们只需要处理onNestedPreScroll、onNestedScrollView以及onStopNestedScroll等几个方法。 

- onNestedPreScroll  
 这里的逻辑主要是子View进行处理，根据当前滑动的位置等消耗滑动距离dy。然后将未处理的剩下的dy传递给consumed[1]。  
判断的逻辑是：当RecyclerView已经滑动到最顶端的时候，再继续往下拉，header就需要出现。当RecyclerView已经滑动到最底端的时候，再继续往上拉，footer就需要出现。这两种情况下，comsumed[1]=dy,表示parent直接消耗掉所有的滑动距离。但是，这里需要考虑到一些特殊情况：  
	- 当header已经出现的时候，手指上滑。如果header依然可见，这个时候需要parent处理，并且consumed[1]=dy。如果继续上滑header不见了，这个时候，parent就不需要处理，直接交由子view自己处理，但是此时，comsumed[1]只是之前header显示的时候总距离，并不是dy。
	- 当footer已经出现的时候，手指下滑。如果footer依然可见，这个时候需要parent处理，并且consumed[1]=dy。如果继续下滑footer不见了，这个时候，parent就不需要处理，直接交由子view自己处理，但是此时，consumed[1]只是之前footer显示的时候总距离，并不是dy。  
	具体逻辑请参考代码，代码里面的注释也比较详细，就是这一块的逻辑比较复杂，因为要考虑到上面的两种特殊情况。
- onNestedScroll  
 这个地方，我们还要根据dyUnconsumed != 0来对RecyclerView进行处理。如果parent并没有消耗完所有的滑动距离，那么RecyclerView需要滑动相应的距离。
- onStopNestedScroll  
RecyclerView滑动结束之后的回调，在这里我们可以写刷新或者加载的逻辑，具体参考代码。

## 注意事项
- 在Activity中添加如下代码即可直接使用
``` 
mLayout.addHeaderView(mHeaderView, DisplayUtil.dpToPx(MainActivity.this, 60));  
mLayout.addFooterView(mFooterView, DisplayUtil.dpToPx(MainActivity.this, 40));  
mLayout.setMyRecyclerView(new LinearLayoutManager(MainActivity.this, LinearLayoutManager.VERTICAL, false),
        mAdapter, true);  
mLayout.addOnTouchUpListener(this);
```
- 下拉刷新和上拉加载的逻辑处理，只需要重写方法
```
void onDataRefreshing();  
void onDataLoadingMore();
```
- 下拉刷新和上拉加载完成后，记得调用
```
void onRefreshFinish(boolean);  
void onLoadMoreFinish(boolean);
```  
这里面会有状态的改变、动画的停止以及界面的更新。

## github链接
[欢迎访问](https://github.com/mfkauko/PullLoadRecyclerView/)  
您的star是我前进的动力，欢迎指正其中的问题，大家共同进步。