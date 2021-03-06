# 一切从main()方法开始

Android中一个应用程序是从  ActivityThread.java  中的 main() 方法开始。

main()方法中主要做的事情有:

1. 初始化主线程的 Looper、Handler, 并使主线程进入等待接收消息的无限循环状态

2. 调用 attach() 方法, 发送出初始化 Application 的消息。

## 创建 Application 的消息是如何发送的呢？

ActivityThread 的 attach() 方法最终会发送出一条创建 Application 的消息——H.BIND_APPLICATION 到主线程的 Handler 中。

attach()方法源码如下:

```
    private void attach(boolean system, long startSeq) {
            ...

            final IActivityManager mgr = ActivityManager.getService();
            try {
                mgr.attachApplication(mAppThread, startSeq);
            } catch (RemoteException ex) {
                throw ex.rethrowFromSystemServer();
            }
           ...

    public static IActivityManager getService() {
        return IActivityManagerSingleton.get();
    }

    private static final Singleton<IActivityManager> IActivityManagerSingleton =
            new Singleton<IActivityManager>() {
                @Override
                protected IActivityManager create() {
                    // 这里是通过 ServiceManager 获取到 IBinder 实例
                    final IBinder b = ServiceManager.getService(Context.ACTIVITY_SERVICE);
                    final IActivityManager am = IActivityManager.Stub.asInterface(b);
                    return am;
                }
            };
```

当我们调用 ActivityManager.getService() 获得的实际是一个代理类的实例 ActivityManagerProxy, 其 IActivityManager 接口。
ActivityManagerProxy 是 ActivityManagerNative 的一个内部类。

1. ActivityManagerProxy 的构造函数:

```
public ActivityManagerProxy(IBinder remote) {
        mRemote = remote;
}
```

这个构造函数非常的简单。首先它需要一个 IBinder 参数, 然后赋值给 mRemote 变量。这个mRemote显然是 ActivityManagerProxy 的成员变量，
对它的操作是由 ActivityManagerProxy 来代理间接进行的。这样设计的好处是保护了mRemote，并且能够在操作 mRemote 前执行一些别的事务，
并且我们是以IActivityManager的身份来进行这些操作的！这就非常巧妙了。

## attachApplication(mAppThread)方法

```
	public void attachApplication(IApplicationThread app){
	  ...
	  mRemote.transact(ATTACH_APPLICATION_TRANSACTION, data, reply, 0);
	  ...
	}
```

这个方法中上面这一句是关键:调用了 IBinder 实例的 tansact() 方法，并且把参数放到了 data , 最终传递给 ActivityManagerService。

现在，我们已经基本知道了 IActivityManager 是个什么东东了,其实它就是 ActivityManagerProxy 的一个实现类, 它主要代理了内核与ActivityManagerService 通讯的 Binder 实例。

当调用 mRemote.transact(ATTACH_APPLICATION_TRANSACTION, data, reply, 0), 通过AIDL会远程调用 ActivityManagerService 的 OnTransact 方法, 该方法会调用 ApplicationThread#bindApplicatation(), ApplicationThread 会发送一条 BIND_APPLICATION 的 message.

## ApplicationThread mAppThread

```
	private class ApplicationThread extends IApplicationThread.Stub

```
ApplicationThread 是 ActivityThread 中的一个内部类, ApplicationThread 其实也是一个Binder! ApplicationThread 作为 IApplicationThread 的一个实例, 承担了最后发送 Activity 生命周期、及其它一些消息的任务。

## ActivityManagerService 调度发送初始化消息

经过上面的辗转，ApplicationThread 终于到了 ActivityManagerService 中了。

ActivityManagerService 中有一这样的方法：

```
	private final boolean attachApplicationLocked(IApplicationThread thread
	, int pid) {
	    ...
	    thread.bindApplication();
	    //注意啦！
	    ...
	}

```

ApplicationThread 以 IApplicationThread 的身份到了 ActivityManagerService 中，经过一系列的操作，最终被调用了自己的 bindApplication() 方法,发出初始化 Applicationd 的消息。

