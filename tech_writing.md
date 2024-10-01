# Context란?

## Context 개념
어플리케이션에 환경에 관한 전역적인 정보를 전달하기 위한 인터페이스를 `context`라고 합니다. 또한 이것은 안드로이드 시스템에서 구현을 제공하는 추상클래스입니다.

`context`를 통해 다음과 같은 행동들을 할 수 있습니다.
- 어플리케이션 환경에 대한 리소스 및 클래스에 접근
    - ex) packageName, resource
- broadcasting 및 intent 수신과 같은 어플리케이션 수준의 작업에 대한 up-call을 허용
     - ex) startActivity, bindService

## Context 생성 원리 

위의 설명과 같이, `context`는 추상 클래스이기 때문에 구현체가 필요합니다.

그 구현체가 바로 `ContextWrapper`입니다.


### `ContextWrapper`
- `context`에 대한 모든 호출을 `ContextWrapper`이 위임하여 대행
- 원래의 `Context`를 변경하지 않고 원하는 동작만 수정하기 위해 사용


```
public class ContextWrapper extends Context {
    @UnsupportedAppUsage
    Context mBase;

    protected void attachBaseContext(Context base) {
        if (mBase != null) {
            throw new IllegalStateException("Base context already set");
        }
        mBase = base;
    }
}
```

`attachBaseContext()` 메서드를 이용해 `baseContext`를 초기화 합니다.

### `ContextThemeWrapper`

`ContextThemeWrapper`는 `ContextWrapper`의 wrapper입니다.
`Context`의 테마를 수정하거나 교체할 때 사용합니다.

```
public class ContextThemeWrapper extends ContextWrapper {

    public ContextThemeWrapper() {
        super(null);
    }

    public ContextThemeWrapper(Context base, @StyleRes int themeResId) {
        super(base);
        mThemeResource = themeResId;
    }

    public ContextThemeWrapper(Context base, Resources.Theme theme) {
        super(base);
        mTheme = theme;
    }

}
```

### `Activity`

그렇다면 context는 어떻게 초기화될까요?

Activity의 `attachBaseContext()` 메서드를 이용해 `mBase`를 초기화합니다.


```
@UiContext
public class Activity extends ContextThemeWrapper {

    @Override
    protected void attachBaseContext(Context newBase) {
        super.attachBaseContext(newBase);
        if (newBase != null) {
            newBase.setAutofillClient(getAutofillClient());
            newBase.setContentCaptureOptions(getContentCaptureOptions());
        }
    }

}

public class ContextThemeWrapper extends ContextWrapper {

    public ContextThemeWrapper() {
        super(null);
    }

    public ContextThemeWrapper(Context base, @StyleRes int themeResId) {
        super(base);
        mThemeResource = themeResId;
    }

    // ...

}

public class ContextWrapper extends Context {
    protected void attachBaseContext(Context base) {
        if (mBase != null) {
            throw new IllegalStateException("Base context already set");
        }
        mBase = base;
    }
}

```

그리고 `attachBaseContext()` 메서드는 Activity 내 `attach` 메서드 내부에서 호출됩니다. 

```
@UnsupportedAppUsage(maxTargetSdk = Build.VERSION_CODES.R, trackingBug = 170729553)
final void attach(Context context, ActivityThread aThread,
        Instrumentation instr, IBinder token, int ident,
        Application application, Intent intent, ActivityInfo info,
        CharSequence title, Activity parent, String id,
        NonConfigurationInstances lastNonConfigurationInstances,
        Configuration config, String referrer, IVoiceInteractor voiceInteractor,
        Window window, ActivityConfigCallback activityConfigCallback, IBinder assistToken,
        IBinder shareableActivityToken) {
    attachBaseContext(context);
}
```

### `ActivityThread`
- `ActivityThread`는 애플리케이션 프로세스에서 메인 스레드의 실행을 관리하고, 활동, 브로드캐스트 및 기타 작업을 예약 및 실행하며, 활동 관리자가 요청하는 대로 activity manager가 요청하는 작업을 관리합니다.


```
/**  Core implementation of activity launch. */
private Activity performLaunchActivity(ActivityClientRecord r, Intent customIntent) {

        // ...

        ContextImpl appContext = createBaseContextForActivity(r);
        Activity activity = null;


        try {
            Application app = r.packageInfo.makeApplicationInner(false, mInstrumentation);

            // ...

            activity.attach(
                /* context = */ appContext, 
                /* application = */ app, 
                // ...
            );
        } 
        
        return activity;
}
```

`ActivityThread`는 어플리케이션의 시작점이며, 액티비티를 launch 할 때 perfomLaunchActivity를 호출합니다.

