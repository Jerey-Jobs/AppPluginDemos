![](/plugin.png)

插件化入门篇-如何启动一个未注册过的Activity
---


几乎所有的插件化都会要的一个需求，启动一个未注册的Activiy，即加载插件包中的Activity，并且主应用并不知道插件应用中会有什么Activity，这是各个插件化框架主力解决的问题之一。

今天我们学习一下占坑式插件化框架的启动Activity原理。

关于动态代理的知识,了解过`Retrofit`的源码的或者看过Java`设计模式之代理模式`的高级使用的，应该都了解了。本章不做介绍，主介绍hook+反射

### Hook是什么？
Hook直白点说就是拦截方法，自己对其参数等进行修改,或者替换返回值，达到自己不可告人的目的的一件事。

## 寻找Hook点

对于启动Activity，老实说光`startActivity`便有很多要说，很多文章会带着你一直追到`ActivityManagerService`中的若干个方法，最后再调用本地的`ActivityThread`里面的方法去启动本进程的Activity。

所以光上面的流程我们看出，我们把要启动的Activity信息发给AMS，其做了各种检查各种操作后真正让`Activity`启动的还是我们的`ActivityThread`

### startActivity流程

我们`startActivity`是context的方法，去找`Context`实现类`class ContextImpl extends Context `。
``` java
    @Override
    public void startActivities(Intent[] intents) {
        warnIfCallingFromSystemProcess();
        startActivities(intents, null);
    }
        /** @hide */
    @Override
    public void startActivitiesAsUser(Intent[] intents, Bundle options, UserHandle userHandle) {
        if ((intents[0].getFlags()&Intent.FLAG_ACTIVITY_NEW_TASK) == 0) {
            throw new AndroidRuntimeException(
                    "Calling startActivities() from outside of an Activity "
                    + " context requires the FLAG_ACTIVITY_NEW_TASK flag on first Intent."
                    + " Is this really what you want?");
        }
        mMainThread.getInstrumentation().execStartActivitiesAsUser(
            getOuterContext(), mMainThread.getApplicationThread(), null,
            (Activity)null, intents, options, userHandle.getIdentifier());
    }
```

看到最后调用的是`mMainThread.getInstrumentation().execStartActivitiesAsUser`方法，不用着急，直接ctrl鼠标左击进去。是`Instrumentation`类。

``` java
    public void execStartActivitiesAsUser(Context who, IBinder contextThread,
            IBinder token, Activity target, Intent[] intents, Bundle options,
            int userId) {
        IApplicationThread whoThread = (IApplicationThread) contextThread;
        if (mActivityMonitors != null) {
            synchronized (mSync) {
                final int N = mActivityMonitors.size();
                for (int i=0; i<N; i++) {
                    final ActivityMonitor am = mActivityMonitors.get(i);
                    if (am.match(who, null, intents[0])) {
                        am.mHits++;
                        if (am.isBlocking()) {
                            return;
                        }
                        break;
                    }
                }
            }
        }
        try {
            String[] resolvedTypes = new String[intents.length];
            for (int i=0; i<intents.length; i++) {
                intents[i].migrateExtraStreamToClipData();
                intents[i].prepareToLeaveProcess();
                resolvedTypes[i] = intents[i].resolveTypeIfNeeded(who.getContentResolver());
            }
            int result = ActivityManagerNative.getDefault()
                .startActivities(whoThread, who.getBasePackageName(), intents, resolvedTypes,
                        token, options, userId);
            checkStartActivityResult(result, intents[0]);
        } catch (RemoteException e) {
        }
    }
```

这边我们看到了，是调用`ActivityManagerNative`的方法启动activity了。进去这个类我们只能看到一堆的binder通信，调用AMS的方法，不过此时我们不用关心了，因为我们知道接下来是
AMS的事情。AMS是活在另一个PID的玩意儿，我们只关心我们自己的pid，另一个进程的东西我们没权限干坏事。

不过这边我们需要注意，`ActivityManagerNative`居然是个单类，那么我们hook它会安全很多，毕竟这个对象是单类。