```
  public final void bindApplication(String processName, ApplicationInfo appInfo,
                List<ProviderInfo> providers, ComponentName instrumentationName,
                ProfilerInfo profilerInfo, Bundle instrumentationArgs,
                IInstrumentationWatcher instrumentationWatcher,
                IUiAutomationConnection instrumentationUiConnection, int debugMode,
                boolean enableBinderTracking, boolean trackAllocation,
                boolean isRestrictedBackupMode, boolean persistent, Configuration config,
                CompatibilityInfo compatInfo, Map services, Bundle coreSettings,
                String buildSerial, boolean autofillCompatibilityEnabled) {

            ...

            sendMessage(H.BIND_APPLICATION, data);
        }
```

但是，这个地方，我们只要知道最后发了一条 H.BIND_APPLICATION 消息，接着程序开始了。

总结一下,其实是这样子的:

1. ActivityThread 的 attach 方法中的 mgr.attachApplication(mAppThread, startSeq),是通过 AIDL 来调用 ActivityManagerService 中的        attachApplication.

2. 而ActivityManagerService 的 attachApplication 方法中,通过会 IApplicationThread 接口(这里也是一个AIDL)调用 ApplicationThread 的        bindApplication 方法.

3. ApplicationThread 为 ActivityThread 的一个内部类,其 bindApplication 方法会发送一条 BIND_APPLICATION 的 message.当 ActivityThread    的内部 Handler 收到该 message 后会 handleBindApplication 来完成 application 的创建.

# 收到初始化消息之后的世界

上面我们已经找到初始化 Applicaitond 的消息是在哪发送的了。现在，需要看一看收到消息后都发生了些什么。

现在上图的H下面找到第一个消息：H.BIND_APPLICATION。一旦接收到这个消息就开始创建Application了。这个过程是在 handleBindApplication() 中完成的。

```
private void handleBindApplication(AppBindData data) {
    ...
    //通过反射初始化一个 Instrumentation, 反射中用到的 className 来自于 ActivityManagerService 传过来的Binder
    mInstrumentation = (Instrumentation)
        cl.loadClass(data.instrumentationName.getClassName())
        .newInstance();
    ...
    //通过 LoadedApp 命令创建 Application 实例
    Application app = data.info.makeApplication(data.restrictedBackupMode, null);
    mInitialApplication = app;
    ...
    // 回调 Application 的 onCreate() 方法
    mInstrumentation.callApplicationOnCreate(app);
    ...
}

```

## Instrumentation

Instrumentation 会在应用程序的任何代码运行之前被实例化, 它能够允许你监视应用程序和系统的所有交互。

上面的代码我们可以看出，Instrumentation 是在 Application 初始化之前就被创建了。

### 那么它是如何实现监视应用程序和系统交互的呢？

打开这个类你可以发现，最终 Apllication 的创建，Activity 的创建，以及生命周期都会经过这个对象去执行。简单点说，就是把这些操作包装了一层。通过操作 Instrumentation 进而实现上述的功能。

那么这样做究竟有什么好处呢？仔细想想。Instrumentation 作为抽象，当我们约定好需要实现的功能之后，我们只需要给 Instrumentation 添加这些抽象功能，
然后调用就好。 __至于如何实现这些功能，都交给 Instrumentation 仪器的实现对象就好__。

啊！这是多态的运用。啊！这是依赖抽象，不依赖具体的实践。 __这是上层提出需求, 底层定义接口即依赖倒置原则的践行__. 

从代码中可以看到，这里实例化 Instrumentation 的方法是反射！而反射的ClassName是来自于从 ActivityManagerService 中传过来的Binder的。

套路太深！就是为了隐藏具体的实现对象。但是这样耦合性会很低。

看看最后调的 callApplicationOnCreate()方法,来回调 application 对象的 onCreate 方法。

```
    public void callApplicationOnCreate(Application app) {
        app.onCreate();
    }

```
## makeApplication

