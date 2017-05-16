---
layout: post
title:  "仿美团/大众点评选择城市界面"
date:   2017-05-12 10:52:30 +0800
categories: Android
tag: Android开发
---
本文实现的是美团、大众点评等app的选择城市界面。不过最近发现美团选择城市、微信联系人界面的列表header并没有固定了，不知道是出于什么原因。  
特别感谢：  

- [列表头部固定](https://github.com/timehop/sticky-headers-recyclerview)  
- [优雅的为recyclerview添加header和footer](http://blog.csdn.net/lmj623565791/article/details/51854533)

### 实现
之前的项目，获取城市列表都是从服务器拉取，不仅浪费流量而且效率极低，因为城市列表这种数据改动几率非常非常小，数据很固定。所以自己跟服务器端商量后，将所有的城市列表数据拉下来后保存到了sqlite数据库中。暂时还没有做升级的处理。只是简单的将数据库中的所有数据展示处理，实现类似美团等app的选择城市界面。  

- 整个界面分为两个部分：一是城市列表界面，一是搜索界面。所以这里采用fragment的hide和show来进行这两个界面的切换。  

- 搜索的时候根据输入的字符判断：如果全部为中文，就搜索Name字段；其他就搜索PinYin字段，搜索为空的界面或者提示还没有添加，这个根据需要可自行添加。  

- RecyclerView添加StickyHeader的方法很简单，只需要一句话：
``` 
mRecyclerView.addItemDecoration(new StickyRecyclerHeadersDecoration(mAdapter));
```
这里的mAdapter使用了hong-yang大神修改的HeaderAndFooterWrapper，里面根据itemType来判断header和footer，同时也实现了StickyRecyclerHeadersAdapter，里面有关于stickyHeader的各种逻辑。关于这个stickyHeader的实现，后面会专门写文章进行分析。这里只需要知道如何使用就行了。

- Activity或者Fragment在从数据库中查询完数据后，需要将每个header表示的字母在RecyclerView.Adapter中对应的position，方便后续点击SliderView中的各字母时RecyclerView的跳转。Map<String, Integer> mIndexer就存放了对应每个字母的position。SliderView中点击字母时或者进行滑动的事件处理通过SliderView中的OnItemClickListener实现的。这里我们进行的操作是：
```
((LinearLayoutManager) mRecyclerView.getLayoutManager()).scrollToPositionWithOffset(mIndexer.get(s), 0);
```

### Tips

- 这里的RecyclerView添加了header，所以在adapter的各种方法中，我们需要获取到每个item的实际位置：
```
position - getHeadersCount()
```
否则会影响到stickyHeader的展示，一开始没有注意到这个，因为使用的是大神提供的Demo，代码里面也没有进行修改，出现了很多问题。

- DBManager很简单的使用的系统自带的
 ```
 SQLiteDatabase.openOrCreateDatabase(dbPath, null);
 ```
查询数据也使用
```
sqliteDB.query(SQLiteDatabase db，String[] columns, String selection, String[] selectArgs, String groupBy, String having, String orderBy);
```
并没有使用第三方的框架，性能问题待考虑。
	