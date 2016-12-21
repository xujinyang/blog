---
title: >-
  解决React-Native bug:about DialogModule onHostResume
date: 2016-09-13 18:45:49
tags: react-native
---

问题是这样的，线上的Bugly爆出这样一个错误，而且延续了好多个版本，一直没有解决，崩溃次数已经上千次，因为刚看过RN源码所以斗胆尝试解决一下。

> Attached DialogModule to host with pending alert but no FragmentManager (not attached to an Activity).
com.facebook.infer.annotation.Assertions.java.lang.Object assertNotNull

下面是问题的解决过程：
### google
首先想到的是把这个报错扔到google里面看看，找到了有类似的错误，但是没有看到好的解决方案，比如这个[issue](https://github.com/facebook/react-native/issues/9018)这个哥们应该是中国的，还没有人回应。还有找到的stackoverflow[上一个问题](http://stackoverflow.com/questions/35254232/react-native-maps-integration-issue-with-react-native-0-19-0)有一个回答：

	What does the rest of your activity look like? I ran into this but the problem was that I was not implementing DefaultHardwareBackBtnHandler in my activity.

因为某种原因，我们的RN版本一直使用的是0.23，也就是自己实现的reactActivity，DefaultHardwareBackBtnHandler我们也是实现了的，这个回答也被排除了。

那么网上找不到问题的解决方案也就没辙了，实际上上一次我尝试解决也是这样放弃的。

### 自己动手丰衣足食

没办法崩溃的人越来越多，逼着要尽快解决,其实也挺快的。

首先我找到了DialogModule.java，然后顺利的找到了报错的文本信息

```
public class DialogModule extends ReactContextBaseJavaModule implements LifecycleEventListener {

  private class FragmentManagerHelper {

  @Override
  public void onHostResume() {
    mIsInForeground = true;
    // Check if a dialog has been created while the host was paused, so that we can show it now.
    FragmentManagerHelper fragmentManagerHelper = getFragmentManagerHelper();
    Assertions.assertNotNull(
        fragmentManagerHelper,
        "Attached DialogModule to host with pending alert but no FragmentManager " +
        "(not attached to an Activity).");
    fragmentManagerHelper.showPendingAlert();
  }

}

```
报错信息完全对上了，意味着就是这个地方崩溃了的，那么接着就开始看onHostResume 方法在什么地方调用的。简单看了一下DialogModule实现了LifecycleEventListener 接口
```
public interface LifecycleEventListener {

  /**
   * Called when host (activity/service) receives resume event (e.g. {@link Activity#onResume}
   */
  void onHostResume();

  /**
   * Called when host (activity/service) receives pause event (e.g. {@link Activity#onPause}
   */
  void onHostPause();

  /**
   * Called when host (activity/service) receives destroy event (e.g. {@link Activity#onDestroy}
   */
  void onHostDestroy();

}
```
onHostResume 是生命周期中的一环，而且又是个接口，那么我开始怀疑Activity在执行onResume和他有某种关系。到BaseRNActivity.java(不用找了，这是我们自己写的一个类，这是Rn老版本的写法，现在直接使用ReactActivity就可以)

```
 @Override
    protected void onResume() {
        super.onResume();
        if (mReactInstanceManager != null) {
            mReactInstanceManager.onHostResume(this, this);
        }
    }
```
因为之前分析过RN的源码，所以很清楚的知道，mReactInstanceManager的实现都在ReactInstanceManagerImpl.java中， so 直接找到onHostResume

```
  @Override
  public void onHostResume(Activity activity, DefaultHardwareBackBtnHandler defaultBackButtonImpl) {
    UiThreadUtil.assertOnUiThread();

    mDefaultBackButtonImpl = defaultBackButtonImpl;
    if (mUseDeveloperSupport) {
      mDevSupportManager.setDevSupportEnabled(true);
    }

    mCurrentActivity = activity;
    moveToResumedLifecycleState(false);
  }
```
代码很简单，判断是不是在主线程，设置DevSupport，执行moveToResumedLifecycleState

```
  private void moveToResumedLifecycleState(boolean force) {
    if (mCurrentReactContext != null) {
      // we currently don't have an onCreate callback so we call onResume for both transitions
      if (force ||
          mLifecycleState == LifecycleState.BEFORE_RESUME ||
          mLifecycleState == LifecycleState.BEFORE_CREATE) {
        mCurrentReactContext.onHostResume(mCurrentActivity);
      }
    }
    mLifecycleState = LifecycleState.RESUMED;
  }
```
继续进onHostResume
```
public void onHostResume(@Nullable Activity activity) {
    UiThreadUtil.assertOnUiThread();
    mCurrentActivity = new WeakReference(activity);
    for (LifecycleEventListener listener : mLifecycleEventListeners) {
      listener.onHostResume();
    }
  }
```

这里会然一笑，果然有个循环在执行这个LifecycleEventListener的onHostResume，到这里就不用再往下走了，我们已经确定了是在activity执行resume的时候，调用了DialogModule的onHostResume方法，这个时候fragmentManagerHelper为空照成了空指针错误。

那fragmentManagerHelper 什么时候为空尼？？再回到DialogModle的onHostResume方法进去到
getFragmentManagerHelper

```
private @Nullable FragmentManagerHelper getFragmentManagerHelper() {
    Activity activity = getCurrentActivity();
    if (activity == null) {
      return null;
    }
    if (activity instanceof FragmentActivity) {
      return new FragmentManagerHelper(((FragmentActivity) activity).getSupportFragmentManager());
    } else {
      return new FragmentManagerHelper(activity.getFragmentManager());
    }
  }
```
从代码可以看出，只有当activity为空的时候，才会出现返回null，那activity什么时候会null，再进去

```
private @Nullable WeakReference<Activity> mCurrentActivity;

Activity getCurrentActivity() {
    if (mCurrentActivity == null) {
      return null;
    }
    return mCurrentActivity.get();
  }
```
原来mCurrentActivity是个WeakReference 弱引用，那么当系统垃圾回收的时候，就有可能为把它干掉了。

接下来开始思考，什么情况下，会出现调用Activity的onResume的时候，WeakReference会为空，下面全是经验之谈了，要造回收的场景，首选就是打开开发者模式的->不保留活动，不保留活动是意思是:用户离开后既销毁每个活动，

离开页面有三种情况：
1. 按back键
2. 按home 
3. 切换到其他应用再切回来

测试了一下，第一种情况，会正常的处理back逻辑，没有崩溃，第二第三种场景都成功复现了这个bug

喜大普奔，**复现了bug意味着bug解决了一半**

### 解决问题

解决问题很简单，只要在fragmentManagerHelper使用前判空就可以，但是DialogModule是系统自带的，要想修复这个问题，还需要自己写个DialogModule，有点太重了，因为我们使用的RN版本很老了，我就想看看最新的版本也没有解决这个问题，升级RN 版本到0.33

```
 @Override
  public void onHostResume() {
    mIsInForeground = true;
    // Check if a dialog has been created while the host was paused, so that we can show it now.
    FragmentManagerHelper fragmentManagerHelper = getFragmentManagerHelper();
    if (fragmentManagerHelper != null) {
      fragmentManagerHelper.showPendingAlert();
    } else {
      FLog.w(DialogModule.class, "onHostResume called but no FragmentManager found");
    }
  }
```

果然，已经修复了 ，那我们只要升级Rn版本就可以了。react-native 更新的非常快，很多bug都可以通过升级版本来解决，当然这是在碰运气，你去看看RN的issues已经超过了1000,你遇到的bug，能不能在下个版本修复，上天保佑吧。

通过这个bug的探索过程，我发现很多问题都是有迹可循的，只要我们耐心的分析，沉下心看源码。

希望这个bolg能帮助到遇到这个坑的同学。


