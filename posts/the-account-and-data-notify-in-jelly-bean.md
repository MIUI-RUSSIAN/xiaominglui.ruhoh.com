---
title:
date: '2012-10-08'
description:
categories:
  - 'android api guides notes'
tags:
  - 'android'
  - 'account'
  - 'content provider'
---

# 概述
在Android平台开发Web Service相关的网络应用，难免需要与账户认证、数据同步与更新通知问题打交道。因此在接下来的学习过程中，希望能解决心中的以下疑问：
 
+ 在处理互联网账户认证与更新通知问题时，Android提供了哪些可用接口？
+ Android 4.1 中，账户认证与更新通知的流程？
+ Android 4.1中，账户认证与更新通知底层实现背后所隐含的设计思想？
 
参考前辈探索出的劳动成果，可知上述问题涉及 _ContentService_ 和 _AccountManagerService_ 这两个服务。其中 _ContentService_ 包含以下两个主要功能：
 
+ 它是Android平台中数据更新通知的执行者。
+ 它是Android平台中数据同步服务的管理中枢。当用户通过Android手机中的Contacts应用将联系人信息同步到远端服务器时，就需要和 _ContentService_ 交互。
 
_AccountManagerService_ 负责管理Android手机中用户的账户，在4.0.1版本中，这些账户指的是用户的在线（online）账户，例如用户在Google、Facebook上注册的账户。4.1版本中，出现了管理本地账户的代码，看来Google准备在以后的Android版本中提供本地多用户支持，具体对用户来说就是可以像目前流行的桌面电脑操作系统那样，一台设备可以支持多个不同的本地账户，用不同的账户登录，呈现不同的应用、主题等等。由于目前正式发行的4.1版本并没有提供本地多用户支持的功能，故分析的重点仍是在线账户部分。
 
# 数据更新通知机制分析
Web Service相关的网络应用，按照用户的使用流程，一般是先进行账户认证（添加账户），接下来才会有数据同步和数据更新通知的问题。不过在这里先分析数据更新通知机制，主要原因是这个部分相对要简单一些（参考[深入理解Android 卷II][0]），先捡软柿子捏，也给自己增加一些学习的信心。
 
如果应用需要监控某数据项的变化，可以采用一个类似_while_循环的语句不断查询以判断其值是否发生了改变。显然，这种实现方式的效率很低。了解设计模式的话，不难发现数据通知机制对应的是观察者（Observer）模式。Android在这个问题上也遵循常法，采用观察者模式实现数据更新通知机制，具体包含两个步骤：第一步，注册观察者；第二步，通知观察者。
 
## 初识 _ContentService_
 
_SystemServer_ 中启动 _ContentService_ 的代码如下： 
[SystemServer.java][1]::ServerThread.run
 
    public void run() {
    ...
    ContentService.main(context,
                    factoryTest == SystemServer.FACTORY_TEST_LOW_LEVEL);
    ...
    }
   
以上代码中直接调用了 _ContentService_ 的`main()`函数。且在一般情况下，该函数第二个参数为`false`。此main函数的代码如下： 
[ContentService.java][2]::main
 
    public static IContentService main(Context context, boolean factoryTest) {
        // 构造ContentService实例
        ContentService service = new ContentService(context, factoryTest);
        // 将ContentService注册到ServiceManager中，其注册名叫content
        ServiceManager.addService(ContentResolver.CONTENT_SERVICE_NAME, service);
        return service;
    }
   
_ContentService_ 的构造函数代码如下： 
[ContentService.java][2]::ContentService
 
    /*package*/ ContentService(Context context, boolean factoryTest) {
        mContext = context;
        mFactoryTest = factoryTest;
        getSyncManager(); // 获取同步服务管理对象，接下来看它的代码
    }
   
[ContentService.java][2]::getSyncManager
 
    private SyncManager getSyncManager() {
        synchronized(mSyncManagerLock) {
            try {
                // Try to create the SyncManager, return null if it fails (e.g. the disk is full).
                if (mSyncManager == null) mSyncManager = new SyncManager(mContext, mFactoryTest);
            } catch (SQLiteException e) {
                Log.e(TAG, "Can't create SyncManager", e);
            }
            return mSyncManager;
        }
    }
   
目前看到的只是 _ContentService_ 基本的初始化过程，还比较简单，看到这里也不会对数据更新通知机制的具体实现有什么概念。不过根据前面对更新通知机制两个步骤的介绍，自然我们接下来首先要看看注册观察者的代码实现。
 
## _ContentResolver_ 的`registerContentObserver()`分析
 
在Android中[ContentProviders][3]是数据源接口，是观察者模式中的主题。因此应该实现注册观察者的代码。结合[ContentProviders][3]的相关知识与文档，不难找到： 
[ContentResolver.java][4]::registerContentObserver_
 
    public final void registerContentObserver(Uri uri, boolean notifyForDescendents,
            ContentObserver observer)
    {
        try {
            getContentService().registerContentObserver(uri, notifyForDescendents,
                    observer.getContentObserver());
        } catch (RemoteException e) {
        }
    }
   
根据源码可知该方法不可被覆盖，这个方法的参数很简单也很清楚，根据文档很容易明白。
 
这个函数最终调用 _ContentService_ 的`registerContentObserver()`函数，接下来看看_ContentService_ 的`registerContentObserver()`函数的代码 
[ContentService.java][2]::registerContentObserver_
 
    public void registerContentObserver(Uri uri, boolean notifyForDescendents,
            IContentObserver observer) {
        if (observer == null || uri == null) {
            throw new IllegalArgumentException("You must pass a valid uri and observer");
        }
        synchronized (mRootNode) {
            // ContentService要做的事情其实很简单，就是保存uri和observer的对应关系到
            // 其内部变量mRootNode中
            mRootNode.addObserverLocked(uri, observer, notifyForDescendents, mRootNode,
                    Binder.getCallingUid(), Binder.getCallingPid());
            if (false) Log.v(TAG, "Registered observer " + observer + " at " + uri +
                    " with notifyForDescendents " + notifyForDescendents);
        }
    }
   
`mRootNode`是 _ContentService_ 的成员变量，其类型为 `ObserverNode` 。 `ObserverNode` 的组织形式是数据结构中的树，其叶子节点的类型为 `ObserverEntry` ，它保存了 `uri` 和对应的 _IContentObserver_ 对象（暂不分析其内部实现）。
 
至此，客户端已经为某数据项设置了 _ContentObserver_ 。继续看更新通知机制实施的第二步——通知观察者。
 
## [ContentResolver][3]的 `notifyChange()` 分析
 