```
public Application makeApplication(boolean forceDefaultAppClass,
    Instrumentation instrumentation) {
    ...
    //Application 的类名, 明显是要用反射了。
    String appClass = mApplicationInfo.className;
    ...
    //留意下 Context
    ContextImpl appContext = ContextImpl.createAppContext(mActivityThread
        , this);
    //通过 Instrumentation 创建 Application
    app = mActivityThread.mInstrumentation
        .newApplication( cl, appClass, appContext);
    ...
}
```

在这个方法中，我们需要知道的就是，在取得 Application 的实际类名之后，最终的创建工作还是交由 Instrumentation 去完成，就像前面所说的一样。

## 现在把目光移回 Instrumentation

看看 newApplication() 中是如何完成 Application 的创建的。

```
static public Application newApplication(Class<?> clazz
    , Context context) throws InstantiationException
    , IllegalAccessException
    , ClassNotFoundException {
    	 //反射创建，简单粗暴
        Application app = (Application)clazz.newInstance();
         //关注下这里，Application被创建后第一个调用的方法
        //目的是为了绑定Context。
        app.attach(context);
        return app;
    }
```

# LaunchActivity

当 Application 初始化完成后，系统会根据 Manifests 中的配置发送一个 Intent 去启动相应的 Activity。

Ps:Android 9.0 Activity启动流程会有所改变

* Activity：startActivity 方法的真正实现在 Activity 中

* Instrumentation：用来辅助完成启动 Activity 的过程

* ActivityThread（包含 ApplicationThread + ApplicationThreadNative + IApplicationThread）: 真正启动 Activity 的实现都在这里

```
    @Override
    public void startActivity(Intent intent) {
        this.startActivity(intent, null);
    }

    @Override
    public void startActivity(Intent intent, @Nullable Bundle options) {
        if (options != null) {
            startActivityForResult(intent, -1, options);
        } else {
            // Note we want to go through this call for compatibility with
            // applications that may have overridden the method.
            startActivityForResult(intent, -1);
        }
    }

```

显然, 最终都是由 startActivityForResult 来实现的,

```
    public void startActivityForResult(@RequiresPermission Intent intent, int requestCode,
            @Nullable Bundle options) {
         //一般的 Activity 的 mParent为null，mParent 常用在 ActivityGroup 中, ActivityGroup 已废弃
        if (mParent == null) {
            options = transferSpringboardActivityOptions(options);
         //这里会启动新的Activity，核心功能都在mMainThread.getApplicationThread()中完成
            Instrumentation.ActivityResult ar =
                mInstrumentation.execStartActivity(
                    this, mMainThread.getApplicationThread(), mToken, this,
                    intent, requestCode, options);
            if (ar != null) {
         //发送结果，即 onActivityResult 会被调用
                mMainThread.sendActivityResult(
                    mToken, mEmbeddedID, requestCode, ar.getResultCode(),
                    ar.getResultData());
            }
            if (requestCode >= 0) {
                // If this start is requesting a result, we can avoid making
                // the activity visible until the result is received.  Setting
                // this code during onCreate(Bundle savedInstanceState) or onResume() will keep the
                // activity hidden during this time, to avoid flickering.
                // This can only be done when a result is requested because
                // that guarantees we will get information back when the
                // activity is finished, no matter what happens to it.
                mStartedActivity = true;
            }

            cancelInputsAndStartExitTransition(options);
            // TODO Consider clearing/flushing other event sources and events for child windows.
        } else {
         //在ActivityGroup内部的Activity调用startActivity的时候会走到这里，内部处理逻辑和上面是类似的
            if (options != null) {
                mParent.startActivityFromChild(this, intent, requestCode, options);
            } else {
                // Note we want to go through this method for compatibility with
                // existing applications that may have overridden it.
                mParent.startActivityFromChild(this, intent, requestCode);
            }
        }
    }
```

可以发现，真正打开 activity 的实现在 Instrumentation 的 execStartActivity 方法中，去看看

Instrumentation#execStartActivity

