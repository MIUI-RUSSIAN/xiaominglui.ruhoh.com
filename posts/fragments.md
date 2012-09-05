---
title:
date: '2012-09-05'
description:
categories:
  - 'android api guides notes'
tags:
  - android
  - fragment
---

## Fragments简介
一个[Fragment][1]可以用来提供[Activity][2]中的一部分用户界面或者提供一种行为。我们可以吧[Fragment][1]看作是[Activity][2]中的模块（或者是“sub activity”）。通过这些模块，可以很方便的构建多窗口用户界面或者达到复用的**目的**。

### 特性：
+ [Fragment][1]必须嵌入[Activity][2]，并且它的生命周期受到其所在的activity生命周期的影响。

+ [Fragment][1]作为[Activity][2]布局一部分时，它需要位于布局层级的某个[ViewGroup][3]中。

+ [Fragment][1]也可以不提供UI，作为activiy的工作线程来用。

## Fragments中的设计思想
引入fragments的初衷是为大屏幕设备，比如平板电脑，提供一种更灵活、动态的界面设计支持。因此在使用fragments进行界面设计时需要把握的**设计思想**是：*模块化与考虑activity组件的可复用性*。一个典型的例子如下图所示：

![体现Fragments设计思想的例子](https://developer.android.com/images/fundamentals/fragments.png "体现Fragments设计思想的例子")

## 创建Fragments的方法
作为一个类，其基本用法很直接：继承[Fragment][1]或者其子类并实现相关回调函数。下面介绍应用开发中，使用Fragment的典型操作：
### 为fragment增加用户界面
我们在回调函数_onCreateView()_中指定界面布局，且该回调函数必须返回一个[View][4]对象。

> **注意**：若使用的fragment是[ListFragment][5]的子类，则默认的实现会返回一个[ListView][6]，因此无需覆盖该回调函数。

用下面的例子说明`onCreateView()`的使用方法：

    public static class ExampleFragment extends Fragment {
        @Override
        public View onCreateView(LayoutInflater inflater, ViewGroup container,
                                 Bundle savedInstanceState) {
            // Inflate the layout example_fragment.xml for this fragment
            return inflater.inflate(R.layout.example_fragment, container, false);
        }
    }
    
+ inflater: 用来展开资源中的布局文件的[LayoutInflater][7]
+ container: 要依存的[ViewGroup][3]对象
+ savedInstanceState: 保存之前状态的[Bundle][8]

### 把fragment加入到activity
按照惯例有xml布局文件与动态加入两种方法将fragment加到activity的布局中：

+ 这种方法和Android的其他界面元素一样，不再详细说明。系统会根据布局文件将_onCreateView()_返回的[View][4]放置到界面层次中`<fragment>`元素所对应的位置。

> **注意**：三种提供唯一标识符（必须）的方法：

> 1. 使用_android:id_属性
> 1. 使用_android:tag_属性
> 1. 若未提供以上任何方式，则系统使用fragment所在容器的id替代

+ 在代码中动态加入已有的[ViewGroup][3]

        FragmentManager fragmentManager = getFragmentManager()
        FragmentTransaction fragmentTransaction = fragmentManager.beginTransaction();
        
        ExampleFragment fragment = new ExampleFragment();
        fragmentTransaction.add(R.id.fragment_container, fragment);
        fragmentTransaction.commit();

在activity中加入一个没有界面的fragment:

    fragmentTransaction.add(fragment, tag_for_the_fragment)
    
_tag_可以尽可以用来标示有界面的fragment，也可以标示没有界面的fragment，不过没有界面的fragment只能通过tag来标示。
## 管理Fragments
使用[FragmentManager][9]对象管理。通过该对象，可以：

+ 通过_findFragmentById()_或_findFragmentByTag()_获得activity中已经存在的fragments。
+ 通过_popBackStack()_从_back stack_中将fragment弹出。
+ 通过_addOnBackStackChangedListener()_注册监听器监听_back stack_的变化。
## 执行Fragments的事务
    FragmentManager fragmentManager = getFragmentManager();
    FragmentTransaction fragmentTransaction = fragmentManager.beginTransaction();
每个事务（Transaction）是你期望在同一时间执行的一个变化集合，使用_add()_，_remove()_和_replace()_方法设置改变，然后调用_commit()_方法使改变生效。

在执行_commt()_前可以通过_addToBackStack()_将fragment的变化事务加入后退栈（用户点击后退按钮时可以回退到之前的状态）。

若不将fragment加入后退栈，remove事务执行时该fragment对象会_destroyed_；加入后退栈，对象则会_stopped_

事务的变化过程可以使用动画。

调用_commit()_**不会**立即执行事务。而是由UI线程（main线程）在可以执行时尽快执行。不过，如果需要，你可以从UI线程调用_executePendingTransactions()_立即执行以及提交的事务。一般情况这么做是没必要的，除非事务是其他线程中的一个依赖条件。

> **警告**： 只能在activity保存其状态（用户离开activity）前调用_commit()_提交事务。如果在保存状态后提交，则会抛出异常。因为如果这样做的话，activity的状态restore后，commit之后状态会丢失。如果你不介意这个问题，使用_commitAllowingStateLoss()_提交事务。
## 与Activity之间如何通信
从fragment中获得activity实例：

    View listView = getActivity().findViewById(R.id.list);
从activity中获得fragment实例：

    ExampleFragment fragment = (ExampleFragment) getFragmentManager().findFragmentById(R.id.example_fragment);
    
在fragment中创建到activity的事件回调函数：

1. 在fragment中定义回调接口

        public static class FragmentA extends ListFragment {
            ...
            // Container Activity must implement this interface
            public interface OnArticleSelectedListener {
                public void onArticleSelected(Uri articleUri);
            }
            ...
        }
1. 宿主activity实现fragment中的回调接口
1. fragment关联宿主activity（回调接口类型）

        public static class FragmentA extends ListFragment {
            OnArticleSelectedListener mListener;
            ...
            @Override
            public void onAttach(Activity activity) {
                super.onAttach(activity);
                try {
                    mListener = (OnArticleSelectedListener) activity;
                } catch (ClassCastException e) {
                    throw new ClassCastException(activity.toString() + " must implement OnArticleSelectedListener");
                }
            }
            ...
        }
1. 在fragment中调用回调函数

        public static class FragmentA extends ListFragment {
            OnArticleSelectedListener mListener;
            ...
            @Override
            public void onListItemClick(ListView l, View v, int position, long id) {
                // Append the clicked item's row ID with the content provider Uri
                Uri noteUri = ContentUris.withAppendedId(ArticleColumns.CONTENT_URI, id);
                // Send the event and Uri to the host activity
                mListener.onArticleSelected(noteUri);
            }
            ...
        }
        
在界面上增加Action Bar元素

1. 在_onCreate()_中调用_setHasOptionsMenu (boolean hasMenu)_
1. 实现_onCreateOptionsMenu (Menu menu, MenuInflater inflater)_
1. 实现_onOptionsItemSelected (MenuItem item)_

为fragment注册Context Menu

1. 在_onCreate()_中调用_registerForContextMenu (View view)_
1. 实现_onCreateContextMenu (ContextMenu menu, View v, ContextMenu.ContextMenuInfo menuInfo)_
1. 实现_onContextItemSelected (MenuItem item)_

由上可见在fragment中关联Action Bar和Context Menu的用法一致。

> **注意**：

> + 宿主activity首先接收到on-item-selected回调响应函数，宿主activity未作相应处理时fragment才会收到 on-item-selected回调响应函数。
> + 这种机制同时适用于Option Menu和Context Menu。

## 处理Fragments的生命周期
管理fragment的生命周期与管理activity的生命周期类似。两者最大的区别是它们在各自的回退栈中是如何存储的：

> activity默认情况下stop时被存放在由系统管理的回退栈中，而fragment是被remove时若调用了_addToBackStack()_，则被存放在由宿主activity管理的回退栈中。

除此之外，管理activity生命周期的方法一般都适用于管理fragment的生命周期。

对于fragment，还需考虑的是activity的生命周期如何影响fragment的生命周期。

> **小心**：在fragment中，若需要Context对象，你可以调用`getActivity ()`，不过只有与宿主activity保持关联时才能调用，不然只会返回`null`。

### 协调fragment生命周期与宿主activity生命周期

+ 宿主activity的生命周期直接影响fragment的生命周期：每一个activity生命周期回调函数导致fragment中相似的回调函数被调用。
+ fragment有一些额外的回调函数与宿主activity进行独特的交互以完成一些操作。这些回调说明如下：

  + _onAttach (Activity activity)_: 与宿主activity建立关联时回调
  + _onCreateView (LayoutInflater inflater, ViewGroup container, Bundle savedInstanceState)_: 创建与fragment相关的视图结构时调用
  + _onActivityCreated (Bundle savedInstanceState)_: 宿主activity的_onCreate()_函数返回时调用
  + _onDestroyView ()_: 与fragment相关的视图要被remove时调用
  + _onDetach ()_要与宿主activity解除关联时调用。
  
  ![宿主activity的生命周期对fragment生命周期的影响](https://developer.android.com/images/activity_fragment_lifecycle.png "宿主activity的生命周期对fragment生命周期的影响")
  
  一旦宿主activity进入resumed状态，你就可以自由地增加或者删除fragment，也就是说只有宿主activity处在resumed状态，fragment才可以独立改变（目前体会不深）。一旦宿主activity离开resumed状态，fragment的状态就要受到宿主activity生命状态的驱使。

[0]: https://developer.android.com/guide/components/fragments.html
[1]: https://developer.android.com/reference/android/app/Fragment.html
[2]: https://developer.android.com/reference/android/app/Activity.html
[3]: https://developer.android.com/reference/android/view/ViewGroup.html
[4]: https://developer.android.com/reference/android/view/View.html
[5]: https://developer.android.com/reference/android/app/ListFragment.html
[6]: https://developer.android.com/reference/android/widget/ListView.html
[7]: https://developer.android.com/reference/android/view/LayoutInflater.html
[8]: https://developer.android.com/reference/android/os/Bundle.html
[9]: https://developer.android.com/reference/android/app/FragmentManager.html
