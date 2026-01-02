Android应用的页面启动分为应用进程和系统进程，本篇会从应用进程讲起，我会把涉及的源码全部贴出来，并以注释的形式注明作用。

首先当我们想要进行页面跳转的时候，通常会创建一个Intent对象，利用 `Intent(Context packageContext, Class<?> cls) `构造函数创建一个intent的对象，第一个参数放当前Activity的对象，第二个参数放目标页面的class对象。然后再调用系统函数`startActivity`就可以完成跳转了。
```java
Intent intent = new Intent(MainActivity.this, TestActivity.class);
startActivity(intent);
```
```java
public void startActivity(Intent intent) {
    this.startActivity(intent, null);
}
```
```java
public void startActivity(Intent intent, @Nullable Bundle options) {
    //自动填充相关的逻辑，本次案例不会触发
    if (mIntent != null && mIntent.hasExtra(AutofillManager.EXTRA_RESTORE_SESSION_TOKEN)
            && mIntent.hasExtra(AutofillManager.EXTRA_RESTORE_CROSS_ACTIVITY)) {
        if (TextUtils.equals(getPackageName(),
                intent.resolveActivity(getPackageManager()).getPackageName())) {
            // Apply Autofill restore mechanism on the started activity by startActivity()
            final IBinder token =
                    mIntent.getIBinderExtra(AutofillManager.EXTRA_RESTORE_SESSION_TOKEN);
            // Remove restore ability from current activity
            mIntent.removeExtra(AutofillManager.EXTRA_RESTORE_SESSION_TOKEN);
            mIntent.removeExtra(AutofillManager.EXTRA_RESTORE_CROSS_ACTIVITY);
            // Put restore token
            intent.putExtra(AutofillManager.EXTRA_RESTORE_SESSION_TOKEN, token);
            intent.putExtra(AutofillManager.EXTRA_RESTORE_CROSS_ACTIVITY, true);
        }
    }
    if (options != null) {
        startActivityForResult(intent, -1, options);
    } else {
        // Note we want to go through this call for compatibility with
        // applications that may have overridden the method.
        startActivityForResult(intent, -1);
    }
}
```
这里涉及到了一个自动填充的恢复逻辑，我们本次的示例并不会触发，所以无需分心，直接去看下一个判断。这里涉及到了一个options字段。这个options字段是从前面传递过来的，通过回溯我们可以得知该值为null，所以会走入else逻辑，调用两个参数的`startActivityForResult`函数。
```java
public void startActivityForResult(@RequiresPermission Intent intent, int requestCode) {
    startActivityForResult(intent, requestCode, null);
}
```
```java
public void startActivityForResult(@RequiresPermission Intent intent, int requestCode,
        @Nullable Bundle options) {
    if (mParent == null) {
        // 在本次案例中这个options返回值仍然为null
        options = transferSpringboardActivityOptions(options);
        // 把本次跳转的请求交给Instrumentation类继续处理
        Instrumentation.ActivityResult ar =
            mInstrumentation.execStartActivity(
                this, mMainThread.getApplicationThread(), mToken, this,
                intent, requestCode, options);
        if (ar != null) {
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
这里涉及到了一个mParent对象，这个mParent是Activity类型，是以前搭配ActivityGroup使用，不过这套API已经过时了，目前已经被Fragment所取代。
这个mParent对象的值通常是null，所以本次案例上，该判断结果为true。并不会触发else相关逻辑。
在之后的流程中会走到`Instrumentation`的`execStartActivity`中继续启动，从这里开始传递的参数就逐渐变多起来了，后续再慢慢介绍
```java
public ActivityResult execStartActivity(
        Context who, IBinder contextThread, IBinder token, Activity target,
        Intent intent, int requestCode, Bundle options) {
    IApplicationThread whoThread = (IApplicationThread) contextThread;
    // 触发一下onProvideReferrer的回调，这个函数可以用作埋点场景，用来获取跳转来源
    Uri referrer = target != null ? target.onProvideReferrer() : null;
    if (referrer != null) {
        intent.putExtra(Intent.EXTRA_REFERRER, referrer);
    }
    // 这里是自动化测试会用到的逻辑，我们本次案例不用关心
    if (mActivityMonitors != null) {
        synchronized (mSync) {
            final int N = mActivityMonitors.size();
            for (int i=0; i<N; i++) {
                final ActivityMonitor am = mActivityMonitors.get(i);
                ActivityResult result = null;
                if (am.ignoreMatchingSpecificIntents()) {
                    result = am.onStartActivity(intent);
                }
                if (result != null) {
                    am.mHits++;
                    return result;
                } else if (am.match(who, null, intent)) {
                    am.mHits++;
                    if (am.isBlocking()) {
                        return requestCode >= 0 ? am.getResult() : null;
                    }
                    break;
                }
            }
        }
    }
    try {
        intent.migrateExtraStreamToClipData(who);
        intent.prepareToLeaveProcess(who);
        // 这里开始跨进程调用系统服务，继续启动
        int result = ActivityTaskManager.getService().startActivity(whoThread,
                who.getBasePackageName(), who.getAttributionTag(), intent,
                intent.resolveTypeIfNeeded(who.getContentResolver()), token,
                target != null ? target.mEmbeddedID : null, requestCode, 0, null, options);
        checkStartActivityResult(result, intent);
    } catch (RemoteException e) {
        throw new RuntimeException("Failure from system", e);
    }
    return null;
}
```
这个函数中，看起来代码量挺多的，但是核心逻辑其实就一个，就是把启动请求转发给`ActivityTaskManager`然后就等待启动结果了
这里简单介绍几个`execStartActivity`的参数
`who`代表是发起方，在本次案例里指的是MainActivity
`contextThread`代表的是发起方对应的进程信息，是`ApplicationThread`对象
`token`代表的是发起方的一个标记，用于查找发起方的`ActivityRecord`
`target`代表的是发起方，是一个Activity类型，用于回调`onProvideReferrer`
`intent`这个就是封装了目标页面的意图信息
`requestCode`这个就是启动code，本案例中为-1
`options`启动参数，涉及到一些动画之类的，本案例中为null
接下来我们通过IPC机制来到`system_server`进程中，进入`ActivityTaskManager`类中一探究竟吧.
```java
public final int startActivity(IApplicationThread caller, String callingPackage,
        String callingFeatureId, Intent intent, String resolvedType, IBinder resultTo,
        String resultWho, int requestCode, int startFlags, ProfilerInfo profilerInfo,
        Bundle bOptions) {
    return startActivityAsUser(caller, callingPackage, callingFeatureId, intent, resolvedType,
            resultTo, resultWho, requestCode, startFlags, profilerInfo, bOptions,
            UserHandle.getCallingUserId());
}
```
可以看到进入`system_server`进程的第一个函数很简单，仅仅是内部调用了一下`startActivityAsUser`函数，多传递了一个`userId`参数。这里的`userId`则是通过`Binder`获取调用者的uid，然后再通过`UserHandle`计算得到。这个值通常是机主，也就是0。好，继续向下看吧
```java
public int startActivityAsUser(IApplicationThread caller, String callingPackage,
        String callingFeatureId, Intent intent, String resolvedType, IBinder resultTo,
        String resultWho, int requestCode, int startFlags, ProfilerInfo profilerInfo,
        Bundle bOptions, int userId) {
    return startActivityAsUser(caller, callingPackage, callingFeatureId, intent, resolvedType,
            resultTo, resultWho, requestCode, startFlags, profilerInfo, bOptions, userId,
            true /*validateIncomingUser*/);
}
```
这个函数也比较简单，调用了另一个重载函数，多传递了一个true进去。继续往下看
```java
private int startActivityAsUser(IApplicationThread caller, String callingPackage,
        @Nullable String callingFeatureId, Intent intent, String resolvedType,
        IBinder resultTo, String resultWho, int requestCode, int startFlags,
        ProfilerInfo profilerInfo, Bundle bOptions, int userId, boolean validateIncomingUser) {
    assertPackageMatchesCallingUid(callingPackage);
    enforceNotIsolatedCaller("startActivityAsUser");

    // 对调用用户做合法性校验、如果不是同一个用户，还要校验权限等
    userId = getActivityStartController().checkTargetUser(userId, validateIncomingUser,
            Binder.getCallingPid(), Binder.getCallingUid(), "startActivityAsUser");

    // TODO: Switch to user app stacks here.
    return getActivityStartController().obtainStarter(intent, "startActivityAsUser")
            .setCaller(caller)
            .setCallingPackage(callingPackage)
            .setCallingFeatureId(callingFeatureId)
            .setResolvedType(resolvedType)
            .setResultTo(resultTo)
            .setResultWho(resultWho)
            .setRequestCode(requestCode)
            .setStartFlags(startFlags)
            .setProfilerInfo(profilerInfo)
            .setActivityOptions(bOptions)
            .setUserId(userId)
            .execute();

}
```
在这个函数里，做了一下用户的合法性校验，校验通过后继续向下执行，通过`obtainStarter`函数获取一个`ActivityStarter`对象，然后把启动请求涉及的参数都通过set方法设置进去。最后调用`execute`来继续启动