Provider的数据更新后，会调用[ContentResolver][3]的 `nofifyChange(uri, null)` ，一般情况第二个参数为 `null` 。（不为 `null` 的情况在后面本节结论中有说明）例如在 _MediaProvider_ 中的 `update()` 函数： 
[MediaProvider.java][6]::update
 
    public int update(Uri uri, ContentValues initialValues, String userWhere,
            String[] whereArgs) {
        int count;
        // Log.v(TAG, "update for uri="+uri+", initValues="+initialValues);
        int match = URI_MATCHER.match(uri);
        DatabaseHelper helper = getDatabaseForUri(uri);
        if (helper == null) {
            throw new UnsupportedOperationException(
                    "Unknown URI: " + uri);
        }
        helper.mNumUpdates++;
 
        SQLiteDatabase db = helper.getWritableDatabase();
       
        ......
       
        synchronized (sGetTableAndWhereParam) {
            getTableAndWhere(uri, match, userWhere, sGetTableAndWhereParam);
           
            ......
           
            switch (match) {
            ......
                case VIDEO_MEDIA:
                case VIDEO_MEDIA_ID:
                    {
                        ContentValues values = new ContentValues(initialValues);
                        // Don't allow bucket id or display name to be updated directly.
                        // The same names are used for both images and table columns, so
                        // we use the ImageColumns constants here.
                        values.remove(ImageColumns.BUCKET_ID);
                        values.remove(ImageColumns.BUCKET_DISPLAY_NAME);
                        // If the data is being modified update the bucket values
                        String data = values.getAsString(MediaColumns.DATA);
                        if (data != null) {
                            computeBucketValues(data, values);
                        }
                        computeTakenTime(values);
                        helper.mNumUpdates++;
                        count = db.update(sGetTableAndWhereParam.table, values,
                                sGetTableAndWhereParam.where, whereArgs);
                        // if this is a request from MediaScanner, DATA should contains file path
                        // we only process update request from media scanner, otherwise the requests
                        // could be duplicate.
                        if (count > 0 && values.getAsString(MediaStore.MediaColumns.DATA) != null) {
                            helper.mNumQueries++;
                            Cursor c = db.query(sGetTableAndWhereParam.table,
                                    READY_FLAG_PROJECTION, sGetTableAndWhereParam.where,
                                    whereArgs, null, null, null);
                            if (c != null) {
                                try {
                                    while (c.moveToNext()) {
                                        long magic = c.getLong(2);
                                        if (magic == 0) {
                                            requestMediaThumbnail(c.getString(1), uri,
                                                    MediaThumbRequest.PRIORITY_NORMAL, 0);
                                        }
                                    }
                                } finally {
                                    c.close();
                                }
                            }
                        }
                    }
                    break;
                   
                ......   
               
            }
        }
        // in a transaction, the code that began the transaction should be taking
        // care of notifications once it ends the transaction successfully
        if (count > 0 && !db.inTransaction()) {
            getContext().getContentResolver().notifyChange(uri, null);
        }
        return count;
    }
 
下面接着看[ContentResolver][3]的 `notifyChange()` 
[ContentResolver.java][4]::nofifyChange
 
    public void notifyChange(Uri uri, ContentObserver observer) {
        notifyChange(uri, observer, true /* sync to network */);
    }
 
    public void notifyChange(Uri uri, ContentObserver observer, boolean syncToNetwork) {
        // 第三个参数syncToNetwork用于控制是否需要发起一次数据同步请求
        try {
            // 调用ContentService的nofifyChange函数
            getContentService().notifyChange(
                    uri, observer == null ? null : observer.getContentObserver(),
                    observer != null && observer.deliverSelfNotifications(), syncToNetwork);
        } catch (RemoteException e) {
        }
    }
   
[ContentService.java][2]::nofifyChange
 
    public void notifyChange(Uri uri, IContentObserver observer,
            boolean observerWantsSelfNotifications, boolean syncToNetwork) {
        if (Log.isLoggable(TAG, Log.VERBOSE)) {
            Log.v(TAG, "Notifying update of " + uri + " from observer " + observer
                    + ", syncToNetwork " + syncToNetwork);
        }
 
        int userId = UserId.getCallingUserId();
        // This makes it so that future permission checks will be in the context of this
        // process rather than the caller's process. We will restore this before returning.
        long identityToken = clearCallingIdentity();
        try {
            ArrayList<ObserverCall> calls = new ArrayList<ObserverCall>();
            synchronized (mRootNode) {
                mRootNode.collectObserversLocked(uri, 0, observer, observerWantsSelfNotifications,
                        calls);
            }
            final int numCalls = calls.size();
            for (int i=0; i<numCalls; i++) {
                ObserverCall oc = calls.get(i);
                try {
                    oc.mObserver.onChange(oc.mSelfChange, uri);
                    if (Log.isLoggable(TAG, Log.VERBOSE)) {
                        Log.v(TAG, "Notified " + oc.mObserver + " of " + "update at " + uri);
                    }
                } catch (RemoteException ex) {
                    synchronized (mRootNode) {
                        Log.w(TAG, "Found dead observer, removing");
                        IBinder binder = oc.mObserver.asBinder();
                        final ArrayList<ObserverNode.ObserverEntry> list
                                = oc.mNode.mObservers;
                        int numList = list.size();
                        for (int j=0; j<numList; j++) {
                            ObserverNode.ObserverEntry oe = list.get(j);
                            if (oe.observer.asBinder() == binder) {
                                list.remove(j);
                                j--;
                                numList--;
                            }
                        }
                    }
                }
            }
            if (syncToNetwork) {
                SyncManager syncManager = getSyncManager();
                if (syncManager != null) {
                    syncManager.scheduleLocalSync(null /* all accounts */, userId,
                            uri.getAuthority());
                }
            }
        } finally {
            restoreCallingIdentity(identityToken);
        }
    }
   
## 更新通知机制总结和深入探讨
根据上述分析，可以总结出数据更新通知机制的流程图 ![数据更新通知机制的流程图][7]
 
结合流程图我们可以得出以下结论：
 
+ 4.1版本在数据更新通知部分的实现与4.0.1版本完全一样，没有变化。
+ Android使用观察者模式实现了数据更新通知机制。
+ 与典型的观察者模式不一样的地方是：一般情况下，观察者（Observer）对象的维护与通知由 _ContentService_ 集中管理。不过也提供了接口方便按照主题对象（provider）自己处理观察者的维护和通知工作。
 
作为开发者，所开发的应用需要关注特定数据的变化时，使用该数据更新通知机制的步骤为：
 
1. 从应用的 _Context_ 中获得 _ContentResovler_ 对象实例
1. 调用 _ContentResovler_ 的 `final void registerContentObserver(Uri uri, boolean notifyForDescendents, ContentObserver observer)()` ，其中各参数的具体含义请异步参看 [ContentResolver][3] API文档
1. 在应用中实现 _ContentObserver_ 的 `onChange()` 方法以处理相应数据变化
 
个人认为，谷歌提供这样实现数据更新通知机制有以下几点好处：
 
+ 可以使开发者在provider开发上无需考虑观察者管理的问题，专注于数据逻辑的处理。
+ 集中统一处理观察者的维护和通知，可以使不同开发者开发的provider获得一致的管理效率。
+ 提供必要的接口，给开发者提供优化观察者管理效率的机会。
 
 
# 账户添加与认证机制分析
_AccountManagerService_ 负责管理手机中用户的在线账户，主要涉及的账户的添加和删除、_AuthToken_ 的获取和更新。要研究这部分内容，需要了解以下方面的知识背景：
 