- 엄밀히 따지면 `ZygoteInit`에서 프로세스가 시작되지만, 메인 Looper가 ActivityThread에서 초기화되므로 어플리케이션의 시작점과 같은 표현이라 생각합니다.


### `ApplicationContext`

그리고 지금부터, `applicatonContext`가 어떻게 생성되는지 뜯어보겠습니다.

우선은 함수가 호출되는 순서를 먼저 설명하겠습니다.

```
public static void main(String[] args) {
        Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "ActivityThreadMain");

        // Install selective syscall interception
        AndroidOs.install();

        // ...

        / Call per-process mainline module initialization.
        initializeMainlineModules();

        Process.setArgV0("<pre-initialized>");

        Looper.prepareMainLooper();

        ActivityThread thread = new ActivityThread();
        thread.attach(false, startSeq);
}
```

`ActivityThread` 내부에서 `thread.attach(false, startSeq)`를 통해 `applicationContext`가 초기화되고 생성됩니다.

```
@UnsupportedAppUsage
private void attach(boolean system, long startSeq) {
    sCurrentActivityThread = this;

    if (!system) {
        android.ddm.DdmHandleAppName.setAppName("<pre-initialized>",
                                                    UserHandle.myUserId());
        RuntimeInit.setApplicationObject(mAppThread.asBinder());
        final IActivityManager mgr = ActivityManager.getService();
        try {
            mgr.attachApplication(mAppThread, startSeq);
        } catch (RemoteException ex) {
            throw ex.rethrowFromSystemServer();
        }
    }

}

```

`attach` 메서드 내부를 보면, `attachApplication` 메서드를 통해 초기화하고 있습니다.
이렇게 코드를 따라가면, `ActivityManagerService`와 `ApplicationThread`에 도달하여 `applicationContext`를 초기화합니다.

이에 대한 자세한 내용은, 주제에서 벗어나기 때문에 추후에 다른 문서에서 다루도록 하겠습니다.

그 후 `ApplicationThread`를 통해 Handler로 메시지를 받아, 메시지의 데이터를 이용해 `handleBindApplication`메서드를 통해 `application`을 생성합니다.

```
@UnsupportedAppUsage
private void handleBindApplication(AppBindData data) {
    app = data.info.makeApplicationInner(data.restrictedBackupMode, null);

    mInitialApplication = app;  

    // ...
}
``` 

이렇게 하여, `mInitialApplication`가 초기화 되고, `currentApplication` 메서드를  통해 Applicaton 객체를 싱글톤으로 가져오게 됩니다.

```
@UnsupportedAppUsage
public static Application currentApplication() {
    ActivityThread am = currentActivityThread();
    return am != null ? am.mInitialApplication : null;
}

```

위의 로직 순서를 정리하면 다음과 같습니다.

1. `ActivityThread#main` 메서드의 thread.attach(false, startSeq) 실행
    - ApplicationContext를 초기화하고 싱글톤으로 생성
        - ActivityManager.getService() 를 통해 `ActivityManagerService` 를 가져옴
        - `attachApplication` 메서드 실행

2. `ActivityManagerService#attachApplicaton` 실행
    - 어플리케이션 데이터를 가져와서 `AppliactionContext` 생성
        - `attachApplicationLocked` 메서드에서 어플리케이션 데이터를 가져오고, `ApplicationThread`에 전달합니다.
        - `ApplicationThread#bindApplication` 메서드 파라미터를 통해 들어온 데이터들를 `AppBundle`로 랩핑합니다.
        - `Handler`를 통해 랩핑한 데이터를 전달합니다.

3. `ActivityThread#handleBindApplication` 실행 
    - Handler를 통해 전달받은 데이터를 이용해 `applicationContext` 생성


이제, `ApplicationContext`를 초기화하는 과정을 자세히 뜯어보겠습니다.

```
private void handleBindApplication(AppBindData data) {

    // ...
    app = data.info.makeApplicationInner(data.restrictedBackupMode, null);
    mInitialApplication = app;
}
```

`makeApplicationInner` 메서드는 `applicationContext`를 생성하고 있습니다.

이때, `data.info`는 `LoadedApk`라는 클래스를 제공합니다. 
이 클래스는 현재 로드된 apk에 대해 유지되는 로컬 상태를 제공하는 클래스입니다.

**이렇게 어플리케이션에 관한 전역 정보를 이 `LoadedApk` 클래스에서 가져와 어플리케이션 객체를 생성합니다.**

`makeApplication` 코드 내부는 다음과 같습니다.