```
    public ActivityResult execStartActivity(
        Context who, IBinder contextThread, IBinder token, String target,
        Intent intent, int requestCode, Bundle options) {

        //核心功能在这个 whoThread 中完成，其内部 scheduleLaunchActivity 方法用于完成 activity 的打开
        IApplicationThread whoThread = (IApplicationThread) contextThread;
        if (mActivityMonitors != null) {
            synchronized (mSync) {
                final int N = mActivityMonitors.size();
                for (int i=0; i<N; i++) {
                  //先查找一遍看是否存在这个 activity
                    final ActivityMonitor am = mActivityMonitors.get(i);
                    ActivityResult result = null;
                    if (am.ignoreMatchingSpecificIntents()) {
                        result = am.onStartActivity(intent);
                    }
                    if (result != null) {
                        am.mHits++;
                        return result;
                    } else if (am.match(who, null, intent)) {
                    //如果找到了就跳出循环
                        am.mHits++;
                        if (am.isBlocking()) {
                   //如果目标activity无法打开，直接return
                            return requestCode >= 0 ? am.getResult() : null;
                        }
                        break;
                    }
                }
            }
        }
        try {
            intent.migrateExtraStreamToClipData();
            intent.prepareToLeaveProcess(who);
            //这里才是真正打开 activity 的地方，核心功能在 whoThread 中完成。
            int result = ActivityManager.getService()
                .startActivity(whoThread, who.getBasePackageName(), intent,
                        intent.resolveTypeIfNeeded(who.getContentResolver()),
                        token, target, requestCode, 0, null, options);

                 //这个方法是专门抛异常的，它会对结果进行检查，如果无法打开activity，
		//则抛出诸如ActivityNotFoundException类似的各种异常
            checkStartActivityResult(result, intent);
        } catch (RemoteException e) {
            throw new RuntimeException("Failure from system", e);
        }
        return null;
    }
```

再说一下这个方法 checkStartActivityResult，它也专业抛异常的，看代码，相信大家对下面的异常信息不陌生吧，就是它干的, 其中最熟悉的非 Unable to find explicit activity class 莫属了，如果你在xml中没有注册目标 activity，此异常将会抛出。

再来看看  ActivityManager.getService() 得到的是啥？

```
    public static IActivityManager getService() {
        return IActivityManagerSingleton.get();
    }

    private static final Singleton<IActivityManager> IActivityManagerSingleton =
            new Singleton<IActivityManager>() {
                @Override
                protected IActivityManager create() {
                    final IBinder b = ServiceManager.getService(Context.ACTIVITY_SERVICE);
                    final IActivityManager am = IActivityManager.Stub.asInterface(b);
                    return am;
                }
            };
```

通过上面代码可知得到的是一个 IActivityManager，其为 ActivityManagerService 的远程 Binder 对象。通过远程Binder对象调用 ActivityManagerService 的 startActivity 方法。

接着startActivity 这个调用了很多层级的函数。。这里省略 。。。到后面调用了一个 Ibind 的接口回到 app 进程中 ApplicationThread 类的 handleLaunchActivity 方法。

然后调用 H.sendMessage（） msg.what 为 LAUNCH_ACTIVITY，紧接着 去执行 performLaunchActivity（）, performLaunchActivity里面的动作是 newActivity (真真正正的创建activity）

## 补充

### Android中为什么主线程不会因为Looper.loop()里的死循环卡死？

对于线程既然是一段可执行的代码，当可执行代码执行完成后，线程生命周期便该终止了，线程退出。而对于主线程，我们是绝不希望会被运行一段时间，自己就退出，
那么如何保证能一直存活呢？简单做法就是可执行代码是能一直执行下去的，死循环便能保证不会被退出。

但这里可能又引发了另一个问题，既然是死循环又如何去处理其他事务呢？

答案是：通过创建新线程的方式。

真正会卡死主线程的操作是在回调方法 onCreate/onStart/onResume 等操作时间过长会导致掉帧, 甚至发生ANR。looper.loop 本身不会导致应用卡死。

### 在哪里为这个死循环准备了一个新线程去运转？

事实上在主线程进入消息循环之前便创建了新 binder 线程, 在代码 ActivityThread.main() 中：

```
public static void main(String[] args) {
        ....

        //创建Looper和MessageQueue对象，用于处理主线程的消息
        Looper.prepareMainLooper();

        //创建ActivityThread对象
        ActivityThread thread = new ActivityThread(); 

        //建立Binder通道 (创建新线程)
        thread.attach(false);

        Looper.loop(); //消息循环运行
        throw new RuntimeException("Main thread loop unexpectedly exited");
    }
```
thread.attach(false) 便会创建一个 Binder 线程（即 ApplicationThread Binder的服务端, 用于接收系统服务AMS发送来的事件).
该 Binder 线程通过 Handler 将 Message 发送给主线程。