``` java
    static public IActivityManager getDefault() {
        return gDefault.get();
    }

    private static final Singleton<IActivityManager> gDefault = new Singleton<IActivityManager>() {
        protected IActivityManager create() {
            IBinder b = ServiceManager.getService("activity");
            if (false) {
                Log.v("ActivityManager", "default service binder = " + b);
            }
            IActivityManager am = asInterface(b);
            if (false) {
                Log.v("ActivityManager", "default service = " + am);
            }
            return am;
        }
    };
```

说是说AMS的事情不用关心，但是我们得关心AMS什么时候回调回来，让我们启动Activity。

去`ActivityThread`看，一搜里面有个`handleLaunchActivity`方法，是在Handler里面被调用的，而且`ActivityThread`也是我们喜欢的对象，因为这个对象存在于整个应用生命周期中。

``` java
        public void handleMessage(Message msg) {
            if (DEBUG_MESSAGES) Slog.v(TAG, ">>> handling: " + codeToString(msg.what));
            switch (msg.what) {
                case LAUNCH_ACTIVITY: {   //这个值是常量100
                    Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "activityStart");
                    ActivityClientRecord r = (ActivityClientRecord)msg.obj;

                    r.packageInfo = getPackageInfoNoCheck(
                            r.activityInfo.applicationInfo, r.compatInfo);
                    handleLaunchActivity(r, null);
                    Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                } break;
```

看了这么多，我们可算是知道启动Activity的入口和出口了，下面我们需要进行欺骗。

### 实现欺骗

欺骗系统就欺骗两个地方，我们在`AndroidManifest`里面申明一个假`Activity`，然后在启动真实`Activity`的地方，将`Intent`里面的`Activity`替换成我们已经注册过的。再在`ActivityThread` launch Activity的时候，替换成我们需要启动的便实现了启动一个未注册过的Activity的效果。

### 代码实现
- 写一个占坑Activity，在`AndroidManifest`注册
``` java
        /**
         * 占坑专用
         */
        public class TmpActivity extends Activity {

            @Override
            protected void onCreate(Bundle savedInstanceState) {
                super.onCreate(savedInstanceState);
                setContentView(R.layout.activity_tmp);
            }
        }

        <!--占坑专用Activity-->
        <activity android:name=".TmpActivity"/>
```

- 在`attachBaseContext`中欺骗应用