```
private Application makeApplicationInner(boolean forceDefaultAppClass,
       Instrumentation instrumentation, boolean allowDuplicateInstances) {

    ContextImpl appContext = ContextImpl.createAppContext(mActivityThread, this);
    app = mActivityThread.mInstrumentation.newApplication(cl, appClass, appContext);
    appContext.setOuterContext(app);

    //...

    return app;
}

```

makeApplication 메서드에서는 `ContextImpl#createAppContext`를 이용해 ContextImpl 객체를 생성하고 application을 생성하고 있습니다.

ContextImpl 내부를 보면

```
class ContextImpl extends Context {
    // ...

    final @NonNull ActivityThread mMainThread;
    final @NonNull LoadedApk mPackageInfo;

    private ContextImpl(
        @NonNull ActivityThread mainThread,
        @NonNull LoadedApk packageInfo,
        // ...
    ) {
        // ...

        mMainThread = mainThread;
        mPackageInfo = packageInfo;
    }

    @Override
    public Context createApplicationContext(ApplicationInfo application, int flags)
            throws NameNotFoundException {
        LoadedApk pi = mMainThread.getPackageInfo(application, mResources.getCompatibilityInfo(),
                flags | CONTEXT_REGISTER_PACKAGE);
        if (pi != null) {
            ContextImpl c = new ContextImpl(this, mMainThread, pi, ContextParams.EMPTY,
                    mAttributionSource.getAttributionTag(),
                    mAttributionSource.getNext(),
                    null, mToken, new UserHandle(UserHandle.getUserId(application.uid)),
                    flags, null, null);

            final int displayId = getDisplayId();
            final Integer overrideDisplayId = mForceDisplayOverrideInResources
                    ? displayId : null;

            c.setResources(createResources(mToken, pi, null, overrideDisplayId, null,
                    getDisplayAdjustments(displayId).getCompatibilityInfo(), null));
            if (c.mResources != null) {
                return c;
            }
        }

        // ...
    }
}
```

이렇게 packInfo와 mainThread 데이터를 저장하고 있습니다.

이러한 데이터를 바탕으로 `ApplicationContext` 객체를 생성합니다.

그리고 생성한 `ÀpplicationContext`를 Intstrumentation의 newApplication 메서드 파라미터로 넘겨서 `Application` 객체를 생성합니다.

- `Intrumentation`은 어플리케이션 계측 코드 구현을 위한 베이스 클래스 입니다.

```
public Application newApplication(ClassLoader cl, String className, Context context)
        throws InstantiationException, IllegalAccessException, 
        ClassNotFoundException {
    Application app = getFactory(context.getPackageName())
            .instantiateApplication(cl, className);
    app.attach(context);
    return app;
}
```

이렇게 newApplication 메서드를 통해 application 객체를 생성하고 applicationContext를 초기화 합니다.


### `ActivityContext`

그렇다면, `ActivityContext`와 `ApplicatonContext`는 어떻게 다를까요?

다시 돌아와 `ActivityThread#performLaunchActiviy`를 보겠습니다.

```
static ContextImpl createActivityContext(ActivityThread mainThread,
        LoadedApk packageInfo, ActivityInfo activityInfo, IBinder activityToken, int displayId,
        Configuration overrideConfiguration) {
    if (packageInfo == null) throw new IllegalArgumentException("packageInfo");

    // ...

    ContextImpl context = new ContextImpl(null, mainThread, packageInfo,            ContextParams.EMPTY,
        attributionTag, null, activityInfo.splitName, activityToken, null, 0, classLoader,
        null);
    context.mContextType = CONTEXT_TYPE_ACTIVITY;
    context.mIsConfigurationBasedContext = true;

    // Create the base resources for which all configuration contexts for this Activity
    // will be rebased upon.
    context.setResources(resourcesManager.createBaseTokenResources(activityToken,
            packageInfo.getResDir(),
            splitDirs,
            packageInfo.getOverlayDirs(),
            packageInfo.getOverlayPaths(),
            packageInfo.getApplicationInfo().sharedLibraryFiles,
            displayId,
            overrideConfiguration,
            compatInfo,
            classLoader,
            packageInfo.getApplication() == null ? null
                    : packageInfo.getApplication().getResources().getLoaders()));
    context.setDisplay(resourcesManager.getAdjustedDisplay(
            displayId, context.getResources()));
    return context;
}
```

`createActivityContext`는 `createAppContext`와 달리 Context에 리소스를 직접 생성하여 지정합니다.
또한, `CONTEXT_TYPE_ACTIVITY`으로 Context의 타입을 지정합니다.

마지막으로, ContextWrapper 본래의 클래스를 그대로 사용하여 별도의 객체를 생성하고 있습니다.

이러한 부분 때문에 ActivityContext와 ApplicationContext 간의 차이가 발생합니다.