### 主线程的死循环一直运行是不是特别消耗CPU资源呢？ 

其实不然，这里就涉及到Linux pipe/epoll机制，简单说就是在主线程的MessageQueue没有消息时，便阻塞在loop的queue.next()中的nativePollOnce()方法里, 详情见Android消息机制1-Handler(Java层)，此时主线程会释放CPU资源进入休眠状态，直到下个消息到达或者有事务发生，通过往pipe管道写端写入数据来唤醒主线程工作。这里采用的epoll机制，是一种IO多路复用机制，可以同时监控多个描述符，当某个描述符就绪(读或写就绪)，则立刻通知相应程序进行读或写操作，本质同步I/O，即读写是阻塞的。 

所以说，主线程大多数时候都是处于休眠状态，并不会消耗大量CPU资源。

###  Activity 的生命周期是怎么实现在死循环体外能够执行起来的？

ActivityThread的内部类H继承于Handler，通过handler消息机制，简单说Handler机制用于同一个进程的线程间通信。

Activity的生命周期都是依靠主线程的Looper.loop，当收到不同Message时则采用相应措施。在H.handleMessage(msg)方法中，根据接收到不同的msg，执行相应的生命周期。

system_server 进程是系统进程，java framework框架的核心载体, 里面运行了大量的系统服务. 比如这里提供 ApplicationThreadProxy（简称ATP），ActivityManagerService（简称AMS）,这个两个服务都运行在 system_server 进程的不同线程中. 由于 ATP 和 AMS 都是基于IBinder接口都是binder线程，binder 线程的创建与销毁都是由 binder 驱动来决定的。

 ![](https://github.com/xianfeng92/android-code-read/blob/master/images/ActivityThread_MessageModule.jpg)

App进程则是我们常说的应用程序，主线程主要负责 Activity/Service 等组件的生命周期以及UI相关操作都运行在这个线程；另外，每个App进程中至少会有两个binder线程 ApplicationThread(简称AT) 和 ActivityManagerProxy（简称AMP）。Binder 用于不同进程之间通信，由一个进程的Binder客户端向另一个进程的服务端发送事务，比如图中线程2向线程4发送事务；而handler用于同一个进程中不同线程的通信，比如图中线程4向主线程发送消息。

结合图说说Activity生命周期，比如暂停Activity，流程如下：

* 线程1的AMS中调用线程2的ATP；（由于同一个进程的线程间资源共享，可以相互直接调用，但需要注意多线程并发问题）线程2通过binder传输到App进程的线程4

* 线程4通过handler消息机制，将暂停Activity的消息发送给主线程

* 主线程在looper.loop()中循环遍历消息，当收到暂停Activity的消息时，便将消息分发给ActivityThread.H.handleMessage()方法，再经过方法的调用，
  最后便会调用到Activity.onPause()，当onPause()处理完后，继续循环loop下去。

# 参考

[Android源码分析-Activity的启动过程](https://blog.csdn.net/singwhatiwanna/article/details/18154335)

[3分钟看懂Activity启动流程](https://www.jianshu.com/p/9ecea420eb52)

[Android Launcher 启动 Activity 的工作过程](https://blog.csdn.net/qian520ao/article/details/78156214)

[Android中为什么主线程不会因为Looper.loop()里的死循环卡死？](https://www.zhihu.com/question/34652589)