+ Android 2.0 （API Level 5+）的基础
+ [Service](http://developer.android.com/guide/components/services.html)
+ [OAuth 2.0](http://oauth.net/2/)
+ [OAuth 1.0](http://tools.ietf.org/html/rfc5849)
 
对于开发应用来说，[AccountManager][8] 是使用 _AccountManagerService_ 所提供相关服务的接口。从SDK文档中可以看到，其主要支持以下操作：
 
+ 添加账户
 
        addAccount()
        addAccountExplicitly()
       
+ 获得AuthToken
 
        getAuthToken()
        getAuthTokenByFeatures()
       
接下来我们以添加账户这条线索探索Android所提供的 _AccountManagerService_ 的实现以及[AccountManager][8]的使用方法。
 
## 初识 _AccountManagerService_
还是先看看 _AccountManagerService_ 创建时的代码：
 
[SystemServer.java][1]::ServerThread.run
 
    // The AccountManager must come before the ContentService
    try {
        Slog.i(TAG, "Account Manager");
        // 注册AccountManagerService到ServiceManager，服务名为account
        ServiceManager.addService(Context.ACCOUNT_SERVICE,
                new AccountManagerService(context));
    } catch (Throwable e) {
        Slog.e(TAG, "Failure starting Account Manager", e);
    }
   
_AccountManagerService_ 构造函数的代码如下：
 
[AccountManagerService.java][9]::AccountManagerService
 
    public AccountManagerService(Context context) {
        this(context, context.getPackageManager(), new AccountAuthenticatorCache(context));
    }
 
其中创建了一个 _AccountAuthenticatorCache_ 的对象，接下来就看看这个对象是干嘛用的。
 
### _AccountAuthenticatorCache_ 分析
 
_AccountAuthenticatorCache_ 是Android平台中账户验证服务（Account Authenticator Service， AAS）的管理中心。AAS由应用程序通过在 _AndroidManife.xml_ 中输出符合指定要求的Servic信息而来。这些符合要求的具体格式在后面介绍。
 
先来看AccountAuthenticatorCache类图![AccountAuthenticatorCache类图][10]所示的派生关系：
 
由上面的派生关系可知：
 
+ AccountAuthenticatorCache是RegisteredServicesCache的子类。RegisteredServicesCache是一个模板类，专门用于管理系统中指定Service的信息收集和更新，具体是哪些Service，则由RegisteredServicesCache构造时的泛型参数指定。对于AccountAuthenticatorCache的情况，这个参数就是AuthenticatorDescription。
+ AuthenticatorDescription继承了Parcelable接口，这代表它可以跨Binder传递。该类描述了AAS相关的信息。
+ AccountAuthenticatorCache实现了IAccountAuthenticatorCache接口。这个接口供外部调用者使用以获取AAS的信息。
 
接下来看AccountAuthenticatorCache构造的具体代码： 
[AccountAuthenticatorCache.java][11]::AccountAuthenticatorCache
 
    public AccountAuthenticatorCache(Context context) {
        /*
        ACTION_AUTHENTICATOR_INTENT值为“android.accounts.AccountAuthenticator” - interfaceName
        AUTHENTICATOR_META_DATA_NAME值为“android.accounts.AccountAuthenticator” - metaDataName
        AUTHENTICATOR_ATTRIBUTES_NAME值为“account-authenticator” - attributeName
        */
        super(context, AccountManager.ACTION_AUTHENTICATOR_INTENT,
                AccountManager.AUTHENTICATOR_META_DATA_NAME,
                AccountManager.AUTHENTICATOR_ATTRIBUTES_NAME, sSerializer);
    }
 
_AccountAuthenticatorCache_ 调用其基类的构造函数时，传递了3个字符串参数，这3个参数用于控制RegisteredServiceCache从PackageManagerService获取哪些Service的信息。
 
下面看看 _RegisteredServicesCache_ 构造时干了些什么事？ 
[RegisteredServicesCache.java][12]::RegisteredServicesCache
 
    public RegisteredServicesCache(Context context, String interfaceName, String metaDataName,
            String attributeName, XmlSerializerAndParser<V> serializerAndParser) {
        mContext = context;
        // 保存传递进来的参数
        mInterfaceName = interfaceName;
        mMetaDataName = metaDataName;
        mAttributesName = attributeName;
        mSerializerAndParser = serializerAndParser;
 
        File dataDir = Environment.getDataDirectory();
        File systemDir = new File(dataDir, "system");
        // syncDir指向 /data/system/registered_service目录
        File syncDir = new File(systemDir, "registered_services");
        // 下面这个文件指向syncDir目录下的android.accounts.AccountAuthenticator.xml
        mPersistentServicesFile = new AtomicFile(new File(syncDir, interfaceName + ".xml"));
       
        //生成服务信息
        generateServicesMap();
 
        final BroadcastReceiver receiver = new BroadcastReceiver() {
            @Override
            public void onReceive(Context context1, Intent intent) {
                generateServicesMap();
            }
        };
        // 注册Package安装、卸载和更新等广播监听者
        mReceiver = new AtomicReference<BroadcastReceiver>(receiver);
        IntentFilter intentFilter = new IntentFilter();
        intentFilter.addAction(Intent.ACTION_PACKAGE_ADDED);
        intentFilter.addAction(Intent.ACTION_PACKAGE_CHANGED);
        intentFilter.addAction(Intent.ACTION_PACKAGE_REMOVED);
        intentFilter.addDataScheme("package");
        mContext.registerReceiver(receiver, intentFilter);
        // Register for events related to sdcard installation.
        IntentFilter sdFilter = new IntentFilter();
        sdFilter.addAction(Intent.ACTION_EXTERNAL_APPLICATIONS_AVAILABLE);
        sdFilter.addAction(Intent.ACTION_EXTERNAL_APPLICATIONS_UNAVAILABLE);
        mContext.registerReceiver(receiver, sdFilter);
    }
 
由以上代码可知：
 
+ 成员变量mPersistentServicesFile指向 _/data/system/registered\_service_ 目录下的一个文件，该文件保存了以前获取的对应Service的信息。该文件在没有root过的手机上应该是由于权限的原因无法看到，root手机或者通过`adb`可以看到。就 _AccountAuthenticator_ 而言，`mPersistentServicesFile`指向该目录的 _android.accounts.AccountAuthenticator.xml_文件。
+ 由于 _RegisteredServicesCache_ 管理的是系统中指定Service的信息，当系统中有Package安装、卸载或更新时，RegisteredServicesCache也需要对应更新自己的信息，因为有些Service可能会随着APK的变化而变化。
 
目前，对`generateServicesMap()`函数还不清楚，下面就看看其代码： 
[RegisteredServicesCache.java][12]::generateServicesMap
 
    void generateServicesMap() {
        // 获取 PackageManager 接口，用来和 PackageManagerService 交互
        PackageManager pm = mContext.getPackageManager();
        ArrayList<ServiceInfo<V>> serviceInfos = new ArrayList<ServiceInfo<V>>();
        /*
        在本例中，查询PKMS中满足Intent Action为“android.accounts.AccountAuthenticator”的服务信息。由以下代码可知，这些信息指的是Service中声明的MetaData信息
        */
        List<ResolveInfo> resolveInfos = pm.queryIntentServices(new Intent(mInterfaceName),
                PackageManager.GET_META_DATA);
        for (ResolveInfo resolveInfo : resolveInfos) {
            try {
                ServiceInfo<V> info = parseServiceInfo(resolveInfo);
                if (info == null) {
                    Log.w(TAG, "Unable to load service info " + resolveInfo.toString());
                    continue;
                }
                serviceInfos.add(info);
            } catch (XmlPullParserException e) {
                Log.w(TAG, "Unable to load service info " + resolveInfo.toString(), e);
            } catch (IOException e) {
                Log.w(TAG, "Unable to load service info " + resolveInfo.toString(), e);
            }
        }
 
        synchronized (mServicesLock) {
            if (mPersistentServices == null) {
                readPersistentServicesLocked();
            }
            mServices = Maps.newHashMap();
            StringBuilder changes = new StringBuilder();
            for (ServiceInfo<V> info : serviceInfos) {
                // four cases:
                // - doesn't exist yet
                //   - add, notify user that it was added
                // - exists and the UID is the same
                //   - replace, don't notify user
                // - exists, the UID is different, and the new one is not a system package
                //   - ignore
                // - exists, the UID is different, and the new one is a system package
                //   - add, notify user that it was added
                Integer previousUid = mPersistentServices.get(info.type);
                if (previousUid == null) {
                    changes.append("  New service added: ").append(info).append("\n");
                    mServices.put(info.type, info);
                    mPersistentServices.put(info.type, info.uid);
                    if (!mPersistentServicesFileDidNotExist) {
                        notifyListener(info.type, false /* removed */);
                    }
                } else if (previousUid == info.uid) {
                    if (Log.isLoggable(TAG, Log.VERBOSE)) {
                        changes.append("  Existing service (nop): ").append(info).append("\n");
                    }
                    mServices.put(info.type, info);
                } else if (inSystemImage(info.uid)
                        || !containsTypeAndUid(serviceInfos, info.type, previousUid)) {
                    if (inSystemImage(info.uid)) {
                        changes.append("  System service replacing existing: ").append(info)
                                .append("\n");
                    } else {
                        changes.append("  Existing service replacing a removed service: ")
                                .append(info).append("\n");
                    }
                    mServices.put(info.type, info);
                    mPersistentServices.put(info.type, info.uid);
                    notifyListener(info.type, false /* removed */);
                } else {
                    // ignore
                    changes.append("  Existing service with new uid ignored: ").append(info)
                            .append("\n");
                }
            }
 
            ArrayList<V> toBeRemoved = Lists.newArrayList();
            for (V v1 : mPersistentServices.keySet()) {
                if (!containsType(serviceInfos, v1)) {
                    toBeRemoved.add(v1);
                }
            }
            for (V v1 : toBeRemoved) {
                mPersistentServices.remove(v1);
                changes.append("  Service removed: ").append(v1).append("\n");
                notifyListener(v1, true /* removed */);
            }
            if (changes.length() > 0) {
                Log.d(TAG, "generateServicesMap(" + mInterfaceName + "): " +
                        serviceInfos.size() + " services:\n" + changes);
                writePersistentServicesLocked();
            } else {
                Log.d(TAG, "generateServicesMap(" + mInterfaceName + "): " +
                        serviceInfos.size() + " services unchanged");
            }
            mPersistentServicesFileDidNotExist = false;
        }
    }
 
我们接着看`parseServiceInof()`方法是怎么解析服务信息的： 
[RegisteredServicesCache.java][12]::parseServiceInfo
 
    private ServiceInfo<V> parseServiceInfo(ResolveInfo service)
            throws XmlPullParserException, IOException {
        android.content.pm.ServiceInfo si = service.serviceInfo;
        ComponentName componentName = new ComponentName(si.packageName, si.name);
 
        PackageManager pm = mContext.getPackageManager();
 
        XmlResourceParser parser = null;
        try {
            // 解析MetaData信息
            parser = si.loadXmlMetaData(pm, mMetaDataName);
            if (parser == null) {
                throw new XmlPullParserException("No " + mMetaDataName + " meta-data");
            }
 
            AttributeSet attrs = Xml.asAttributeSet(parser);
 
            int type;
            while ((type=parser.next()) != XmlPullParser.END_DOCUMENT
                    && type != XmlPullParser.START_TAG) {
            }
 
            String nodeName = parser.getName();
            if (!mAttributesName.equals(nodeName)) {
                throw new XmlPullParserException(
                        "Meta-data does not start with " + mAttributesName +  " tag");
            }
           
            // 调用子类实现的 parseServiceAttributes 得到一个真实的对象，在本例中它是
            // AuthenticatorDescription。
 
            V v = parseServiceAttributes(pm.getResourcesForApplication(si.applicationInfo),
                    si.packageName, attrs);
            if (v == null) {
                return null;
            }
            final android.content.pm.ServiceInfo serviceInfo = service.serviceInfo;
            final ApplicationInfo applicationInfo = serviceInfo.applicationInfo;
            final int uid = applicationInfo.uid;
            return new ServiceInfo<V>(v, componentName, uid);
        } catch (NameNotFoundException e) {
            throw new XmlPullParserException(
                    "Unable to load resources for pacakge " + si.packageName);
        } finally {
            if (parser != null) parser.close();
        }
    }
 
`parseServiceInfo()` 将解析Service中的MetaData信息，然后调用子类实现的 `parseServiceAttributes()` 函数，以获取特定类型Service的信息。
 
_AccountAuthenticatorCache_ 分析总结：
 
从上面的源码分析可知， _AccountAuthenticatorCache_ 的功能为：当应用（APK）被安装、更新和删除时，扫描据应用（APK）的 _AndroidManifest.xml_ ，收集相关信息生成 _android.accounts.AccountAuthenticator.xml_ 文件。这个文件保存了系统当前可用的在线账户类型信息。系统会根据这些信息在系统设置中账户与同步的页面显示对应在线账户的图标、名称，并根据应用（APK）的其他设置显示用户点击账户图标后的后续认证界面。
 
相关信息如下： 
用于设置账户类型、图标、名称等
 
    <account-authenticator xmlns:android="http://schemas.android.com/apk/res/android"
        android:accountType="com.trello"
        android:icon="@drawable/ic_launcher"
        android:smallIcon="@drawable/ic_launcher"
        android:label="@string/account_name"
        android:accountPreferences="@xml/account_preferences"
    />
 
用于关联后续认证动作
 
    <!-- The authenticator service -->
    <service
        android:name=".authenticator.AuthenticationService"
        android:exported="true">
        <intent-filter>
            <action
                android:name="android.accounts.AccountAuthenticator" />
        </intent-filter>
        <meta-data
            android:name="android.accounts.AccountAuthenticator"
            android:resource="@xml/authenticator" />
    </service>
 
android.accounts.AccountAuthenticator.xml文件的典型内容
 
    <?xml version='1.0' encoding='utf-8' standalone='yes' ?>
    <services>
    <service uid="10039" type="com.android.exchange" />
    <service uid="10039" type="com.android.email" />
    <service uid="10048" type="com.trello" />
    </services>
 
> 提示：上面文件中的uid是在为PackageManagerService解析APK文件时赋予APK的。详细代码在 frameworks/base/services/java/com/android/server/pm/Settings.java 中的 newUserIdLPw 函数。
 
### _AccountManagerService_ 构造函数分析
_AccountManagerService_ 构造函数代码如下（这个部分4.1与4.0.1不一样）： 
[AccountManagerService.java][13]::AccountManagerService
 
    public AccountManagerService(Context context, PackageManager packageManager,
            IAccountAuthenticatorCache authenticatorCache) {
        mContext = context;
        mPackageManager = packageManager;
 
        mMessageThread = new HandlerThread("AccountManagerService");
        mMessageThread.start();
        mMessageHandler = new MessageHandler(mMessageThread.getLooper());
 
        mAuthenticatorCache = authenticatorCache;
        mAuthenticatorCache.setListener(this, null /* Handler */);
 
        sThis.set(this);
 
        UserAccounts accounts = initUser(0);
 
        IntentFilter intentFilter = new IntentFilter();
        intentFilter.addAction(Intent.ACTION_PACKAGE_REMOVED);
        intentFilter.addDataScheme("package");
        mContext.registerReceiver(new BroadcastReceiver() {
            @Override
            public void onReceive(Context context1, Intent intent) {
                purgeOldGrantsAll();
            }
        }, intentFilter);
 
        IntentFilter userFilter = new IntentFilter();
        userFilter.addAction(Intent.ACTION_USER_REMOVED);
        mContext.registerReceiver(new BroadcastReceiver() {
            @Override
            public void onReceive(Context context, Intent intent) {
                onUserRemoved(intent);
            }
        }, userFilter);
    }
   
与 4.0.1 的源码比较后，发现有以下变化：
 
1. 与 _accounts.db_ 相关的代码封装到 _UserAccounts_ 类中。从 _UserAccounts_ 的构造函数中可以看到相比之前，增加了 `userId` 字段，这应该与本地多用户支持有关。
1. 监听 _ACTION_USER_REMOVED_ 事件，用于处理本地账户的删除操作
1. 不难看到 _AccountManagerService_ 增加了本地账户的管理操作（多用户支持）
 
## _AccountManager_ `addAccount()` 分析
这一节分析 _AccountManagerService_ 中的一个重要的函数，即`addAccount`，其功能是为某项账户添加一个用户。下面以Email为例来认识AAS的处理流程。
 
 
### _AccountManager_ 的 `addAccount()` 发起请求
_AccountManagerService_ 是一个运行在 _SystemServer_ 中的服务，客户端进程必须借助 _AccountManager_ 的`addAccount()` 函数开始分析。
 
[AccountManager.java][14]::addAccount
 
    public AccountManagerFuture<Bundle> addAccount(final String accountType,
            final String authTokenType, final String[] requiredFeatures,
            final Bundle addAccountOptions,
            final Activity activity, AccountManagerCallback<Bundle> callback, Handler handler) {
        if (accountType == null) throw new IllegalArgumentException("accountType is null");
        final Bundle optionsIn = new Bundle();
        if (addAccountOptions != null) {
            optionsIn.putAll(addAccountOptions); //保存客户端传入的addAccountOptions
        }
        optionsIn.putString(KEY_ANDROID_PACKAGE_NAME, mContext.getPackageName());
 
        // 构造一个匿名类对象，该类继承自AmsTask，并实现了doWork函数。addAccount返回前
        // 将调用该对象的start函数
        return new AmsTask(activity, handler, callback) {
            public void doWork() throws RemoteException {
                mService.addAcount(mResponse, accountType, authTokenType,
                        requiredFeatures, activity != null, optionsIn);
            }
        }.start();
    }
 
在以上代码中：
 
+ `addAccount` 的返回值类型是 `AccountManagerFuture<Bundle>` 。其中， `AccountManagerFuture` 是一个模板Interface，其真实类型只有在分析 `addAccount()` 的实现时才能知道。
+ `addAccount()` 的第一个参数 `accountType` 代表账户类型。该参数不能为空。
+ `authTokenType` 、 `requiredFeatures`和`addAccountOptions`与具体的AAS服务有关。如果想添加指定账户类型的Account，则须对其背后的AAS有所了解。
+ `activity`：此参数和界面有关。例如有些AAS需要用户输入用户名和密码，故需启动一个Activity。在这种情况下，AAS会返回一个Intent，客户端将通过这个Activity启动Intent所标示的Activity。具体情况见后面介绍。
+ `callback` 和 `handler` ：这两个参数与如何获取 `addAccount()` 返回结果有关。如这两个参数为空，客户端则须单独启动一个线程去调用 `AccountManagerFuture` 的 `getResult()` 函数。
 
 
在以上代码中，_AccountManager_ 的 `addAccount()` 函数将返回一个匿名类对象，该匿名类继承自 _AmsTask_ 类。那么， _AmsTask_ 又是什么呢？
 
#### _AmsTask_ 介绍
先看 ![_AmsTask_ 继承关系图][15]：
 
由上图可知：
 
+ _AmsTask_ 继承自 _FutureTask_ ，并实现了 _AccountManagerFuture_ 接口。 _FutureTask_ 是Java concurrent库中的一个常用的类。 _AmsTask_ 定义了一个 `doWork()` 虚函数，该函数必须由子类实现。
+ 一个 _AmsTask_ 对象中有一个 `mResponse` 成员，该成员的类型是 _AmsTask_ 中的内部类 _Response_ 。从 _Response_ 的派生关系可知， _Response_ 将参与Binder通信，并且它是Binder通信的Bn端。而 _AccountManagerService_ 的 `addAccount()` 将得到它的Bp端对象。当添加完账户后， _AccountManagerService_ 会通过这个Bp端对象的 `onResult()` 或 `onError()` 函数向 _Response_ 通知处理结果。
 
#### _AmsTask_匿名类处理分析
_AccountManager_ 的 `addAccount()` 最终返回给客户端的是一个 _AmsTask_ 的子类，首先来了解它的构造函数，其代码如下： 
[AccountManager.java][14]::AmsTask
 
    public AmsTask(Activity activity, Handler handler, AccountManagerCallback<Bundle> callback) {
        super(new Callable<Bundle>() {
            public Bundle call() throws Exception {
                throw new IllegalStateException("this should never be called");
            }
        }); // 调用基类构造函数
 
        // 保存客户端传递的参数
        mHandler = handler;
        mCallback = callback;
        mActivity = activity;
        mResponse = new Response(); // 构造一个Response对象，并保存到mResponse中
    }
 
下一步调用的是这个匿名类的 `start()` 函数，代码如下： 
[AccountManager.java][14]::AmsTask.start
 
    public final AccountManagerFuture<Bundle> start() {
        try {
            doWork();
        } catch (RemoteException e) {
            setException(e);
        }
        return this;
    }
 
匿名类实现的 `doWork()` 函数即下面这个函数： 
[AccountManager.java][14]::addAccount
 
    public void doWork() throws RemoteException {
        mService.addAcount(mResponse, accountType, authTokenType,
                requiredFeatures, activity != null, optionsIn);
    }
 
### _AccountManagerService_ `addAccount()` 转发请求
[AccountManagerService.java][13]:: addAccount
 
    public void addAcount(final IAccountManagerResponse response, final String accountType,
           final String authTokenType, final String[] requiredFeatures,
            final boolean expectActivityLaunch, final Bundle optionsIn) {
        if (Log.isLoggable(TAG, Log.VERBOSE)) {
            Log.v(TAG, "addAccount: accountType " + accountType
                    + ", response " + response
                    + ", authTokenType " + authTokenType
                    + ", requiredFeatures " + stringArrayToString(requiredFeatures)
                    + ", expectActivityLaunch " + expectActivityLaunch
                    + ", caller's uid " + Binder.getCallingUid()
                    + ", pid " + Binder.getCallingPid());
        }
        if (response == null) throw new IllegalArgumentException("response is null");
        if (accountType == null) throw new IllegalArgumentException("accountType is null");
        checkManageAccountsPermission();
 
        UserAccounts accounts = getUserAccountsForCaller();
        final int pid = Binder.getCallingPid();
        final int uid = Binder.getCallingUid();
        final Bundle options = (optionsIn == null) ? new Bundle() : optionsIn;
        options.putInt(AccountManager.KEY_CALLER_UID, uid);
        options.putInt(AccountManager.KEY_CALLER_PID, pid);
 
        long identityToken = clearCallingIdentity();
        try {
            // 创建一个匿名类对象，该匿名类派生自Session类，最后调用该匿名类的bind函数
            new Session(accounts, response, accountType, expectActivityLaunch,
                    true /* stripAuthTokenFromResult */) {
                public void run() throws RemoteException {
                    mAuthenticator.addAccount(this, mAccountType, authTokenType, requiredFeatures,
                            options);
                }
 
                protected String toDebugString(long now) {
                    return super.toDebugString(now) + ", addAccount"
                            + ", accountType " + accountType
                            + ", requiredFeatures "
                            + (requiredFeatures != null
                              ? TextUtils.join(",", requiredFeatures)
                              : null);
                }
            }.bind();
        } finally {
            restoreCallingIdentity(identityToken);
        }
    }
 
相比 4.0.1版本，4.1中增加了类型为UserAccounts的变量 `accounts` ，并加入到 _Session_ 对象的构造函数中。联想到在前面 _AccountManagerService_ 构造函数的变化，这部分变化应该也是为本地多用户支持服务的。
 
由以上代码可知， _AccountManagerService_ 的`addAccount()`函数最后也创建了一个匿名类对象，该匿名类派生自 _Session_ 。 `addAccount()` 最后还调用了这个对象的 `bind()` 函数。下面看看 _Session_ 做了什么？
 
#### _Session_ 介绍
Session家族成员如![Session家族示意图][16]所示：
 
由图可知：
 
+ _Session_ 从 _IAccountAuthenticatorResponseStub_ 派生，这表明它将参与与Binder通信，并且是Bn端。那么这个Binder通信的目标是谁呢？它正是具体的AAS服务。 _AccountManagerService_ 会将自己传递给AAS，这样，AAS就得到 _IAccountAuthenticatorResponse_ 的Bp端对象。当AAS完成了具体的账户添加工作后，会通过 _IAccountAuthenticatorResponse_ 的Bp端对象向 _Session_ 返回处理结果。
+ _Session_ 通过 `mResponse` 成员变量指向来自客户端的 _IAccountManagerResponse_ 接口，当 _Session_ 收到AAS的返回结果后，又通过 _IAccountAuthenticatorResponse_ 的Bp端对象向客户端返回处理结果。
+ _Session_ `mAuthenticator` 变量的类型是 _IAccountAuthenticator_ ，它用于和远端的AAS通信。客户端发起的请求将通过 _Session_ 经由 `mAuthenticator` 调用对应AAS中的函数。
 
由上面两幅图可知， _AccountManagerService_ 在 `addAccount()` 流程中起桥梁的作用，具体如下：
 
+ 客户端将请求发送给 _AccountManagerService_ ，然后 _AccountManagerService_ 再转发给对应的AAS
+ AAS处理完的结果先返回给 _AccountManagerService_ ，再由 _AccountManagerService_ 返回给客户端。
 
下面看 _Session_ 匿名类的处理
 
#### _Session_ 匿名类处理分析
首先调用 _Session_ 的构造函数，代码为： 
[AccountManagerService.java][13]::Session
 
    public Session(UserAccounts accounts, IAccountManagerResponse response, String accountType,
            boolean expectActivityLaunch, boolean stripAuthTokenFromResult) {
        super();
        if (response == null) throw new IllegalArgumentException("response is null");
        if (accountType == null) throw new IllegalArgumentException("accountType is null");
        mAccounts = accounts;
        mStripAuthTokenFromResult = stripAuthTokenFromResult;
        mResponse = response;
        mAccountType = accountType;
        mExpectActivityLaunch = expectActivityLaunch;
        mCreationTime = SystemClock.elapsedRealtime();
        synchronized (mSessions) {
            // 将这个匿名类对象保存到AccountManagerService中mSessions成员中
            mSessions.put(toString(), this);
        }
        try { // 监听客户端死亡消息
            response.asBinder().linkToDeath(this, 0 /* flags */);
        } catch (RemoteException e) {
            mResponse = null;
            binderDied();
        }
    }
 
 
很明显，相对4.0.1，在构造函数中增加了 `accounts` 参数，为多用户支持服务。
 
获得匿名对象类后， `addAccount()` 将调用其 `bind()` 函数，该函数由 _Session_ 实现，代码如下： 
[AccountManagerService.java][13]::Session:bind
 
    void bind() {
        if (Log.isLoggable(TAG, Log.VERBOSE)) {
            Log.v(TAG, "initiating bind to authenticator type " + mAccountType);
        }
        if (!bindToAuthenticator(mAccountType)) {
            Log.d(TAG, "bind attempt failed for " + toDebugString());
            onError(AccountManager.ERROR_CODE_REMOTE_EXCEPTION, "bind failure");
        }
    }
 
 
`bindToAuthenticator()` 的代码： 
[AccountManagerService.java][13]::Session:bindToAuthenticator
 
    private boolean bindToAuthenticator(String authenticatorType) {
        // 从 mAuthenticatorCache中查询满足指定类型的服务信息
        AccountAuthenticatorCache.ServiceInfo<AuthenticatorDescription> authenticatorInfo =
                mAuthenticatorCache.getServiceInfo(
                        AuthenticatorDescription.newKey(authenticatorType));
        if (authenticatorInfo == null) {
            if (Log.isLoggable(TAG, Log.VERBOSE)) {
                Log.v(TAG, "there is no authenticator for " + authenticatorType
                        + ", bailing out");
            }
            return false;
        }
 
        Intent intent = new Intent();
        intent.setAction(AccountManager.ACTION_AUTHENTICATOR_INTENT);
        intent.setComponent(authenticatorInfo.componentName);
        if (Log.isLoggable(TAG, Log.VERBOSE)) {
            Log.v(TAG, "performing bindService to " + authenticatorInfo.componentName);
        }
        // 通过 bindService 启动指定的服务，成功与否将通过第二个参数传递的 ServiceConnection接口返回
        if (!mContext.bindService(intent, this, Context.BIND_AUTO_CREATE, mAccounts.userId)) {
            if (Log.isLoggable(TAG, Log.VERBOSE)) {
                Log.v(TAG, "bindService to " + authenticatorInfo.componentName + " failed");
            }
            return false;
        }
 
 
        return true;
    }
 
由以上代码可知， _Session_ 的 `bind()` 函数将启动指定类型的Service，这是通过 `bindService()` 函数完成的。如果服务启动成功， _Session_ 的 `onServiceConnected()` 函数将被调用，这部分代码如下： 
[AccountManagerService.java][13]::Session:onServiceConnected
 
    public void onServiceConnected(ComponentName name, IBinder service) {
        mAuthenticator = IAccountAuthenticator.Stub.asInterface(service);
        try {
            run(); // 调用匿名类实现的run函数
        } catch (RemoteException e) {
            onError(AccountManager.ERROR_CODE_REMOTE_EXCEPTION,
                    "remote exception");
        }
    }
 
匿名类实现的 `run()` 函数代码如下： 
[AccountManagerService.java][13]::addAccount 返回的匿名类
 
    public void run() throws RemoteException {
        // 调用远端AAS实现的addAccount函数
        mAuthenticator.addAccount(this, mAccountType, authTokenType, requiredFeatures,
                options);
    }
    
由以上代码可知， _AccountManagerService_ 在 `addAccount()` 中最终调用AAS实现的 `addAccount()` 函数。
 
### _EasAuthenticatorService_ 处理请求
_EasAuthenticatorService_ 的创建是 _AccountManagerService_ 调用了 `bindService()` 完成的，该函数会触发 _EasAuthenticatorService_ 的 `onBind()` 函数的调用，代码如下：  
[EasAuthenticatorService.java][17]::onBind
 
    public IBinder onBind(Intent intent) {
        if (AccountManager.ACTION_AUTHENTICATOR_INTENT.equals(intent.getAction())) {
            return new EasAuthenticator(this).getIBinder();
        } else {
            return null;
        }
    }
 
下面来分析 _EasAuthenticator_
 
#### _EasAuthenticator_ 介绍
_EasAuthenticator_ 是 _EasAuthenticatorService_ 定义的内部类，其家族关系如 ![EasAuthenticator][18] 所示：
 
由图可知：
 
+ _EasAuthenticator_ 从 _AbstractAccountAuthenticator_ 类派生。 _AbstractAccountAuthenticator_ 内有有一个 `mTransport` 的成员变量，其类型是 _AbstractAccountAuthenticator_ 的内部类 _Transport_ 。在前面的 `onBind()` 函数中， _EasAuthenticator_ 的 `getIBinder()` 函数返回的就是这个变量。
+ _Transport_ 类继承自Binder，故它将参与Binder通信，并且是 _IAccountAuthenticator_ 的Bn端。 _Session_ 匿名类通过 `onSserviceConnected()` 函数将得到一个 _IAccountAuthenticator_ 的Bp端对象。
 
当由 _AccountManagerService_ 的 `addAccount()` 创建的那个 _Session_ 匿名类调用 _IAccountAuthenticator_ Bp端对象的 `addAccount()` 时，将触发位于Email进程中的 _IAccountAuthenticator_  Bn端的 `addAccount()` 。下面来分析Bn端的 `addAccount()` 函数。
 
#### _EasAuthenticator_ 的 `addAccount()`函数分析
根据上文的描述可知，Email进程中首先被触发的是 _IAccountAuthenticator_ Bn端的 `addAccount()` 函数，其代码如下： 
[AbstractAccountAuthenticator.java][19]::Transport:addAccount
 
        public void addAccount(IAccountAuthenticatorResponse response, String accountType,
                String authTokenType, String[] features, Bundle options)
                throws RemoteException {
            if (Log.isLoggable(TAG, Log.VERBOSE)) {
                Log.v(TAG, "addAccount: accountType " + accountType
                        + ", authTokenType " + authTokenType
                        + ", features " + (features == null ? "[]" : Arrays.toString(features)));
            }
            checkBinderPermission();
            try {
                final Bundle result = AbstractAccountAuthenticator.this.addAccount(
                    new AccountAuthenticatorResponse(response),
                        accountType, authTokenType, features, options);
                if (Log.isLoggable(TAG, Log.VERBOSE)) {
                    result.keySet(); // force it to be unparcelled
                    Log.v(TAG, "addAccount: result " + AccountManager.sanitizeResult(result));
                }
                if (result != null) {
                    response.onResult(result);
                }
            } catch (Exception e) {
                handleException(response, "addAccount", accountType, e);
            }
        }
 
本例中 _AbstractAccountAuthenticator_ 子类（即 _EasAuthenticator_ ）实现的 `addAccount()` 函数，代码如下： 
[EasAuthenticatorService.java][17]::EasAuthenticator.addAccount
 
        public Bundle addAccount(AccountAuthenticatorResponse response, String accountType,
                String authTokenType, String[] requiredFeatures, Bundle options)
                throws NetworkErrorException {
            // There are two cases here:
            // 1) We are called with a username/password; this comes from the traditional email
            //    app UI; we simply create the account and return the proper bundle
            if (options != null && options.containsKey(OPTIONS_PASSWORD)
                    && options.containsKey(OPTIONS_USERNAME)) {
                final Account account = new Account(options.getString(OPTIONS_USERNAME),
                        AccountManagerTypes.TYPE_EXCHANGE);
                AccountManager.get(EasAuthenticatorService.this).addAccountExplicitly(
                            account, options.getString(OPTIONS_PASSWORD), null);
 
                // Set up contacts syncing.  ExchangeService will use info from ContentResolver
                // to determine syncability of Contacts for Exchange
                boolean syncContacts = false;
                if (options.containsKey(OPTIONS_CONTACTS_SYNC_ENABLED) &&
                        options.getBoolean(OPTIONS_CONTACTS_SYNC_ENABLED)) {
                    syncContacts = true;
                }
                ContentResolver.setIsSyncable(account, ContactsContract.AUTHORITY, 1);
                ContentResolver.setSyncAutomatically(account, ContactsContract.AUTHORITY,
                        syncContacts);
 
                // Set up calendar syncing, as above
                boolean syncCalendar = false;
                if (options.containsKey(OPTIONS_CALENDAR_SYNC_ENABLED) &&
                        options.getBoolean(OPTIONS_CALENDAR_SYNC_ENABLED)) {
                    syncCalendar = true;
                }
                ContentResolver.setIsSyncable(account, CalendarProviderStub.AUTHORITY, 1);
                ContentResolver.setSyncAutomatically(account, CalendarProviderStub.AUTHORITY,
                        syncCalendar);
 
                // Set up email syncing, as above
                boolean syncEmail = false;
                if (options.containsKey(OPTIONS_EMAIL_SYNC_ENABLED) &&
                        options.getBoolean(OPTIONS_EMAIL_SYNC_ENABLED)) {
                    syncEmail = true;
                }
                ContentResolver.setIsSyncable(account, EmailContent.AUTHORITY, 1);
                ContentResolver.setSyncAutomatically(account, EmailContent.AUTHORITY,
                        syncEmail);
 
                Bundle b = new Bundle();
                b.putString(AccountManager.KEY_ACCOUNT_NAME, options.getString(OPTIONS_USERNAME));
                b.putString(AccountManager.KEY_ACCOUNT_TYPE, AccountManagerTypes.TYPE_EXCHANGE);
                return b;
            // 2) The other case is that we're creating a new account from an Account manager
            //    activity.  In this case, we add an intent that will be used to gather the
            //    account information...
            } else {
               Bundle b = new Bundle();
                Intent intent =
                    AccountSetupBasics.actionSetupExchangeIntent(EasAuthenticatorService.this);
                intent.putExtra(AccountManager.KEY_ACCOUNT_AUTHENTICATOR_RESPONSE, response);
                b.putParcelable(AccountManager.KEY_INTENT, intent);
                return b;
            }
        }
 
不同的AAS有自己特定的处理逻辑，以上代码向读者展示了 _EasAuthenticatorService_ 的工作流程。虽然每个AAS的处理方式各有不同，但Android还是定义了一些通用的参数，例如，OPTION_USERNAME用于表示用户名，OPTION_PASSWORD用于表示密码等。关于这方面的内容，读者可以查阅SDK中 [AccountManager][8] 的文档说明。
 
### 返回值的处理流程
在 _EasAuthenticator_ 的 `addAccount()` 返回处理结果后， _AbstractAuthenticator_ 将通过 _IAccountAuthenticatorResponse_ 的 `onResult()` 将其返回给由 _AccountManagerService_ 创建的 _Session_ 匿名类对象。来看 _Session_ 的 `onResult()` 函数，其代码如下： 
[AccountManagerService.java][9]::Session:onResult
 
    public void onResult(Bundle result) {
        mNumResults++;
        if (result != null && !TextUtils.isEmpty(result.getString(AccountManager.KEY_AUTHTOKEN))) {
            String accountName = result.getString(AccountManager.KEY_ACCOUNT_NAME);
            String accountType = result.getString(AccountManager.KEY_ACCOUNT_TYPE);
            if (!TextUtils.isEmpty(accountName) && !TextUtils.isEmpty(accountType)) {
                Account account = new Account(accountName, accountType);
                cancelNotification(getSigninRequiredNotificationId(mAccounts, account));
            }
        }
        IAccountManagerResponse response;
        if (mExpectActivityLaunch && result != null
                && result.containsKey(AccountManager.KEY_INTENT)) {
            response = mResponse;
        } else {
            response = getResponseAndClose();
        }
        if (response != null) {
            try {
                if (result == null) {
                    if (Log.isLoggable(TAG, Log.VERBOSE)) {
                        Log.v(TAG, getClass().getSimpleName()
                                + " calling onError() on response " + response);
                    }
                    response.onError(AccountManager.ERROR_CODE_INVALID_RESPONSE,
                            "null bundle returned");
                } else {
                    if (mStripAuthTokenFromResult) {
                        result.remove(AccountManager.KEY_AUTHTOKEN);
                    }
                    if (Log.isLoggable(TAG, Log.VERBOSE)) {
                        Log.v(TAG, getClass().getSimpleName()
                                + " calling onResult() on response " + response);
                    }
                    response.onResult(result);
                }
            } catch (RemoteException e) {
                // if the caller is dead then there is no one to care about remote exceptions
                if (Log.isLoggable(TAG, Log.VERBOSE)) {
                    Log.v(TAG, "failure while notifying response", e);
                }
            }
        }
    }
 
客户端的 _IAccountAuthenticatorResponse_ 接口由 _AmsTask_ 内部类 _Response_ 实现，其代码为： 
[AccountManager.java][14]::AmsTask::Response.onResult_
 
    public void onResult(Bundle bundle) {
        Intent intent = bundle.getParcelable(KEY_INTENT);
        if (intent != null && mActivity != null) {
            // since the user provided an Activity we will silently start intents
            // that we see
            mActivity.startActivity(intent);
            // leave the Future running to wait for the real response to this request
        } else if (bundle.getBoolean("retry")) {
            try {
                doWork();
            } catch (RemoteException e) {
                // this will only happen if the system process is dead, which means
                // we will be dying ourselves
            }
        } else {
            // 将返回结果保存起来，当客户调用getResult时，就会得到相关结果
            set(bundle);
        }
    }
 
## [AccountManager][8] 的 `addAccount()` 分析总结
该流程涉及3个模块，分别是客户端、AccountManagerService和AAS。其中 `addAccount()` 相关流程如 ![AccountManager的addAccount处理流程][20] 所示：
 
 
## Android账户管理分析总结
经过上面的代码级分析，我得出以下信息：
 
+ Android基于Java concurrent并发类构建了一种异步的账户管理流程框架，账户的添加、认证等主要操作都在这个异步的框架中完成。
+ 虽然账户管理框架是异步的，但 [AccountManager][8] 也提供了以同步的方法获得AuthToken的方法： `blockingGetAuthToken()`
+ Android以异步的方式实现账户管理，很大一部分考虑应该是管理在线账户时，需要访问网络，因此适合采用异步方法。
+ 账户管理框架目前只涉及在线账户，但从4.1版本开始，代码中已经出现本地账户管理的代码，也许不久的新版本中将出现本地多用户支持的功能。
 
## Android账户管理框架的使用方法
 
下面将应用开发中使用Android账户管理框架的步骤概况如下：
 
1. 在 _AndroidManifest.xml_ 文件中申明相关权限、服务
1. 在应用中获得 [AccountManager][8] 实例对象
1. 根据需求选择添加账户或者获得Token的具体方法
1. 如果所采用的认证服务器采用了不同于OAuth 2.0的协议，则自己需要继承 [AbstractAccountAuthenticator][21] 实现自己的AAS。
 
 
# 参考文献
1. 《深入理解Android 卷II》，邓凡平著
1. http://developer.android.com/
 
[0]: http://book.douban.com/subject/11542973/
[1]: ./frameworks/base/services/java/com/android/server/SystemServer.java "./frameworks/base/services/java/com/android/server/SystemServer.java"
[2]: ./frameworks/base/core/java/android/content/ContentService.java "./frameworks/base/core/java/android/content/ContentService.java"
[3]: https://developer.android.com/reference/android/content/ContentResolver.html
[4]: ./frameworks/base/core/java/android/content/ContentResolver.java "./frameworks/base/core/java/android/content/ContentResolver.java"
[5]: https://developer.android.com/guide/topics/providers/content-providers.html
[6]: ./packages/providers/mediaprovider/src/com/android/providers/media/MediaProvider.java "./packages/providers/mediaprovider/src/com/android/providers/media/MediaProvider.java"
[7]: .\md_pics\2012-09-13_16-57-28_512.jpg "数据更新通知机制的流程图"
[8]: http://developer.android.com/reference/android/accounts/AccountManager.html
[9]: ./frameworks/base/core/java/android/accounts/AccountManagerService.java "./frameworks/base/core/java/android/accounts/AccountManagerService.java"
[10]: .\md_pics\2012-09-24_09-15-12_369.jpg "AccountAuthenticatorCache类图"
[11]: ./frameworks/base/core/java/android/accounts/AccountAuthenticatorCache.java "/frameworks/base/core/java/android/accounts/AccountAuthenticatorCache.java"
[12]: ./frameworks/base/core/java/android/content/pm/RegisteredServicesCache.java "./frameworks/base/core/java/android/content/pm/RegisteredServicesCache.java"
[13]: ./frameworks/base/core/java/android/accounts/AccountManagerService.java "./frameworks/base/core/java/android/accounts/AccountManagerService.java"
[14]: ./frameworks/base/core/java/android/accounts/AccountManager.java "./frameworks/base/core/java/android/accounts/AccountManager.java"
[15]: .\md_pics\2012-09-24_10-17-28_320.jpg "AmsTask继承关系"
[16]: .\md_pics\2012-09-24_10-36-26_117.jpg "Session家族示意图"
[17]: ./packages/apps/email/src/com/android/email/service/EasAuthenticatorService.java "./packages/apps/email/src/com/android/email/service/EasAuthenticatorService.java"
[18]: .\md_pics\2012-09-24_11-09-26_962.jpg 'EasAuthenticator家族类图'
[19]: ./frameworks/base/core/java/android/accounts/AbstractAccountAuthenticator.java "./frameworks/base/core/java/android/accounts/AbstractAccountAuthenticator.java"
[20]: .\md_pics\2012-09-24_11-26-44_685.jpg "AccountManager的addAccount处理流程"
[21]: http://developer.android.com/reference/android/accounts/AbstractAccountAuthenticator.html
