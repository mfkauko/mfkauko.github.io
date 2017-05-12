---
layout: post
title:  "PullLoadRecyclerView"
date:   2017-05-11 17:23:23 +0800
categories: Android
---
### 使用方法：

- Activity中添加以下代码就可以直接使用：
mLayout.addHeaderView(mHeaderView, DisplayUtil.dpToPx(MainActivity.this, 60));
mLayout.addFooterView(mFooterView, DisplayUtil.dpToPx(MainActivity.this, 40));
mLayout.setMyRecyclerView(new LinearLayoutManager(MainActivity.this, LinearLayoutManager.VERTICAL, false),
        mAdapter, true);
mLayout.addOnTouchUpListener(this);

- 下拉刷新和上拉加载的处理，只需要重写方法：
void onDataRefreshing();
void onDataLoadingMore();
完成后记得调用函数：
void onRefreshFinish(boolean);
void onLoadMoreFinish(boolean);
### 注：只是一个简单的能实现功能的Demo，可以在此基础上进行很多修改，自己的项目中已经封装集成了。这里只是提供了一个快速的实现方法。