---
layout: post
title:  "StickyHeaderRecyclerView源码分析"
date:   2017-05-16 10:52:30 +0800
categories: Android
tag: 源码分析
---
本文对项目中使用的StickHeaderRecyclerView进行分析，不能只使用而不知其中的原理。  

[StickHeaderRecyclerView的github地址](https://github.com/timehop/sticky-headers-recyclerview)

### 基本原理
首先我们是这样使用的：
```
mRecyclerView.addItemDecoration(new StickyRecyclerHeadersDecoration(mAdapter));
```
平时我们在为RecyclerView的item添加分割线的时候也是这样实现的，所以就是通过ItemDecoration来对RecyclerView的每个item进行添加header。  

重写ItemDecoration，我们一般都只需要关注这几个函数：

- getItemOffsets， 为outRect设置4个方向的值，而这些值会被计算进所有的decoration的尺寸中，最后这个尺寸会被计入到RecyclerView的每个item的padding中

- onDraw 为divider设置绘制范围并绘制到canvas上，并且这个范围是可以超出在getItemOffsets中设置的范围的，但是由于这个方法绘制的内容是在child view的下面，所以是看不到的，会存在overDraw

- onDrawOver 绘制在最上层

decoration的onDraw----->child view的onDraw------>decoration的onDrawOver是依次发生的。

### 源码分析
我们来看StickyRecyclerHeadersDecoration中这些方法是如何实现的。

首先，构造函数，主要是生成了很多需要用到的对象，比如HeaderProvider、OrientationProvider、HeaderPositionCalculator、HeaderRender、DimensionCalculator，这些都用于在item进行滑动时的一些判断，暂且先不管。

其次，来看getItemOffsets函数：
```
int itemPosition = parent.getChildAdapterPosition(view);
if (itemPosition == RecyclerView.NO_POSITION) {
    return;
}
if (mHeaderPositionCalculator.hasNewHeader(itemPosition)) {
    View header = getHeaderView(parent, itemPosition);
    setItemOffsetsForHeader(outRect, header, mOrientationProvider.getOrientation(parent));
}
```
```
private void setItemOffsetsForHeader(Rect itemOffsets, View header, int orientation) {
   Rect headerMargins = mDimensionCalculator.getMargins(header);
   if (orientation == LinearLayoutManager.VERTICAL) {
       itemOffsets.top = header.getHeight() + headerMargins.top + headerMargins.bottom;
   } else {
       itemOffsets.left = header.getWidth() + headerMargins.left + headerMargins.right;
   }
}
```
所以，逻辑是：如果当前position对应的是一个新的header，我们就要给outRect设置值，方便后面的绘制。
我们再来看看函数hasNewHeader：
```
public boolean hasNewHeader(int position) {
    if (indexOutOfBounds(position)) {
   	 	return false;
    }

    long headerId = mAdapter.getHeaderId(position);
    if (headerId < 0) {
        return false;
    }

    return position == 0 || headerId != mAdapter.getHeaderId(position - 1);
}
```
如果当前position的headerId和(position -1)的headerId不一样，代表从这个item开始，就有新的header了。

我们再来看onDrawOver函数，在滑动的时候，我们可以观察到header是绘制到child view的上面的，所以这里只重写了onDrawOver函数，并没有处理onDraw函数。
```
@Override
public void onDrawOver(Canvas canvas, RecyclerView parent, RecyclerView.State state) {
    super.onDrawOver(canvas, parent, state);
    mHeaderRects.clear();

    if (parent.getChildCount() <= 0 || mAdapter.getItemCount() <= 0) {
        return;
    }

    for (int i = 0; i < parent.getChildCount(); i++) {
        View itemView = parent.getChildAt(i);
        int position = parent.getChildAdapterPosition(itemView);
        if (position == RecyclerView.NO_POSITION) {
            continue;
        }
        if (hasStickyHeader(i, position) || mHeaderPositionCalculator.hasNewHeader(position)) {
            View header = mHeaderProvider.getHeader(parent, position);
            Rect headerOffset = mHeaderPositionCalculator.getHeaderBounds(parent, header,
                    itemView, hasStickyHeader(i, position));
            mRenderer.drawHeader(parent, canvas, header, headerOffset);
            mHeaderRects.put(position, headerOffset);
        }
    }
}
```
```
private boolean hasStickyHeader(int listChildPosition, int indexInList) {
    if (listChildPosition > 0 || mAdapter.getHeaderId(indexInList) < 0) {
        return false;
    }

    return true;
}
```

```
public Rect getHeaderBounds(RecyclerView recyclerView, View header, View firstView, boolean firstHeader) {
    int orientation = mOrientationProvider.getOrientation(recyclerView);
    Rect bounds = getDefaultHeaderOffset(recyclerView, header, firstView, orientation);

    if (firstHeader && isStickyHeaderBeingPushedOffscreen(recyclerView, header)) {
        View viewAfterNextHeader = getFirstViewUnobscuredByHeader(recyclerView, header);
        int firstViewUnderHeaderPosition = recyclerView.getChildAdapterPosition(viewAfterNextHeader);
        View secondHeader = mHeaderProvider.getHeader(recyclerView, firstViewUnderHeaderPosition);
        translateHeaderWithNextHeader(recyclerView, mOrientationProvider.getOrientation(recyclerView), bounds,
                header, viewAfterNextHeader, secondHeader);
    }

    return bounds;
}
```
我们看onDrawOver的循环里面，如果hasStickyHeader返回true或者hasNewHeader返回true，就根据position取出对应的header，并通过canvas画出来。

注意到这里的判断，如果listChildPostion > 0，就需要判断mAdapter.getHeaderId(indexInList) < 0是否为true。对于项目中的所有indexInList，mAdapter.getHeaderId(indxeInList)均为true。所以hasStickyHeader的返回值主要取决于listChildPosition是否>0。

最开始布局展现的时候，同一个header下的所有child，除了listChildPositon == 0的情况，hasStickyHeader和hasNewHeader均返回false。所以只画了一个header。如果有另一个header出现，hasNewHeader会返回true，同样也会画另一个header。在getHeaderBounds方法里面，会有关于header的offset的计算。

我们再来看getHeaderBounds方法里面，有一个isStickHeaderBeingPushedOffscreen的判断。

```
private boolean isStickyHeaderBeingPushedOffscreen(RecyclerView recyclerView, View stickyHeader) {
    View viewAfterHeader = getFirstViewUnobscuredByHeader(recyclerView, stickyHeader);
    int firstViewUnderHeaderPosition = recyclerView.getChildAdapterPosition(viewAfterHeader);
    if (firstViewUnderHeaderPosition == RecyclerView.NO_POSITION) {
        return false;
    }

    if (firstViewUnderHeaderPosition > 0 && hasNewHeader(firstViewUnderHeaderPosition)) {
        View nextHeader = mHeaderProvider.getHeader(recyclerView, firstViewUnderHeaderPosition);
        Rect nextHeaderMargins = mDimensionCalculator.getMargins(nextHeader);
        Rect headerMargins = mDimensionCalculator.getMargins(stickyHeader);

        if (mOrientationProvider.getOrientation(recyclerView) == LinearLayoutManager.VERTICAL) {
            int topOfNextHeader = viewAfterHeader.getTop() - nextHeaderMargins.bottom - nextHeader.getHeight() - nextHeaderMargins.top;
            int bottomOfThisHeader = recyclerView.getPaddingTop() + stickyHeader.getBottom() + headerMargins.top + headerMargins.bottom;
            if (topOfNextHeader < bottomOfThisHeader) {
                return true;
            }
        } else {
            int leftOfNextHeader = viewAfterHeader.getLeft() - nextHeaderMargins.right - nextHeader.getWidth() - nextHeaderMargins.left;
            int rightOfThisHeader = recyclerView.getPaddingLeft() + stickyHeader.getRight() + headerMargins.left + headerMargins.right;
            if (leftOfNextHeader < rightOfThisHeader) {
                return true;
            }
        }
    }

    return false;
}
```
getFirstViewUnobscuredByHeader(RecyclerView recyclerview, View header)， 返回的是没有被header遮挡的第一个child view。在最开始的时候，这个child view就是position为0的view。所以isStickyHeaderBeingPushedOffscreen会返回false，也就没有后续流程了。

当滑动屏幕（recyclerview滑动）的时候，StickyRecyclerHeadersDecoration的onDrawOver函数会不停的被调用。所以会不停的循环，执行上面的流程。当我们滑动到两个header已经挨在一起的时候，比如类似下面图中的这种情况：
![两个header贴在一起的情况](/images/posts/sticky/header.png)

如果继续滑动，isStickyHeaderBeingPushedOffscreen会返回true，这个时候会去调用tranlateHeaderWithNextHeader方法。

```
private void translateHeaderWithNextHeader(RecyclerView recyclerView, int orientation, Rect translation,
                                               View currentHeader, View viewAfterNextHeader, View nextHeader) {
    Rect nextHeaderMargins = mDimensionCalculator.getMargins(nextHeader);
    Rect stickyHeaderMargins = mDimensionCalculator.getMargins(currentHeader);
    if (orientation == LinearLayoutManager.VERTICAL) {
        int topOfStickyHeader = getListTop(recyclerView) + stickyHeaderMargins.top + stickyHeaderMargins.bottom;
        int shiftFromNextHeader = viewAfterNextHeader.getTop() - nextHeader.getHeight() - nextHeaderMargins.bottom - nextHeaderMargins.top - currentHeader.getHeight() - topOfStickyHeader;
        if (shiftFromNextHeader < topOfStickyHeader) {
            translation.top += shiftFromNextHeader;
        }
    } else {
        int leftOfStickyHeader = getListLeft(recyclerView) + stickyHeaderMargins.left + stickyHeaderMargins.right;
        int shiftFromNextHeader = viewAfterNextHeader.getLeft() - nextHeader.getWidth() - nextHeaderMargins.right - nextHeaderMargins.left - currentHeader.getWidth() - leftOfStickyHeader;
        if (shiftFromNextHeader < leftOfStickyHeader) {
            translation.left += shiftFromNextHeader;
        }
    }
}
```

这个地方会将上一个header往上顶，offset的值会传递到getHeaderBounds的bounds这个Rect中，接下来会执行drawHeader函数，所以我们看到了上一个header被顶上去的效果。而下面的header会通过getDefaultHeaderOffset来获取到正确的offset值，同样的通过drawHeader绘制到正确的位置。因此我们看到的效果是，上一个header被下面一个header顶上去，直到上一个header完全消失。

上一个header完全消失的时候，上面的流程会继续执行，只是各种条件判断值不一致，导致看起来的效果是当前这个header固定在顶部。


这里，我们把整个stickyHeader的流程分析完了，当然这里只是简单的根据代码流程走了一遍，里面有些逻辑还是比较绕。但是通过这个流程，对整个流程有了更加清楚的认识，如果以后需要自己编写或者根据需求进行改动也比较方便了。