``` java

    @Override
    protected void attachBaseContext(Context newBase) {
        super.attachBaseContext(newBase);

        try {
            /**
             * 欺骗ActivityManagerNative,将要启动的Activity替换成我们的占坑Activity
             */
            Class<?> activityManagerNativeClass = Class.forName("android.app" +
                    ".ActivityManagerNative");

            Field gDefaultField = activityManagerNativeClass.getDeclaredField("gDefault");
            gDefaultField.setAccessible(true);

            Object gDefault = gDefaultField.get(null);

            // gDefault是一个 android.util.Singleton对象; 我们取出这个单例里面的字段
            Class<?> singleton = Class.forName("android.util.Singleton");
            Field mInstanceField = singleton.getDeclaredField("mInstance");
            mInstanceField.setAccessible(true);

            // ActivityManagerNative 的gDefault对象里面原始的 IActivityManager对象
            final Object rawIActivityManager = mInstanceField.get(gDefault);

            // 创建一个这个对象的代理对象, 然后替换这个字段, 让我们的代理对象帮忙干活
            Class<?> iActivityManagerInterface = Class.forName("android.app.IActivityManager");
            Object proxy = Proxy.newProxyInstance(Thread.currentThread().getContextClassLoader(),
                    new Class<?>[] {iActivityManagerInterface}, new InvocationHandler() {


                        @Override
                        public Object invoke(Object proxy, Method method, Object[] args) throws
                                                                                         Throwable {
                            if ("startActivity".equals(method.getName())) {
                                Intent raw;
                                int index = 0;

                                for (int i = 0; i < args.length; i++) {
                                    if (args[i] instanceof Intent) {
                                        index = i;
                                        break;
                                    }
                                }
                                raw = (Intent) args[index];

                                Intent newIntent = new Intent();

                                // 替身Activity的包名, 也就是我们自己的包名
                                String stubPackage = "com.jerey.activityplugin";

                                // 这里我们把启动的Activity临时替换为 StubActivity
                                ComponentName componentName = new ComponentName(stubPackage,
                                        TmpActivity.class
                                                .getName());
                                newIntent.setComponent(componentName);

                                // 把我们原始要启动的TargetActivity先存起来
                                newIntent.putExtra(EXTRA_TARGET_INTENT, raw);

                                // 替换掉Intent, 达到欺骗AMS的目的
                                args[index] = newIntent;

                                Log.d(TAG, "hook succes{s");
                                return method.invoke(rawIActivityManager, args);
                            }
                            return method.invoke(rawIActivityManager, args);
                        }
                    });
            mInstanceField.set(gDefault, proxy);


            /**
             * 欺骗ActivityThread
             */

            // 先获取到当前的ActivityThread对象
            Class<?> activityThreadClass = Class.forName("android.app.ActivityThread");
            Field currentActivityThreadField = activityThreadClass.getDeclaredField
                    ("sCurrentActivityThread");
            currentActivityThreadField.setAccessible(true);
            Object currentActivityThread = currentActivityThreadField.get(null);

            // 由于ActivityThread一个进程只有一个,我们获取这个对象的mH
            Field mHField = activityThreadClass.getDeclaredField("mH");
            mHField.setAccessible(true);
            final Handler mH = (Handler) mHField.get(currentActivityThread);

            // 设置它的回调, 根据源码:
            // 我们自己给他设置一个回调,就会替代之前的回调;

            //        public void dispatchMessage(Message msg) {
            //            if (msg.callback != null) {
            //                handleCallback(msg);
            //            } else {
            //                if (mCallback != null) {
            //                    if (mCallback.handleMessage(msg)) {
            //                        return;
            //                    }
            //                }
            //                handleMessage(msg);
            //            }
            //        }

            Field mCallBackField = Handler.class.getDeclaredField("mCallback");
            mCallBackField.setAccessible(true);

            mCallBackField.set(mH, new Handler.Callback() {
                @Override
                public boolean handleMessage(Message msg) {
                    if (msg.what == 100) {
                        Object obj = msg.obj;
                        // 根据源码:
                        // 这个对象是 ActivityClientRecord 类型
                        // 我们修改它的intent字段为我们原来保存的即可.
                        // switch (msg.what) {
                        //      case LAUNCH_ACTIVITY: {
                        //          Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "activityStart");
                        //          final ActivityClientRecord r = (ActivityClientRecord) msg.obj;

                        //          r.packageInfo = getPackageInfoNoCheck(
                        //                  r.activityInfo.applicationInfo, r.compatInfo);
                        //         handleLaunchActivity(r, null);
                        try {
                            // 把替身恢复成真身
                            Field intent = obj.getClass().getDeclaredField("intent");
                            intent.setAccessible(true);
                            Intent raw = (Intent) intent.get(obj);

                            Intent target = raw.getParcelableExtra(EXTRA_TARGET_INTENT);
                            raw.setComponent(target.getComponent());

                        } catch (NoSuchFieldException e) {
                            e.printStackTrace();
                        } catch (IllegalAccessException e) {
                            e.printStackTrace();
                        }
                        mH.handleMessage(msg);
                    }
                    return true;
                }
            });


        } catch (Exception e) {
            e.printStackTrace();
        }
    }
```

上面的代码，我们先反射拿到`ActivityManagerNative`,然后动态代理`IActivityManager`，Hook其`startActivity`方法，在里面替换掉intent，并将真实的Intent存放在假Intent的参数里面。

在系统最后调用打开假Intent的时候，我们从Intent中取出参数，并打开真正想打开的Activity。

- 打开Activity
我们和正常使用一样，startActivity就能打开我们未注册的Activity了。
``` java
findViewById(R.id.btn).setOnClickListener(new View.OnClickListener() {
    @Override
    public void onClick(View v) {
        startActivity(new Intent(MainActivity.this, UnregisterActivity.class));
    }
});
```

## 总结
上面只是一个Demo，不能支持support包的`AppCompatActivity`,真正的完整的插件化库任务是艰巨的！
还要支持其他组件，都是很麻烦的事情。

----------
本文作者：Anderson/Jerey_Jobs

博客地址   ： [http://jerey.cn/](http://jerey.cn/)<br>
简书地址   :  [Anderson大码渣](http://www.jianshu.com/users/016a5ba708a0/latest_articles)<br>
github地址 :  [https://github.com/Jerey-Jobs](https://github.com/Jerey-Jobs)
