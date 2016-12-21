---
title: React-Native 源码分析一-如何启动JS页面
date: 2016-09-09 11:42:11
tags: React-Native
---
用React-Native也有1个月了，好多疑惑一直挂在心头，没有得到很好的答案，有道是：

		纸上得来终觉浅,绝知此事要躬行

今天来源码中一探究竟,博主使用的环境是
>  "react": "15.3.1",

>  "react-native": "^0.33.0",


先看第一个问题

#### 一切的开始-startReactApplication
	想要搞清楚这个问题，那首先要知道在start一个ReactActivity的时候发生了什么，那下面就一步步的跟着源码走一遍。

在react-native 0.33 版本上我发现ReactActivity使用了类似MVP的模式重构了ReactActivity,
原来在onCreate中的大量代码交给了mDelegate（ReactActivityDelegate）来处理
>ReactActivity.java

```
  @Override
  protected void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    mDelegate.onCreate(savedInstanceState);
  }
```
看到了ReactActivityDelegate的类注释我们就明白了facebook的良苦用心

 * Delegate class for {@link ReactActivity} and {@link ReactFragmentActivity}. You can subclass this to provide custom implementations for e.g. {@link #getReactNativeHost()}, if your Application class doesn't implement {@link ReactApplication}.
 
原来多了一个ReactFragmentActivity的实现，需要共享Delegate,这个在0.33的release notes中也能发现。（学习ReactNative的时候，官方文档，issues，release note都是必须要关注的，很多时候他们能帮你走出遇到的大坑）

* Add ReactFragmentActivity, share delegate with ReactActivity (3c4fd42) - @foghina

那么现在一切都开始于 mDelegate.onCreate(savedInstanceState);

>ReactActivityDelegate.java

```

  protected void onCreate(Bundle savedInstanceState) {
    if (getReactNativeHost().getUseDeveloperSupport() && Build.VERSION.SDK_INT >= 23) {
      // Get permission to show redbox in dev builds.
    	弹窗权限判断代码
    }

    if (mMainComponentName != null) {
      loadApp(mMainComponentName);
    }
    mDoubleTapReloadRecognizer = new DoubleTapReloadRecognizer();
  }
```
再看loadapp
```
 protected void loadApp(String appKey) {
    if (mReactRootView != null) {
      throw new IllegalStateException("Cannot loadApp while app is already running.");
    }
    mReactRootView = createRootView();
    mReactRootView.startReactApplication(
      getReactNativeHost().getReactInstanceManager(),
      appKey,
      getLaunchOptions());
      getPlainActivity().setContentView(mReactRootView);
  }
```
getPlainActivity().setContentView(mReactRootView)我们很熟悉，给Activity设置view，那么mReactRootView 就变得很重要了，我们都知道rn中activity只是个载体，所有的逻辑都在这个mReactRootView中。

```
public class ReactRootView extends SizeMonitoringFrameLayout implements RootView 

public class SizeMonitoringFrameLayout extends FrameLayout 
```
ReactRootView是继承SizeMonitoringFrameLayout ,而SizeMonitoringFrameLayout继承之FrameLayout,从名字看SizeMonitoringFrameLayout，监控大小变化的FrameLayout,可以理解Rn中在一个Activity上各种View的变化，页面的切换，都要依赖这个灵活的SizeMonitoringFrameLayout
因为ReactRootView中方法众多，我们先放下不表。回头看 mReactRootView.startReactApplication的实现。

>ReactRootView.java

```
  public void startReactApplication(
      ReactInstanceManager reactInstanceManager,
      String moduleName,
      @Nullable Bundle launchOptions) {
    UiThreadUtil.assertOnUiThread();

    // TODO(6788889): Use POJO instead of bundle here, apparently we can't just use WritableMap
    // here as it may be deallocated in native after passing via JNI bridge, but we want to reuse
    // it in the case of re-creating the catalyst instance
    Assertions.assertCondition(
        mReactInstanceManager == null,
        "This root view has already been attached to a catalyst instance manager");

    mReactInstanceManager = reactInstanceManager;
    mJSModuleName = moduleName;
    mLaunchOptions = launchOptions;

    if (!mReactInstanceManager.hasStartedCreatingInitialContext()) {
      mReactInstanceManager.createReactContextInBackground();
    }

    // We need to wait for the initial onMeasure, if this view has not yet been measured, we set which
    // will make this view startReactApplication itself to instance manager once onMeasure is called.
    if (mWasMeasured) {
      attachToReactInstanceManager();
    }
  }
```
这个方法三个参数：
- ReactInstanceManager：首先这是一个接口，用来管理CatalystInstance（提供Java与Js互通的环境，创建Java模块注册表及Javascript模块注册表，并遍历实例化模块，最后通过ReactBridge将Js Bundle传送到Js引擎）,将activity的生命周期变化，传递到RootView,还提供了一个配置Builder，从最后build方法可以看到ReactInstanceManager实现类是XReactInstanceManagerImpl，这里有个规律，Rn中的Manager的管理类都是接口，实现类都是xxxImpl，在RN这样平均2周就发一个版本的开源项目中面向接口编程可以最大限度的降低修改带来的影响。
- moduleName: 是和JS端约定的一个string类型的key,js端通过AppRegistry.registerComponent('xxx', () => Root)，Android端复写getMainComponentName（）return 'xxx'；方法实现
-launchOptions:startActivity的时候可以传入一些参数给JS，用于初始化数据，这个方法在RN的文档中还没有添加，我也是看了源码才发现的，有了这个方法就再也不用到了JS页面再调原生方法获取初始化数据了(这个方法也是蠢萌蠢萌的).

startReactApplication方法根据传入的三个参数赋值本地变量后，执行了 mReactInstanceManager.createReactContextInBackground()方法，上面说到mReactInstanceManager接口的实现是XReactInstanceManagerImpl，所以到XReactInstanceManagerImpl中找到createReactContextInBackground方法。
>XReactInstanceManagerImpl.java

```
  @Override
  public void createReactContextInBackground() {
    Assertions.assertCondition(
        !mHasStartedCreatingInitialContext,
        "createReactContextInBackground should only be called when creating the react " +
            "application for the first time. When reloading JS, e.g. from a new file, explicitly" +
            "use recreateReactContextInBackground");

    mHasStartedCreatingInitialContext = true;
    recreateReactContextInBackgroundInner();
  }
```
#### ReactContextInitAsyncTask

方法的注释已经说明了，这里用异步task来初始化ReactContext，再看recreateReactContextInBackgroundInner方法

```
private void recreateReactContextInBackgroundInner() {
    UiThreadUtil.assertOnUiThread();

    if (mUseDeveloperSupport && mJSMainModuleName != null) {
      final DeveloperSettings devSettings = mDevSupportManager.getDevSettings();

      // If remote JS debugging is enabled, load from dev server.
     //这里我去掉了本地debug模式的代码，只关注线上加载bundle的路线

    recreateReactContextInBackgroundFromBundleLoader();
  }
```
继续往下走
```
 private void recreateReactContextInBackgroundFromBundleLoader() {
    recreateReactContextInBackground(
        new JSCJavaScriptExecutor.Factory(mJSCConfig.getConfigMap()),
        mBundleLoader);
  }

```
这里传入俩个参数，一个的JavaScriptExecutor， protected JSCConfig mJSCConfig = JSCConfig.EMPTY; mJSCConfig默认使用JSCConfig.EMPTY类型，那么mJSCConfig.getConfigMap()，也就是返回个WritableNativeMap。

>JSCConfig.java

```
JSCConfig EMPTY = new JSCConfig() {
    @Override
    public WritableNativeMap getConfigMap() {
      return new WritableNativeMap();
    }
  };
```

```
 private void recreateReactContextInBackground(
      JavaScriptExecutor.Factory jsExecutorFactory,
      JSBundleLoader jsBundleLoader) {
    UiThreadUtil.assertOnUiThread();

    ReactContextInitParams initParams =
        new ReactContextInitParams(jsExecutorFactory, jsBundleLoader);
    if (mReactContextInitAsyncTask == null) {
      // No background task to create react context is currently running, create and execute one.
      mReactContextInitAsyncTask = new ReactContextInitAsyncTask();
      mReactContextInitAsyncTask.executeOnExecutor(AsyncTask.THREAD_POOL_EXECUTOR, initParams);
    } else {
      // Background task is currently running, queue up most recent init params to recreate context
      // once task completes.
      mPendingReactContextInitParams = initParams;
    }
  }
```

关键的地方来了，recreateReactContextInBackground方法中，创建了一个ReactContextInitAsyncTask,传入了new ReactContextInitParams(jsExecutorFactory, jsBundleLoader)，那我们就详细看看这个task做了什么大事

>XReactInstanceManagerImpl.java->ReactContextInitAsyncTask

```
  private final class ReactContextInitAsyncTask extends
      AsyncTask<ReactContextInitParams, Void, Result<ReactApplicationContext>> {
    @Override
    protected void onPreExecute() {
      if (mCurrentReactContext != null) {
    	//destroy之前的reactContext	
        tearDownReactContext(mCurrentReactContext);
        mCurrentReactContext = null;
      }
    }

    @Override
    protected Result<ReactApplicationContext> doInBackground(ReactContextInitParams... params) {

      Process.setThreadPriority(Process.THREAD_PRIORITY_DEFAULT);
      Assertions.assertCondition(params != null && params.length > 0 && params[0] != null);
      try {
        JavaScriptExecutor jsExecutor = params[0].getJsExecutorFactory().create();
        return Result.of(createReactContext(jsExecutor, params[0].getJsBundleLoader()));
      } catch (Exception e) {
        // Pass exception to onPostExecute() so it can be handled on the main thread
        return Result.of(e);
      }
    }

    @Override
    protected void onPostExecute(Result<ReactApplicationContext> result) {
      try {
        setupReactContext(result.get());
      } catch (Exception e) {
        mDevSupportManager.handleException(e);
      } finally {
        mReactContextInitAsyncTask = null;
      }

      // Handle enqueued request to re-initialize react context.
      if (mPendingReactContextInitParams != null) {
        recreateReactContextInBackground(
            mPendingReactContextInitParams.getJsExecutorFactory(),
            mPendingReactContextInitParams.getJsBundleLoader());
        mPendingReactContextInitParams = null;
      }
    }

  }
```
#### 核心方法createReactContext
先看doInBackground方法的createReactContext(jsExecutor, params[0].getJsBundleLoader()),这个方法有100行代码，很吓人，需要耐心的一行一行看
```
 NativeModuleRegistry.Builder nativeRegistryBuilder = new NativeModuleRegistry.Builder();
    JavaScriptModuleRegistry.Builder jsModulesBuilder = new JavaScriptModuleRegistry.Builder();
```
nativeRegistryBuilder是java暴露给js调用的api集合,jsModulesBuilder是js暴露给java调用的api集合，下面的方法就是在生成这俩个集合的数据

```
 try {
      CoreModulesPackage coreModulesPackage =
          new CoreModulesPackage(this, mBackBtnHandler, mUIImplementationProvider);
      processPackage(coreModulesPackage, reactContext, nativeRegistryBuilder, jsModulesBuilder);
    } finally {
      Systrace.endSection(TRACE_TAG_REACT_JAVA_BRIDGE);
    }
```
这个方法就是先把系统自带的modeles添加到nativeRegistryBuilder和jsModulesBuilder,系统自带的java modules有AnimationsDebugModule，AndroidInfoModule，ExceptionsManagerModule，Timing，SourceCodeModule，UIManagerModule，DebugComponentOwnershipModule，JSCHeapCapture，JSCSamplingProfiler,其中UIManagerModule在回到jsx如何渲染成原生控件的时候还要详细解析。

#### LazyReactPackage 懒加载

我们还注意到class CoreModulesPackage extends LazyReactPackage，0.33版本的release notes中有这样一句

	Add LazyReactPackage (1feb462) - @AaaChiuuu

支持懒加载reactPackage，我们就顺路看一下如何实现了懒加载。

LazyReactPackage还是继承至ReactPackage，在实现createNativeModules的时候他巧妙的调用了自己的抽象方法getNativeModules，并且TODO中说 Make this default and deprecate ReactPackage，看来这是为了后期就把ReactPackage废弃了，全部换成LazyReactPackage,那到底懒加载是如何做到的尼？看CoreModulesPackage的getNativeModules方法

>CoreModulesPackage.java

```
 @Override
  public List<ModuleSpec> getNativeModules(final ReactApplicationContext reactContext) {
    List<ModuleSpec> moduleSpecList = new ArrayList<>();
    moduleSpecList.add(
      new ModuleSpec(AnimationsDebugModule.class, new Provider<NativeModule>() {
        @Override
        public NativeModule get() {
          return new AnimationsDebugModule(
            reactContext,
            mReactInstanceManager.getDevSupportManager().getDevSettings());
        }
      }));
    .....
    }
```
在构造ModuleSpec的时候传入了AnimationsDebugModule.class和一个provider的get方法返回了一个new AnimationsDebugModule，这里好像没有什么特别的，怎么能实现懒加载尼，再细想一下懒加载什么意思

		A specification for a native module. This exists so that we don't have to pay the cost for creation until/if the module is used.
        提供了说明书给native module，当module使用过之后，我们就不需要再次创建这个module。
然后我们再下点功夫把0.33之前的版本的CoreModulesPackage代码扒拉出来对比一下


```
  @Override
  public List<NativeModule> createNativeModules(
      ReactApplicationContext catalystApplicationContext) {
    Systrace.beginSection(Systrace.TRACE_TAG_REACT_JAVA_BRIDGE, "createUIManagerModule");
    UIManagerModule uiManagerModule;
    try {
      List<ViewManager> viewManagersList = mReactInstanceManager.createAllViewManagers(
          catalystApplicationContext);
      uiManagerModule = new UIManagerModule(
          catalystApplicationContext,
          viewManagersList,
          mUIImplementationProvider.createUIImplementation(
              catalystApplicationContext,
              viewManagersList));
    } finally {
      Systrace.endSection(Systrace.TRACE_TAG_REACT_JAVA_BRIDGE);
    }

    return Arrays.<NativeModule>asList(
        new AnimationsDebugModule(
            catalystApplicationContext,
            mReactInstanceManager.getDevSupportManager().getDevSettings()),
        new AndroidInfoModule(),
        new DeviceEventManagerModule(catalystApplicationContext, mHardwareBackBtnHandler),
        new ExceptionsManagerModule(mReactInstanceManager.getDevSupportManager()),
        new Timing(catalystApplicationContext, mReactInstanceManager.getDevSupportManager()),
        new SourceCodeModule(mReactInstanceManager.getSourceUrl()),
        uiManagerModule,
        new JSCHeapCapture(catalystApplicationContext),
        new DebugComponentOwnershipModule(catalystApplicationContext));
  }
```

这下我们就目标了，所谓了懒加载就是延时加载，之前的方案是，在调用createNativeModules的时候，将所有的NativeModule都实例化传过去，这样势必有很大的内存开销，这也就是很多大厂对RN有顾虑的原因之一，现在有了懒加载，这里的list里面放的是NativeModule的构造方式，什么时候用到的时候调用get方法，才会真正的去实例化。

懒加载的方式确实可以减少很多开销，因为有些modules可能我们更不用不到。



* * *
### 上面理解错误，打脸

这里误人子弟了，对LazyReactPackage理解错误，后来发现他并不是等到使用的时候，才去new module，而是放在原来创建processPackage-》reactPackage.createNativeModules(reactContext)的时候
```
 @Override
  public final List<NativeModule> createNativeModules(ReactApplicationContext reactContext) {
    List<NativeModule> modules = new ArrayList<>();
    for (ModuleSpec holder : getNativeModules(reactContext)) {
      SystraceMessage.beginSection(TRACE_TAG_REACT_JAVA_BRIDGE, "createNativeModule")
        .arg("module", holder.getType())
        .flush();
      try {
        modules.add(holder.getProvider().get());
      } finally {
        Systrace.endSection(TRACE_TAG_REACT_JAVA_BRIDGE);
      }
    }
    return modules;
  }
```
已经创建完成，上面的解析有误，请大家谅解。

对比老版本的RN，发现这里的lazy create是指，把原来createNativeModules 直接在返回list的时候创建所有的module，改为了getNativeModules + for 循环创建 俩步执行，这也是粗略的理解，如果有误，欢迎打脸。

* * *

再回到createReactContext方法

>XReactInstanceManagerImpl.java

```
for (ReactPackage reactPackage : mPackages) {
      Systrace.beginSection(
          TRACE_TAG_REACT_JAVA_BRIDGE,
          "createAndProcessCustomReactPackage");
      try {
        processPackage(reactPackage, reactContext, nativeRegistryBuilder, jsModulesBuilder);
      } finally {
        Systrace.endSection(TRACE_TAG_REACT_JAVA_BRIDGE);
      }
    }

```

加载完系统的module再加载我们自定义的module，简单的demo请看[官方文档](http://facebook.github.io/react-native/docs/native-modules-android.html)

```
 NativeModuleRegistry nativeModuleRegistry;
    try {
       nativeModuleRegistry = nativeRegistryBuilder.build();
    } finally {
      Systrace.endSection(TRACE_TAG_REACT_JAVA_BRIDGE);
      ReactMarker.logMarker(BUILD_NATIVE_MODULE_REGISTRY_END);
    }
```
这里很奇怪，nativeModuleRegistry.build() 又来了一次，回去看代码才知道上面用的是NativeModuleRegistry.Builder nativeRegistryBuilder = new NativeModuleRegistry.Builder();用NativeModuleRegistry.Builder 内部类维护了一个HashMap<String, NativeModule> hashmap,和这里的builder.build并不冲突

>NativeModuleRegistry.java

```
 public NativeModuleRegistry build() {
      Map<Class<NativeModule>, NativeModule> moduleInstances = new HashMap<>();
      for (NativeModule module : mModules.values()) {
        moduleInstances.put((Class<NativeModule>)module.getClass(), module);
      }
      return new NativeModuleRegistry(moduleInstances);
    }
  }
```
又转变成一个class和NativeModule的Map，作为构造参数new了一个NativeModuleRegistry，其中还有个ArrayList<OnBatchCompleteListener> mBatchCompleteListenerModules全局变量，用于js call java结束的时候回掉。
#### CatalystInstanceImpl 登场
NativeModuleRegistry到这里算也是准备就绪了，那么下面就是另一重量级的类要出场了

>XReactInstanceManagerImpl.java

```
CatalystInstanceImpl.Builder catalystInstanceBuilder = new CatalystInstanceImpl.Builder()
     .setReactQueueConfigurationSpec(ReactQueueConfigurationSpec.createDefault())
        .setJSExecutor(jsExecutor)
        .setRegistry(nativeModuleRegistry)
        .setJSModuleRegistry(jsModulesBuilder.build())
        .setJSBundleLoader(jsBundleLoader)
        .setNativeModuleCallExceptionHandler(exceptionHandler);

```
catalystInstanceBuilder 这里又是个builder，那下面肯定有.build()方法

```
 final CatalystInstance catalystInstance;
    try {
      catalystInstance = catalystInstanceBuilder.build();
    } finally {
      Systrace.endSection(TRACE_TAG_REACT_JAVA_BRIDGE);
      ReactMarker.logMarker(CREATE_CATALYST_INSTANCE_END);
    }

 public CatalystInstanceImpl build() {
      return new CatalystInstanceImpl(
          Assertions.assertNotNull(mReactQueueConfigurationSpec),
          Assertions.assertNotNull(mJSExecutor),
          Assertions.assertNotNull(mRegistry),
          Assertions.assertNotNull(mJSModuleRegistry),
          Assertions.assertNotNull(mJSBundleLoader),
          Assertions.assertNotNull(mNativeModuleCallExceptionHandler));
    }
  }

```

果然CatalystInstance 出现之后，我们就把视线转移到CatalystInstanceImpl的构造方法中

	乱花渐欲迷人眼，错综复杂的类，很容易让初学者放弃治疗，坚持再坚持之后，才能达到另一个境界
如果你还没有放弃，那么接着往下看。

>CatalystInstanceImpl

```
 private CatalystInstanceImpl(
      final ReactQueueConfigurationSpec ReactQueueConfigurationSpec,
      final JavaScriptExecutor jsExecutor,
      final NativeModuleRegistry registry,
      final JavaScriptModuleRegistry jsModuleRegistry,
      final JSBundleLoader jsBundleLoader,
      NativeModuleCallExceptionHandler nativeModuleCallExceptionHandler) {
    FLog.d(ReactConstants.TAG, "Initializing React Xplat Bridge.");
  	//C的方法，初始化了mHybridData
    mHybridData = initHybrid();
	//俩个轮循的配置
    mReactQueueConfiguration = ReactQueueConfigurationImpl.create(
        ReactQueueConfigurationSpec,
        new NativeExceptionHandler());
    mBridgeIdleListeners = new CopyOnWriteArrayList<>();
    mJavaRegistry = registry;
    mJSModuleRegistry = jsModuleRegistry;
    mJSBundleLoader = jsBundleLoader;
    mNativeModuleCallExceptionHandler = nativeModuleCallExceptionHandler;
    mTraceListener = new JSProfilerTraceListener(this);

    initializeBridge(
      new BridgeCallback(this),
      jsExecutor,
      mReactQueueConfiguration.getJSQueueThread(),
      mReactQueueConfiguration.getNativeModulesQueueThread(),
      mJavaRegistry.getModuleRegistryHolder(this));
    mMainExecutorToken = getMainExecutorToken();
  }
```
看到initializeBridge方法，说明这条路快走到底了，突然发现initializeBridge是个C的方法，看来真的到底了，这和老版本在initializeBridge方法上的处理不太相同。那就去C代码里面找，果然在react-native/ReactAndroid/src/main/jni/xreact/jni/CatalystInstanceImpl.cpp 中找到了

```
void CatalystInstanceImpl::initializeBridge(
    jni::alias_ref<ReactCallback::javaobject> callback,
    // This executor is actually a factory holder.
    JavaScriptExecutorHolder* jseh,
    jni::alias_ref<JavaMessageQueueThread::javaobject> jsQueue,
    jni::alias_ref<JavaMessageQueueThread::javaobject> moduleQueue,
    ModuleRegistryHolder* mrh) {
  instance_->initializeBridge(folly::make_unique<JInstanceCallback>(callback),
                              jseh->getExecutorFactory(),
                              folly::make_unique<JMessageQueueThread>(jsQueue),
                              folly::make_unique<JMessageQueueThread>(moduleQueue),
                              mrh->getModuleRegistry());
}
```
的实现,因为我们的NativeModule都维护在ModuleRegistryHolder中，所以我们先看 mrh->getModuleRegistry()，找到ModuleRegistryHolder.cpp-> 可是这里并没有getModuleRegistry()方法，有点奇怪，再回到原生方法mJavaRegistry.getModuleRegistryHolder(this)

```
ModuleRegistryHolder getModuleRegistryHolder(
      CatalystInstanceImpl catalystInstanceImpl) {
    ArrayList<JavaModuleWrapper> javaModules = new ArrayList<>();
    ArrayList<CxxModuleWrapper> cxxModules = new ArrayList<>();
    for (NativeModule module : mModuleInstances.values()) {
      if (module instanceof BaseJavaModule) {
        javaModules.add(new JavaModuleWrapper(catalystInstanceImpl, (BaseJavaModule) module));
      } else if (module instanceof CxxModuleWrapper) {
        cxxModules.add((CxxModuleWrapper) module);
      } else {
        throw new IllegalArgumentException("Unknown module type " + module.getClass());
      }
    }
    return new ModuleRegistryHolder(catalystInstanceImpl, javaModules, cxxModules);
  }

public class ModuleRegistryHolder {
  private final HybridData mHybridData;
  private static native HybridData initHybrid(
    CatalystInstanceImpl catalystInstanceImpl,
    Collection<JavaModuleWrapper> javaModules,
    Collection<CxxModuleWrapper> cxxModules);

  public ModuleRegistryHolder(CatalystInstanceImpl catalystInstanceImpl,
                              Collection<JavaModuleWrapper> javaModules,
                              Collection<CxxModuleWrapper> cxxModules) {
    mHybridData = initHybrid(catalystInstanceImpl, javaModules, cxxModules);
  }
}
```
在ModuleRegistryHolder 的构造方法中又执行了C的initHybrid 方法，难道是在这个时候，把配置表注册进去的？
找到react/react-native/ReactAndroid/src/main/jni/xreact/jni/CxxModuleWrapper.cpp 
果然发现了类似生成配置表的方法

老的源码中在java中生成
```
 ReactBridge bridge ;
    try {
        bridge = new ReactBridge(jsExecutor, new CatalystInstanceImpl.NativeModulesReactCallback( null), this.mReactQueueConfiguration.getNativeModulesQueueThread()) ;
        this.mMainExecutorToken = bridge.getMainExecutorToken() ;
    } finally {
        Systrace.endSection(0L );
    }

    Systrace.beginSection(0L , "setBatchedBridgeConfig");

    try {
        bridge.setGlobalVariable("__fbBatchedBridgeConfig" , this.buildModulesConfigJSONProperty( this.mJavaRegistry, jsModulesConfig));
        bridge.setGlobalVariable( "__RCTProfileIsProfiling" , Systrace.isTracing( 0L)?"true" :"false") ;
    } finally {
        Systrace.endSection(0L );
    }
```
现在换到了C中，只需提供initializeBridge 参数即可
```
void CxxModuleWrapper::registerNatives() {
  registerHybrid({
    makeNativeMethod("initHybrid", CxxModuleWrapper::initHybrid),
    makeNativeMethod("getName", CxxModuleWrapper::getName),
    makeNativeMethod("getConstantsJson", CxxModuleWrapper::getConstantsJson),
    makeNativeMethod("getMethods", "()Ljava/util/Map;", CxxModuleWrapper::getMethods),
  });

  CxxMethodWrapper::registerNatives();
}

std::string CxxModuleWrapper::getConstantsJson() {
  std::map<std::string, folly::dynamic> constants = module_->getConstants();
  folly::dynamic constsobject = folly::dynamic::object;

  for (auto& c : constants) {
    constsobject.insert(std::move(c.first), std::move(c.second));
  }

  return facebook::react::detail::toStdString(folly::toJson(constsobject));
}

```
再往下的代码就不贴了，到此为止NativeModule的JSON配置表或者叫映射关系表，在C这个中间层就生成完毕了。

#### JAVA部分的最后一步
扯的太远了，再回到createReactContext方法
reactContext.initializeWithInstance(catalystInstance);
 catalystInstance.runJSBundle();
 
 这里不多解释，调用native void loadScriptFromAssets(AssetManager assetManager, String assetURL)加载解析Jsbundle
 
 解析完毕，就到了ReactContextInitAsyncTask的onPostExecute方法
 
 ```
   @Override
    protected void onPostExecute(Result<ReactApplicationContext> result) {
      try {
        setupReactContext(result.get());
      } catch (Exception e) {
        mDevSupportManager.handleException(e);
      } finally {
        mReactContextInitAsyncTask = null;
      }

    }


private void setupReactContext(ReactApplicationContext reactContext) {
    Systrace.beginSection(TRACE_TAG_REACT_JAVA_BRIDGE, "setupReactContext");
    UiThreadUtil.assertOnUiThread();
    Assertions.assertCondition(mCurrentReactContext == null);
    mCurrentReactContext = Assertions.assertNotNull(reactContext);
    CatalystInstance catalystInstance =
        Assertions.assertNotNull(reactContext.getCatalystInstance());
	//创建注册表
    catalystInstance.initialize();
    mDevSupportManager.onNewReactContextCreated(reactContext);
    mMemoryPressureRouter.addMemoryPressureListener(catalystInstance);
  	//同步生命周期
    moveReactContextToCurrentLifecycleState();

    for (ReactRootView rootView : mAttachedRootViews) {
      attachMeasuredRootViewToInstance(rootView, catalystInstance);
    }

    ReactInstanceEventListener[] listeners =
      new ReactInstanceEventListener[mReactInstanceEventListeners.size()];
    listeners = mReactInstanceEventListeners.toArray(listeners);

    for (ReactInstanceEventListener listener : listeners) {
      listener.onReactContextInitialized(reactContext);
    }
    Systrace.endSection(TRACE_TAG_REACT_JAVA_BRIDGE);
  }
 ```
我们注意到mAttachedRootViews是一个list，这和我们之前版本的单Activity好像有点违背，那我们看mAttachedRootViews是什么时候addView的
```
@Override
  public void attachMeasuredRootView(ReactRootView rootView) {
    UiThreadUtil.assertOnUiThread();
    mAttachedRootViews.add(rootView);

    // If react context is being created in the background, JS application will be started
    // automatically when creation completes, as root view is part of the attached root view list.
    if (mReactContextInitAsyncTask == null && mCurrentReactContext != null) {
      attachMeasuredRootViewToInstance(rootView, mCurrentReactContext.getCatalystInstance());
    }
  }
```
那attachMeasuredRootView 什么时候调用的，一番折腾之后找到了

```
 mReactRootView.startReactApplication(
      getReactNativeHost().getReactInstanceManager(),
      getMainComponentName(),
      getLaunchOptions());


 public void startReactApplication(
      ReactInstanceManager reactInstanceManager,
      String moduleName,
      @Nullable Bundle launchOptions) {
  
    // We need to wait for the initial onMeasure, if this view has not yet been measured, we set which
    // will make this view startReactApplication itself to instance manager once onMeasure is called.
    if (mWasMeasured) {
      attachToReactInstanceManager();
    }
  }

 private void attachToReactInstanceManager() {
    if (mIsAttachedToInstance) {
      return;
    }

    mIsAttachedToInstance = true;
    Assertions.assertNotNull(mReactInstanceManager).attachMeasuredRootView(this);
    getViewTreeObserver().addOnGlobalLayoutListener(getKeyboardListener());
  }

```
只有当没有测量的时候才会把view add 到list中，一个activity只会add一次，那么多个那attachMeasuredRootView是否意味这可以使用多个RNactivity了？因为我知道之前这样使用会出现生命周期的问题，因为ReactInstanceManager是一个单例，这个还需要验证一下。

再回来

>XReactInstanceManagerImpl.java
```
 private void attachMeasuredRootViewToInstance(
      ReactRootView rootView,
      CatalystInstance catalystInstance) {
    Systrace.beginSection(TRACE_TAG_REACT_JAVA_BRIDGE, "attachMeasuredRootViewToInstance");
    UiThreadUtil.assertOnUiThread();

    // Reset view content as it's going to be populated by the application content from JS
    rootView.removeAllViews();
    rootView.setId(View.NO_ID);

    UIManagerModule uiManagerModule = catalystInstance.getNativeModule(UIManagerModule.class);
    int rootTag = uiManagerModule.addMeasuredRootView(rootView);
    rootView.setRootViewTag(rootTag);
    @Nullable Bundle launchOptions = rootView.getLaunchOptions();
    WritableMap initialProps = Arguments.makeNativeMap(launchOptions);
    String jsAppModuleName = rootView.getJSModuleName();

    WritableNativeMap appParams = new WritableNativeMap();
    appParams.putDouble("rootTag", rootTag);
    appParams.putMap("initialProps", initialProps);
    catalystInstance.getJSModule(AppRegistry.class).runApplication(jsAppModuleName, appParams);
    Systrace.endSection(TRACE_TAG_REACT_JAVA_BRIDGE);
  }
```

这个就是正在启动Rn的入口了，通过UIManagerModule告知JS这个RootView是这边的根布局，标记tag为rootTag，catalystInstance.getJSModule(AppRegistry.class).runApplication(jsAppModuleName, appParams);调用AppRegistry.runApplication，AppRegistry也就是JS暴露给Java的方法，通过runApplication将要传递的数据和要启动的appModuleName传递过去。

对应JS那边的方法
index.android.js
```
'use strict';

import React from 'react';
import Root from './root';
import {AppRegistry} from 'react-native';

AppRegistry.registerComponent('CrowdReactApp', () => Root);
```
到此为止从 mReactRootView.startReactApplication 到catalystInstance.getJSModule(AppRegistry.class).runApplication(jsAppModuleName, appParams) 一个JS页面就启动完毕了。

解析过程只是粗略的走了一遍难免漏掉了很多细节，有兴趣的同学，可以具体问题具体分析。

下面还有几个问题我会再去研究：
- [Java和JS的通讯原理](https://zhuanlan.zhihu.com/p/20464825)--大头鬼的这篇博客写的很好，我就不造次了
- [JSX如何渲染成原生页面](http://xujinyang.github.io/2016/09/12/React-Native-%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90%E4%BA%8C-JSX%E5%A6%82%E4%BD%95%E6%B8%B2%E6%9F%93%E6%88%90%E5%8E%9F%E7%94%9F%E9%A1%B5%E9%9D%A2(%E4%B8%8A)/)

 





























