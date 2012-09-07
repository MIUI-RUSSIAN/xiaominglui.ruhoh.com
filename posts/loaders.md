---
title:
date: '2012-09-07'
description:
tags:
  - android
  - loader
categories:
  - 'android api guides notes'
---

# Loaders简介
这个类从Android 3.0引入，带来的**功能**为：

> 在activity或者fragment中异步加载数据变的很容易

Loaders具有下面的特点：

+ 每一个[Activity][0]和[Fragment][1]对象都可以使用
+ 提供异步加载数据的方法
+ 可以监测数据源并在有变化时更新新内容
+ 配置改变重建时可以自动重连之前数据加载器的cursor，因此无需重新查询数据

# 在应用中如何使用Loaders

1. 启动一个Loader  
  一个[Activity][0]或者[Fragment][1]中最多只能有一个[LoaderManager][2]实例，但这个[LoaderManager][2]实例可以管理一个或多个[Loader][3]。    
  一般情况，在activity的`onCreate()`方法或者fragment的`onActivityCreated()`方法中初始化[Loader][3]对象。代码如下：
  
        // Prepare the loader.  Either re-connect with an existing one,
        // or start a new one.
        getLoaderManager().initLoader(0, null, this);    
1. 重启一个[Loader][3]（可选）  
  `initLoader()`是启动（若没有则创建新的）一个[Loader][3]，若你需要废弃旧的数据，重新加载数据的话可以通过`restartLoader()`方法达到目的。如下例所示：
  
        public boolean onQueryTextChanged(String newText) {
            // Called when the action bar search text has changed.  Update
            // the search filter, and restart the loader to do a new query
            // with this filter.
            mCurFilter = !TextUtils.isEmpty(newText) ? newText : null;
            getLoaderManager().restartLoader(0, null, this);
            return true;
        }  
  
1. 如何使用[LoaderManager][2]的回调函数  
  `LoaderManager.LoaderCallbacks`是客户端（activity或者fragment对象）与[LoaderManager][2]交互的接口。因此客户端（activity或者fragment）需要实现该接口。该接口中包含下面几个回调函数：
  
  + `onCreateLoader()` — 以指定ID实例化一个新的[Loader][3]对象并返回。
  + `onLoadFinished()` — 之前创建的loader完成加载后调用该函数，因此在这里可以处理获得的数据。
  + `onLoaderReset()` — 之前创建的loader被复位，导致数据不可用，在这里客户端需要释放对数据的使用。





[0]: https://developer.android.com/reference/android/app/Activity.html
[1]: https://developer.android.com/reference/android/app/Fragment.html
[2]: https://developer.android.com/reference/android/app/LoaderManager.html
[3]: https://developer.android.com/reference/android/content/Loader.html
