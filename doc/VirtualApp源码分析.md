最近发现了一个非常好的开源项目，基本实现了一个 Android 上的沙箱环境，不过应用场景最多的还是应用双开。

【VA github】: https://github.com/asLody/VirtualApp

【VA 的源码注释】: https://github.com/ganyao114/VA_Doc

第一章主要是分析一下项目的整体结构。

# 1 包结构

![csdn.jpg](https://raw.githubusercontent.com/zogodo/SandVXposed/master/doc/csdn1.png)

## android.content

主要是 PackageParser, 该类型覆盖了系统的隐藏类 android.content.pm.PackageParser

## com.lody.virtual

这里就是框架的主体代码了

## com.lody.virtual.client

运行在客户端的代码，指加载到 VA 中的子程序在被 VA 代理(hook)之后, 所运行的代码

## com.lody.virtual.client.hook

hook java 层函数的一些代码

## com.lody.virtual.client.ipc

伪造的一些 framework 层的 IPC 服务类，诸如 ActivityManager, ServiceManager 等等，使用 VXXXXX 命名。hook 之后，子程序就会运行到这里而·不是原来真正的系统 framework 代码。

## com.lody.virtual.client.stub

系统四大组件的插桩，如提前注册在 Menifest 里的几十个 StubActivity。

## com.lody.virtual.remote

一些可序列化 Model，继承于 Parcelable

## com.lody.virtual.server

server 端代码，VA 伪造了一套 framework 层系统 service 的代码，他在一个独立的服务中记录管理组件的各种 Recorder，其逻辑其实与系统原生的相近，通过 Binder 与 client 端的 ipc 包中的 VXXXXManager 通讯。诸如 AMS(VAMS), PMS(VPMS)。

## mirror

系统 framework 的镜像，实现了与 framework 层相对应的结构，封装了反射获取系统隐藏字段和方法的，便于直接调用获取或者赋值以及调用方法。

# 2 Base 一些基础措施的封装

## Mirror framework 层镜像

## 成员变量 Field 映射

根据成员变量类型，映射类型分为几个基本数据类型和对象引用类型。下面就以对象引用类型为例，其他类型类似。

类型 RefObject 代表映射 framework 层同名的泛型类型成员变量。

```java
// Field 映射
@SuppressWarnings("unchecked")
public class RefObject<T> {

    // framework 层对应的 Field
    private Field field;

    public RefObject(Class<?> cls, Field field) throws NoSuchFieldException {
        // 获取 framework 中同名字段的 field
        this.field = cls.getDeclaredField(field.getName());
        this.field.setAccessible(true);
    }

    // 获取变量值
    public T get(Object object) {
        try {
            return (T) this.field.get(object);
        } catch (Exception e) {
            return null;
        }
    }
    // 赋值
    public void set(Object obj, T value) {
        try {
            this.field.set(obj, value);
        } catch (Exception e) {
            //Ignore
        }
    }
}
```

以 framework 层中隐藏类 LoadedApk 来说：

```java
public class LoadedApk {
    public static Class Class = RefClass.load(LoadedApk.class, "android.app.LoadedApk");
    public static RefObject<ApplicationInfo> mApplicationInfo;
    @MethodParams({boolean.class, Instrumentation.class})
    public static RefMethod<Application> makeApplication;
```

mApplicationInfo 就是 LoadedApk 中私有 ApplicationInfo 类型字段的同名映射。

当你引用 LoadedApk Mirror 类时，类加载器加载该类，执行静态成员的初始化表达式 RefClass.load(LoadedApk.class, “android.app.LoadedApk”); Mirror 类中的同名字段将被反射赋值。

下面看一下 RefClass.load() 函数

```java
public static Class load(Class mappingClass, Class<?> realClass) {
        // 获取 Mirror 类的所有字段
        Field[] fields = mappingClass.getDeclaredFields();
        for (Field field : fields) {
            try {
                // 必须是 static 变量
                if (Modifier.isStatic(field.getModifiers())) {
                    // 从预设的 Map 中找到 RefXXXX 的构造器
                    Constructor<?> constructor = REF_TYPES.get(field.getType());
                    if (constructor != null) {
                        // 赋值
                        field.set(null, constructor.newInstance(realClass, field));
                    }
                }
            }
            catch (Exception e) {
                // Ignore
            }
        }
        return realClass;
    }
```

最后调用的话 MirrorClass.mirrorField.get(instance), MirrorClass.mirrorField.set(instance), 就相当于直接调用 framework 层的隐藏字段了。

## Method 映射

其实与 Field 类似，只是 Field 主要是一个 call 即调用方法。

```java
@MethodParams({File.class, int.class})
public static RefMethod<PackageParser.Package> parsePackage;
```

表现在 Mirror 类型中也是一个字段，不过要在字段上边加上注解以标注参数类型。

当然还有一种情况，参数类型也是隐藏的，则要使用全限定名表示

```java
 @MethodReflectParams({"android.content.pm.PackageParser$Package", "int"})
public static RefMethod<Void> collectCertificates;
```

## Java 层 Hook

位于 com.lody.virtual.client.hook

Java 层使用了 Java 自带的动态代理

1. MethodProxy
   Hook 点的代理接口，动态代理中的 call。

   重要的是这三个方法：

```java
public boolean beforeCall(Object who, Method method, Object... args) {
        return true;
    }

public Object call(Object who, Method method, Object... args) throws Throwable {
        return method.invoke(who, args);
    }

public Object afterCall(Object who, Method method, Object[] args, Object result) throws Throwable {
        return result;
    }
```

以 hook getServices 为例：

```java
static class GetServices extends MethodProxy {
        @Override
        public String getMethodName() {
            return "getServices";
        }

        @Override
        public Object call(Object who, Method method, Object... args) throws Throwable {
            int maxNum = (int) args[0];
            int flags = (int) args[1];
            return VActivityManager.get().getServices(maxNum, flags).getList();
        }

        @Override
        public boolean isEnable() {
            return isAppProcess();
        }
    }
```

A. getMethodName 是要 Hook 的方法名
B. Hook getServices 之后发现，真正返回服务的方法变成了仿造的 VActivityManager 对象。而在后面我们会知道这些服务最后都会从 VAMS 中获取，而不是原来的 AMS。
C. 实现了 isEnable 方法，这是 Hook 开关，如果返回 false 则不 Hook 该方法，而在这里的条件是，只有在子程序环境中 Hook，而宿主即框架是不需要 Hook 的，框架仍然需要连接真正的 AMS 以获取在系统 AMS 中注册的“外部” service。

1. 那么上面这个 call 在哪里被调用呢？
   MethodInvocationStub
   这个桩对应一个需要 Hook 的类，各种 Method 可以在内部添加.
   我们需要专注这个方法

```java
/**
     * Add a method proxy.
     *
     * @param methodProxy proxy
     */
    public MethodProxy addMethodProxy(MethodProxy methodProxy) {
        if (methodProxy != null && !TextUtils.isEmpty(methodProxy.getMethodName())) {
            if (mInternalMethodProxies.containsKey(methodProxy.getMethodName())) {
                VLog.w(TAG, "The Hook(%s, %s) you added has been in existence.", methodProxy.getMethodName(),
                        methodProxy.getClass().getName());
                return methodProxy;
            }
            mInternalMethodProxies.put(methodProxy.getMethodName(), methodProxy);
        }
        return methodProxy;
    }
```

这个也是关于动态代理的知识，这里的区别其实就是 Lody 对他做了一些接口的抽象，和一些诸如 Log 的封装。

添加 Hook Method 的方式有两个，一是调用 addMethodProxy，二是在 Stub 上添加 @Inject 注解，具体见下一段。

1. 关于 MethodProxies
   叫这个名字的类很多，一个 MethodProxies 对应一个需要 Hook 的 framework 类型，需要 Hook 的方法以内部类(MethodProxy)的形式罗列在内部。

```java
@Inject(MethodProxies.class)
public class LibCoreStub extends MethodInvocationProxy<MethodInvocationStub<Object>> {
```

将要 Hook 的方法集合 MethodProxies 绑定到 Stub 上。然后就是 Stub 对自己头上注解的解析，最终还是会调用到内部的 addMethodProxy 方法。

```java
protected void onBindMethods() {

        if (mInvocationStub == null) {
            return;
        }
        Class<? extends MethodInvocationProxy> clazz = getClass();
        Inject inject = clazz.getAnnotation(Inject.class);
        if (inject != null) {
            Class<?> proxiesClass = inject.value();
            Class<?>[] innerClasses = proxiesClass.getDeclaredClasses();
            // 遍历内部类
            for (Class<?> innerClass : innerClasses) {
                if (!Modifier.isAbstract(innerClass.getModifiers())
                        && MethodProxy.class.isAssignableFrom(innerClass)
                        && innerClass.getAnnotation(SkipInject.class) == null) {
                    addMethodProxy(innerClass);
                }
            }

        }
    }
```

# 3  运行时结构

这点很重要，VA 在运行时并不是一个简单的单进程的库，其需要在系统调用到其预先注册的 Stub 组件之后接手系统代理 Client App 的四大组件，包括生命周期等一切事物。
VA 参照原生系统 framework 仿造了一套 framework service，还有配套在 client 端的 framework 库。

1. 首先来看一下系统原生的 framework 运作方式
   简单来说，我们平时所用到的 app 运行空间中的 framework api 最终会通过 Binder 远程调用到 framework service 空间的远程服务。
   而远程服务类似 AMS 中的 Recoder 中会持有 app 空间的 Ibinder token 句柄，通过 token 也可以让 framework service 远程调用到 app 空间。

   ![csdn.jpg](https://raw.githubusercontent.com/zogodo/SandVXposed/master/doc/csdn2.png)

2. VA 环境下：
   而在 VA 环境下，情况其实也是类似，只不过在 framework service 和 client app 之间还有另外一个 VA 实现的 VAService，VAService 仿造了 framework service 的一些功能。
   因为在 VA 中运行的 Client App 都是没有(也不能注册)在 framework service 的，注册的只有 VA 预先注册在 Menifest 中的 Stub 而已。所以 frameservice 是无法像普通 App 一样管理 VA Client App 的会话的。
   这就要依靠 VA 仿造的另外一套 VAService 完成对 VA 中 Client App 的会话管理了。

   ![csdn.jpg](https://raw.githubusercontent.com/zogodo/SandVXposed/master/doc/csdn3.png)

   需要注意的是 VA Client 获取 VA Service 的 IBinder 句柄是统一通过 IServiceFetcher 这个句柄，这个看上去有些奇怪。而获得 IServiceFetcher 本身的方式是通过 ContentProvider，可能是 Provider call 的性能较好一些。

# 4 VA 初始化

先看一下代码：
VirtualCore.startup

```java
public void startup(Context context) throws Throwable {
        if (!isStartUp) {
            // 确保 MainThread
            if (Looper.myLooper() != Looper.getMainLooper()) {
                throw new IllegalStateException("VirtualCore.startup() must called in main thread.");
            }
            VASettings.STUB_CP_AUTHORITY = context.getPackageName() + "." + VASettings.STUB_DEF_AUTHORITY;
            ServiceManagerNative.SERVICE_CP_AUTH = context.getPackageName() + "." + ServiceManagerNative.SERVICE_DEF_AUTH;
            this.context = context;
            // 获取 ActivityThread 实例
            mainThread = ActivityThread.currentActivityThread.call();
            unHookPackageManager = context.getPackageManager();
            hostPkgInfo = unHookPackageManager.getPackageInfo(context.getPackageName(), PackageManager.GET_PROVIDERS);
            detectProcessType();
            // hook 系统类
            InvocationStubManager invocationStubManager = InvocationStubManager.getInstance();
            invocationStubManager.init();
            invocationStubManager.injectAll();
            // 修复权限管理
            ContextFixer.fixContext(context);
            isStartUp = true;
            if (initLock != null) {
                initLock.open();
                initLock = null;
            }
        }
    }
```

整个 VA 会运行在四种进程

```java
private enum ProcessType {
        /**
         * Server process
         */
        Server,
        /**
         * Virtual app process
         */
        VAppClient,
        /**
         * Main process
         */
        Main,
        /**
         * Child process
         */
        CHILD
    }
```

分别是前面提到的 VAService 进程，Client App 进程，VA 自身的 App 主进程，子进程。
这样的话，Application 就会被初始化多次，所以要在初始化的时候根据进程类型有选择的做对应的初始化工作。

InvocationStubManager.injectInternal():
主要完成对 Java 层 framework 的 Hook，将其定位到 VA 伪造 VA framework 上去。

```java
private void injectInternal() throws Throwable {
        // VA 自身的 App 进程不需要 Hook
        if (VirtualCore.get().isMainProcess()) {
            return;
        }
        // VAService 需要 Hook AMS 和 PMS
        if (VirtualCore.get().isServerProcess()) {
            addInjector(new ActivityManagerStub());
            addInjector(new PackageManagerStub());
            return;
        }
        // Client APP 需要 Hook 整个 framework，来使其调用到 VA framework
        if (VirtualCore.get().isVAppProcess()) {
            addInjector(new LibCoreStub());
            addInjector(new ActivityManagerStub());
            addInjector(new PackageManagerStub());
            addInjector(HCallbackStub.getDefault());
            addInjector(new ISmsStub());
            addInjector(new ISubStub());
            addInjector(new DropBoxManagerStub());
            .....................
```

VA 初始化主要就是这些

# 5 Client App 的安装

VirtualCore.installPackage

```java
 public InstallResult installPackage(String apkPath, int flags) {
        try {
            // 调用远程 VAService
            return getService().installPackage(apkPath, flags);
        } catch (RemoteException e) {
            return VirtualRuntime.crash(e);
        }
    }
```

最终调用 VAServcie 中的 VAppManagerService.installPackage():

```java
// 安装 apk 先于 installPackageAsUser，主要目的是生成 VPackage 结构
    public synchronized InstallResult installPackage(String path, int flags, boolean notify) {
        long installTime = System.currentTimeMillis();
        if (path == null) {
            return InstallResult.makeFailure("path = NULL");
        }
        // 是否 OPT 优化(dex -> binary)
        boolean skipDexOpt = (flags & InstallStrategy.SKIP_DEX_OPT) != 0;
        // apk path
        File packageFile = new File(path);
        if (!packageFile.exists() || !packageFile.isFile()) {
            return InstallResult.makeFailure("Package File is not exist.");
        }
        VPackage pkg = null;
        try {
            // 进入解析包结构，该结构是可序列化的，为了持久化在磁盘上
            pkg = PackageParserEx.parsePackage(packageFile);
        } catch (Throwable e) {
            e.printStackTrace();
        }
        if (pkg == null || pkg.packageName == null) {
            return InstallResult.makeFailure("Unable to parse the package.");
        }
        InstallResult res = new InstallResult();
        res.packageName = pkg.packageName;
        // PackageCache holds all packages, try to check if we need to update.
        VPackage existOne = PackageCacheManager.get(pkg.packageName);
        PackageSetting existSetting = existOne != null ? (PackageSetting) existOne.mExtras : null;
        if (existOne != null) {
            if ((flags & InstallStrategy.IGNORE_NEW_VERSION) != 0) {
                res.isUpdate = true;
                return res;
            }
            if (!canUpdate(existOne, pkg, flags)) {
                return InstallResult.makeFailure("Not allowed to update the package.");
            }
            res.isUpdate = true;
        }
        // 获得 app 安装文件夹
        File appDir = VEnvironment.getDataAppPackageDirectory(pkg.packageName);
        // so 文件夹
        File libDir = new File(appDir, "lib");
        if (res.isUpdate) {
            FileUtils.deleteDir(libDir);
            VEnvironment.getOdexFile(pkg.packageName).delete();
            VActivityManagerService.get().killAppByPkg(pkg.packageName, VUserHandle.USER_ALL);
        }
        if (!libDir.exists() && !libDir.mkdirs()) {
            return InstallResult.makeFailure("Unable to create lib dir.");
        }

        // 是否基于系统的 apk 加载，前提是安装过的 apk 并且 dependSystem 开关打开
        boolean dependSystem = (flags & InstallStrategy.DEPEND_SYSTEM_IF_EXIST) != 0
                && VirtualCore.get().isOutsideInstalled(pkg.packageName);

        if (existSetting != null && existSetting.dependSystem) {
            dependSystem = false;
        }
        // 复制 so 到 sandbox lib
        NativeLibraryHelperCompat.copyNativeBinaries(new File(path), libDir);

        // 如果不基于系统，一些必要的拷贝工作
        if (!dependSystem) {
            File privatePackageFile = new File(appDir, "base.apk");
            File parentFolder = privatePackageFile.getParentFile();
            if (!parentFolder.exists() && !parentFolder.mkdirs()) {
                VLog.w(TAG, "Warning: unable to create folder : " + privatePackageFile.getPath());
            } else if (privatePackageFile.exists() && !privatePackageFile.delete()) {
                VLog.w(TAG, "Warning: unable to delete file : " + privatePackageFile.getPath());
            }
            try {
                FileUtils.copyFile(packageFile, privatePackageFile);
            } catch (IOException e) {
                privatePackageFile.delete();
                return InstallResult.makeFailure("Unable to copy the package file.");
            }
            packageFile = privatePackageFile;
        }
        if (existOne != null) {
            PackageCacheManager.remove(pkg.packageName);
        }

        // 给上可执行权限，5.0 之后在 SD 卡上执行 bin 需要可执行权限
        chmodPackageDictionary(packageFile);

        // PackageSetting 的一些配置，后面会序列化在磁盘上
        PackageSetting ps;
        if (existSetting != null) {
            ps = existSetting;
        } else {
            ps = new PackageSetting();
        }
        ps.skipDexOpt = skipDexOpt;
        ps.dependSystem = dependSystem;
        ps.apkPath = packageFile.getPath();
        ps.libPath = libDir.getPath();
        ps.packageName = pkg.packageName;
        ps.appId = VUserHandle.getAppId(mUidSystem.getOrCreateUid(pkg));
        if (res.isUpdate) {
            ps.lastUpdateTime = installTime;
        } else {
            ps.firstInstallTime = installTime;
            ps.lastUpdateTime = installTime;
            for (int userId : VUserManagerService.get().getUserIds()) {
                boolean installed = userId == 0;
                ps.setUserState(userId, false/*launched*/, false/*hidden*/, installed);
            }
        }
        //保存 VPackage Cache 到 Disk
        PackageParserEx.savePackageCache(pkg);
        //保存到 RamCache
        PackageCacheManager.put(pkg, ps);
        mPersistenceLayer.save();
        BroadcastSystem.get().startApp(pkg);
        //发送通知 安装完成
        if (notify) {
            notifyAppInstalled(ps, -1);
        }
        res.isSuccess = true;
        return res;
    }
```

APk 的安装主要完成以下几件事情:
1. 解析 menifest 拿到 apk 内部信息，包括组件信息，权限信息等。并将这些信息序列化到磁盘和内存中，以备打开时调用。
2. 准备 App 在 VA 沙箱环境中的私有空间，并且复制一些必要的 apk 和 so libs。
3. 最后通知前台安装完成。

解析 apk menifest：PackageParserEx.parsePackage():

```java
 // 解析包结构
    public static VPackage parsePackage(File packageFile) throws Throwable {
        PackageParser parser = PackageParserCompat.createParser(packageFile);
        // 调用对应系统版本的 parsePackage 方法
        PackageParser.Package p = PackageParserCompat.parsePackage(parser, packageFile, 0);
        // 包含此信息代表其是 share lib
        if (p.requestedPermissions.contains("android.permission.FAKE_PACKAGE_SIGNATURE")
                && p.mAppMetaData != null
                && p.mAppMetaData.containsKey("fake-signature")) {
            String sig = p.mAppMetaData.getString("fake-signature");
            p.mSignatures = new Signature[]{new Signature(sig)};
            VLog.d(TAG, "Using fake-signature feature on : " + p.packageName);
        } else {
            // 验证签名
            PackageParserCompat.collectCertificates(parser, p, PackageParser.PARSE_IS_SYSTEM);
        }
        // 转换成可以序列化在磁盘上的 Cache
        return buildPackageCache(p);
    }
```

这里解析 Menifest 的方法其实是调用了 framework 隐藏方法 android.content.pm.PackageParser.parsePackage 来实现的，这个方法返回 android.content.pm.Package 结构，这个类型也是隐藏的，怎么办？可以从 sdk 中复制这个类到自己的项目中欺骗编译器。这就是上一章一开始提到的。

这里还有一个问题，就是 Package 类是不可序列化的，换句话说就是不能直接保存在磁盘上，我们需要将其转换成可以序列化的 VPackage 类型，这就是 buildPackageCache() 的作用。

VPackage：

```java
public class VPackage implements Parcelable {

    public static final Creator<VPackage> CREATOR = new Creator<VPackage>() {
        @Override
        public VPackage createFromParcel(Parcel source) {
            return new VPackage(source);
        }

        @Override
        public VPackage[] newArray(int size) {
            return new VPackage[size];
        }
    };
    public ArrayList<ActivityComponent> activities;
    public ArrayList<ActivityComponent> receivers;
    public ArrayList<ProviderComponent> providers;
    public ArrayList<ServiceComponent> services;
    public ArrayList<InstrumentationComponent> instrumentation;
    public ArrayList<PermissionComponent> permissions;
    public ArrayList<PermissionGroupComponent> permissionGroups;
    public ArrayList<String> requestedPermissions;
    public ArrayList<String> protectedBroadcasts;
    public ApplicationInfo applicationInfo;
    public Signature[] mSignatures;
    public Bundle mAppMetaData;
    public String packageName;
    public int mPreferredOrder;
    public String mVersionName;
    public String mSharedUserId;
    public ArrayList<String> usesLibraries;
    public int mVersionCode;
    public int mSharedUserLabel;
    // Applications hardware preferences
    public ArrayList<ConfigurationInfo> configPreferences = null;
    // Applications requested features
    public ArrayList<FeatureInfo> reqFeatures = null;
    public Object mExtras;
..........................
```

可以看到 VPackage 几乎保存了 apk 中所有的关键信息，尤其是组件的数据结构会在 app 在 VA 中运行的时候给 VAMS，VPMS 这些 VAService 提供 apk 的组件信息。

关于是否 dependSystem 和 isInstallOutside，这个有关 apk 的动态加载，如果 dependSysytem 并且 apk 已经在外部环境安装了，那么 VA 会调用系统提供的 API 就可以动态加载 APK。反之 VA 需要做一些必要的复制工作然后再费劲的去加载 APK。

在这之前，我们还是要先了解一下 VA Client Framework 和 VAService 之间的通讯方式

# 6 VAService 与通讯

## VAService

首先，VAService 是指 VA 仿造 Android 原生 framework 层 Service 实现的一套副本，举例有 VActivityManagerService，它和系统 AMS 一样，只不过他管理的是 VA 内部 Client App 的组件会话。

![csdn.jpg](https://raw.githubusercontent.com/zogodo/SandVXposed/master/doc/csdn4.png)

## VAService 统一管理

首先所有 VAService 直接继承与 XXX.Stub，也就是 Binder，并且直接使用了一个 Map 储存在 VAService 进程空间中，并没有注册到系统 AMS 中，事实上在 VAService 进程中，每个 Service 都被当作一个普通对象 new 和 初始化。
最终，他们被添加到了 ServiceCache 中:

```java
public class ServiceCache {

    private static final Map<String, IBinder> sCache = new ArrayMap<>(5);

    public static void addService(String name, IBinder service) {
        sCache.put(name, service);
    }

    public static IBinder removeService(String name) {
        return sCache.remove(name);
    }

    public static IBinder getService(String name) {
        return sCache.get(name);
    }

}
```

这个 cache 很简单，就是一个 Map。

而被添加的时机则在 BinderProvider 的 onCreate() 回掉中:

```java
@Override
    public boolean onCreate() {
        Context context = getContext();
        // 这是一个空前台服务，目的是为了保活 VAService 进程，即 :x 进程
        DaemonService.startup(context);
        if (!VirtualCore.get().isStartup()) {
            return true;
        }
        VPackageManagerService.systemReady();
        addService(ServiceManagerNative.PACKAGE, VPackageManagerService.get());
        VActivityManagerService.systemReady(context);
        addService(ServiceManagerNative.ACTIVITY, VActivityManagerService.get());
        addService(ServiceManagerNative.USER, VUserManagerService.get());
        VAppManagerService.systemReady();
        addService(ServiceManagerNative.APP, VAppManagerService.get());
        BroadcastSystem.attach(VActivityManagerService.get(), VAppManagerService.get());
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP) {
            addService(ServiceManagerNative.JOB, VJobSchedulerService.get());
        }
        VNotificationManagerService.systemReady(context);
        addService(ServiceManagerNative.NOTIFICATION, VNotificationManagerService.get());
        VAppManagerService.get().scanApps();
        VAccountManagerService.systemReady();
        addService(ServiceManagerNative.ACCOUNT, VAccountManagerService.get());
        addService(ServiceManagerNative.VS, VirtualStorageService.get());
        addService(ServiceManagerNative.DEVICE, VDeviceManagerService.get());
        return true;
    }
    private void addService(String name, IBinder service) {
        ServiceCache.addService(name, service);
    }
```

需要注意的是 DeamonService 是一个空前台服务，目的是为了保活 VAService 进程，即 :x 进程，因为现在后台服务很容易被杀，在 Android 8.0 以后后台服务只能在后台存活5S，而前台服务则不受影响。

## ServiceFetcher

VA 设计了一个单独的 ServiceFetcher 服务用于向外部暴露 VAService 中的所有服务的 IBinder 句柄，而 ServiceFetcher 本身也是 Binder 服务，也就是说，ServiceFetcher 的 Ibinder 句柄是拿到其他 VAServic IBinder 的钥匙。

```java
    private class ServiceFetcher extends IServiceFetcher.Stub {
        @Override
        public IBinder getService(String name) throws RemoteException {
            if (name != null) {
                return ServiceCache.getService(name);
            }
            return null;
        }

        @Override
        public void addService(String name, IBinder service) throws RemoteException {
            if (name != null && service != null) {
                ServiceCache.addService(name, service);
            }
        }

        @Override
        public void removeService(String name) throws RemoteException {
            if (name != null) {
                ServiceCache.removeService(name);
            }
        }
    }
```

ServicecFetcher 自身的 IBnder 则通过 BinderProvicer 这个ContentProvider 暴露给其他进程:

```java
@Override
    public Bundle call(String method, String arg, Bundle extras) {
        if ("@".equals(method)) {
            Bundle bundle = new Bundle();
            BundleCompat.putBinder(bundle, "_VA_|_binder_", mServiceFetcher);
            return bundle;
        }
        return null;
    }
```

那么，在 Client App 中 VA Client 就可以通过 IServiceFetcher 这个 IBinder 拿到其他服务的 IBinder 了：

```java
    private static IServiceFetcher sFetcher;

    private static IServiceFetcher getServiceFetcher() {
        if (sFetcher == null || !sFetcher.asBinder().isBinderAlive()) {
            synchronized (ServiceManagerNative.class) {
                Context context = VirtualCore.get().getContext();
                Bundle response = new ProviderCall.Builder(context, SERVICE_CP_AUTH).methodName("@").call();
                if (response != null) {
                    IBinder binder = BundleCompat.getBinder(response, "_VA_|_binder_");
                    linkBinderDied(binder);
                    sFetcher = IServiceFetcher.Stub.asInterface(binder);
                }
            }
        }
        return sFetcher;
    }

    // 返回服务的 IBinder 句柄
    public static IBinder getService(String name) {
        // 如果是本地服务，直接本地返回
        if (VirtualCore.get().isServerProcess()) {
            return ServiceCache.getService(name);
        }
        // 通过 ServiceFetcher 的句柄找到远程 Service 的句柄
        IServiceFetcher fetcher = getServiceFetcher();
        if (fetcher != null) {
            try {
                return fetcher.getService(name);
            } catch (RemoteException e) {
                e.printStackTrace();
            }
        }
        VLog.e(TAG, "GetService(%s) return null.", name);
        return null;
    }
```

# 7 启动 App

首先要了解的是 Android App 是组件化的，Apk 其实是 N 多个组件的集合，以及一些资源文件和 Assert，App 的启动有多种情况，只要在一个新的进程中调起了 apk 中任何一个组件，App 将被初始化，Application 将被初始化。

## 启动 Activity

我们先看启动 Activity 的情况:

## Hook startActivity(重定位 Intent 到 StubActivity)

首先在 Client App 中，startActivity 方法必须被 Hook 掉，不然 Client App 调用 startActivity 就直指外部 Activity 去了。

这部分的原理其实与 DroidPlugin 大同小异，由于插件(Client App)中的 Activity 是没有在 AMS 中注册的，AMS 自然无法找到我们的插件 Activity。

Hook 的目的是我们拿到用户的 Intent，把他替换成指向 VA 在 Menifest 中站好坑的 StubActivity 的 Intent，然后将原 Intent 当作 data 打包进新 Intent 以便日后流程再次进入 VA 时恢复。

Hook 的方法就是用我们动态代理生成的代理类对象替换系统原来的 ActiityManagerNative.geDefault 对象。

```java
@Override
public void inject() throws Throwable {
    if (BuildCompat.isOreo()) {
        //Android Oreo(8.X)
        Object singleton = ActivityManagerOreo.IActivityManagerSingleton.get();
        Singleton.mInstance.set(singleton, getInvocationStub().getProxyInterface());
    } else {
        if (ActivityManagerNative.gDefault.type() == IActivityManager.TYPE) {
            ActivityManagerNative.gDefault.set(getInvocationStub().getProxyInterface());
        } else if (ActivityManagerNative.gDefault.type() == Singleton.TYPE) {
            Object gDefault = ActivityManagerNative.gDefault.get();
            Singleton.mInstance.set(gDefault, getInvocationStub().getProxyInterface());
        }
    }
    BinderInvocationStub hookAMBinder = new BinderInvocationStub(getInvocationStub().getBaseInterface());
    hookAMBinder.copyMethodProxies(getInvocationStub());
    ServiceManager.sCache.get().put(Context.ACTIVITY_SERVICE, hookAMBinder);
}
```

好了，下面只要调用到 startActivity 就会被 Hook 到 call。
这个函数需要注意以下几点：

1. VA 有意将安装和卸载 APP 的请求重定向到了卸载 VA 内部 APK 的逻辑。
2. resolveActivityInfo 调用到了 VPM 的 resolveIntent，最终会远程调用到 VPMS 的 resolveIntent，然后 VPMS 就会去查询 VPackage 找到目标 Activity 并将信息附加在 ResolveInfo 中返回 VPM。
3. 最后也是最重要的一点，startActivity 会调用到 VAM.startActivity, 同样最终会远程调用到 VAMS 的 startActivity。

```java
// Hook startActivity
static class StartActivity extends MethodProxy {

    private static final String SCHEME_FILE = "file";
    private static final String SCHEME_PACKAGE = "package";

    @Override
    public String getMethodName() {
        return "startActivity";
    }

    @Override
    public Object call(Object who, Method method, Object... args) throws Throwable {
        int intentIndex = ArrayUtils.indexOfObject(args, Intent.class, 1);
        if (intentIndex < 0) {
            return ActivityManagerCompat.START_INTENT_NOT_RESOLVED;
        }
        int resultToIndex = ArrayUtils.indexOfObject(args, IBinder.class, 2);
        String resolvedType = (String) args[intentIndex + 1];
        Intent intent = (Intent) args[intentIndex];
        intent.setDataAndType(intent.getData(), resolvedType);
        IBinder resultTo = resultToIndex >= 0 ? (IBinder) args[resultToIndex] : null;
        int userId = VUserHandle.myUserId();

        if (ComponentUtils.isStubComponent(intent)) {
            return method.invoke(who, args);
        }

        // 请求安装和卸载界面        if (Intent.ACTION_INSTALL_PACKAGE.equals(intent.getAction())
            || (Intent.ACTION_VIEW.equals(intent.getAction())
                && "application/vnd.android.package-archive".equals(intent.getType()))) {
            if (handleInstallRequest(intent)) {
                return 0;
            }
        } else if ((Intent.ACTION_UNINSTALL_PACKAGE.equals(intent.getAction())
                    || Intent.ACTION_DELETE.equals(intent.getAction()))
                   && "package".equals(intent.getScheme())) {

            if (handleUninstallRequest(intent)) {
                return 0;
            }
        }

        String resultWho = null;
        int requestCode = 0;
        Bundle options = ArrayUtils.getFirst(args, Bundle.class);
        if (resultTo != null) {
            resultWho = (String) args[resultToIndex + 1];
            requestCode = (int) args[resultToIndex + 2];
        }
        // chooser 调用选择界面
        if (ChooserActivity.check(intent)) {
            intent.setComponent(new ComponentName(getHostContext(), ChooserActivity.class));
            intent.putExtra(Constants.EXTRA_USER_HANDLE, userId);
            intent.putExtra(ChooserActivity.EXTRA_DATA, options);
            intent.putExtra(ChooserActivity.EXTRA_WHO, resultWho);
            intent.putExtra(ChooserActivity.EXTRA_REQUEST_CODE, requestCode);
            return method.invoke(who, args);
        }

        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.JELLY_BEAN_MR2) {
            args[intentIndex - 1] = getHostPkg();
        }

        //解析 ActivityInfo
        ActivityInfo activityInfo = VirtualCore.get().resolveActivityInfo(intent, userId);
        if (activityInfo == null) {
            VLog.e("VActivityManager", "Unable to resolve activityInfo : " + intent);
            if (intent.getPackage() != null && isAppPkg(intent.getPackage())) {
                return ActivityManagerCompat.START_INTENT_NOT_RESOLVED;
            }
            return method.invoke(who, args);
        }

        // 调用远程 VAMS.startActivity
        int res = VActivityManager.get().startActivity(intent, activityInfo, resultTo, options, resultWho, requestCode, VUserHandle.myUserId());
        if (res != 0 && resultTo != null && requestCode > 0) {
            VActivityManager.get().sendActivityResult(resultTo, resultWho, requestCode);
        }

        // 处理 Activity 切换动画，因为此时动画还是 Host 的 Stub Activity 默认动画，需要覆盖成子程序包的动画
        if (resultTo != null) {
            ActivityClientRecord r = VActivityManager.get().getActivityRecord(resultTo);
            if (r != null && r.activity != null) {
                try {
                    TypedValue out = new TypedValue();
                    Resources.Theme theme = r.activity.getResources().newTheme();
                    theme.applyStyle(activityInfo.getThemeResource(), true);
                    if (theme.resolveAttribute(android.R.attr.windowAnimationStyle, out, true)) {

                        TypedArray array = theme.obtainStyledAttributes(out.data,
                                                                        new int[]{
                                                                            android.R.attr.activityOpenEnterAnimation,
                                                                            android.R.attr.activityOpenExitAnimation
                                                                        });

                        r.activity.overridePendingTransition(array.getResourceId(0, 0), array.getResourceId(1, 0));
                        array.recycle();
                    }
                } catch (Throwable e) {
                    // Ignore
                }
            }
        }
        return res;
    }


    private boolean handleInstallRequest(Intent intent) {
        IAppRequestListener listener = VirtualCore.get().getAppRequestListener();
        if (listener != null) {
            Uri packageUri = intent.getData();
            if (SCHEME_FILE.equals(packageUri.getScheme())) {
                File sourceFile = new File(packageUri.getPath());
                try {
                    listener.onRequestInstall(sourceFile.getPath());
                    return true;
                } catch (RemoteException e) {
                    e.printStackTrace();
                }
            }

        }
        return false;
    }

    private boolean handleUninstallRequest(Intent intent) {
        IAppRequestListener listener = VirtualCore.get().getAppRequestListener();
        if (listener != null) {
            Uri packageUri = intent.getData();
            if (SCHEME_PACKAGE.equals(packageUri.getScheme())) {
                String pkg = packageUri.getSchemeSpecificPart();
                try {
                    listener.onRequestUninstall(pkg);
                    return true;
                } catch (RemoteException e) {
                    e.printStackTrace();
                }
            }

        }
        return false;
    }

}
```

逻辑最终走到 VAMS 后，VAMS 调用 ActivityStack.startActivityLocked

```java
// 参考 framework 的实现
int startActivityLocked(int userId, Intent intent, ActivityInfo info, IBinder resultTo, Bundle options,
                        String resultWho, int requestCode) {
    optimizeTasksLocked();

    Intent destIntent;
    ActivityRecord sourceRecord = findActivityByToken(userId, resultTo);
    TaskRecord sourceTask = sourceRecord != null ? sourceRecord.task : null;

    // 忽略一大堆对 Flag 的处理
    .............................

        String affinity = ComponentUtils.getTaskAffinity(info);

    // 根据 Flag 寻找合适的 Task
    TaskRecord reuseTask = null;
    switch (reuseTarget) {
        case AFFINITY:
            reuseTask = findTaskByAffinityLocked(userId, affinity);
            break;
        case DOCUMENT:
            reuseTask = findTaskByIntentLocked(userId, intent);
            break;
        case CURRENT:
            reuseTask = sourceTask;
            break;
        default:
            break;
    }

    boolean taskMarked = false;
    if (reuseTask == null) {
        startActivityInNewTaskLocked(userId, intent, info, options);
    } else {
        boolean delivered = false;
        mAM.moveTaskToFront(reuseTask.taskId, 0);
        boolean startTaskToFront = !clearTask && !clearTop && ComponentUtils.isSameIntent(intent, reuseTask.taskRoot);

        if (clearTarget.deliverIntent || singleTop) {
            taskMarked = markTaskByClearTarget(reuseTask, clearTarget, intent.getComponent());
            ActivityRecord topRecord = topActivityInTask(reuseTask);
            if (clearTop && !singleTop && topRecord != null && taskMarked) {
                topRecord.marked = true;
            }
            // Target activity is on top
            if (topRecord != null && !topRecord.marked && topRecord.component.equals(intent.getComponent())) {
                deliverNewIntentLocked(sourceRecord, topRecord, intent);
                delivered = true;
            }
        }
        if (taskMarked) {
            synchronized (mHistory) {
                scheduleFinishMarkedActivityLocked();
            }
        }
        if (!startTaskToFront) {
            if (!delivered) {
                destIntent = startActivityProcess(userId, sourceRecord, intent, info);
                if (destIntent != null) {
                    startActivityFromSourceTask(reuseTask, destIntent, info, resultWho, requestCode, options);
                }
            }
        }
    }
    return 0;
}
```

然后 call 到了 startActivityProcess ，这就是真正替换 Intent 的地方:

```java
// 使用 Host Stub Activity 的 Intent 包装原 Intent 瞒天过海
    private Intent startActivityProcess(int userId, ActivityRecord sourceRecord, Intent intent, ActivityInfo info) {
        intent = new Intent(intent);
        // 获得 Activity 对应的 ProcessRecorder，如果没有则表示这是 Process 第一个打开的组件，需要初始化 Application
        ProcessRecord targetApp = mService.startProcessIfNeedLocked(info.processName, userId, info.packageName);
        if (targetApp == null) {
            return null;
        }
        Intent targetIntent = new Intent();

        // 根据 Client App 的 PID 获取 StubActivity
        String stubActivityPath = fetchStubActivity(targetApp.vpid, info);

        Log.e("gy", "map activity:" + intent.getComponent().getClassName() + " -> " + stubActivityPath);

        targetIntent.setClassName(VirtualCore.get().getHostPkg(), stubActivityPath);
        ComponentName component = intent.getComponent();
        if (component == null) {
            component = ComponentUtils.toComponentName(info);
        }
        targetIntent.setType(component.flattenToString());
        StubActivityRecord saveInstance = new StubActivityRecord(intent, info,
                sourceRecord != null ? sourceRecord.component : null, userId);
        saveInstance.saveToIntent(targetIntent);
        return targetIntent;
    }
```

fetchStubActivity 会根据相同的进程 id 在 VA 的 Menifest 中找到那个提前占坑的 StubActivity：

```java
// 获取合适的 StubActivity，返回 StubActivity 全限定名
    private String fetchStubActivity(int vpid, ActivityInfo targetInfo) {

        boolean isFloating = false;
        boolean isTranslucent = false;
        boolean showWallpaper = false;
        try {
            int[] R_Styleable_Window = R_Hide.styleable.Window.get();
            int R_Styleable_Window_windowIsTranslucent = R_Hide.styleable.Window_windowIsTranslucent.get();
            int R_Styleable_Window_windowIsFloating = R_Hide.styleable.Window_windowIsFloating.get();
            int R_Styleable_Window_windowShowWallpaper = R_Hide.styleable.Window_windowShowWallpaper.get();

            AttributeCache.Entry ent = AttributeCache.instance().get(targetInfo.packageName, targetInfo.theme,
                    R_Styleable_Window);
            if (ent != null && ent.array != null) {
                showWallpaper = ent.array.getBoolean(R_Styleable_Window_windowShowWallpaper, false);
                isTranslucent = ent.array.getBoolean(R_Styleable_Window_windowIsTranslucent, false);
                isFloating = ent.array.getBoolean(R_Styleable_Window_windowIsFloating, false);
            }
        } catch (Throwable e) {
            e.printStackTrace();
        }

        boolean isDialogStyle = isFloating || isTranslucent || showWallpaper;

        // 根据在 Menifest 中注册的 pid
        if (isDialogStyle) {
            return VASettings.getStubDialogName(vpid);
        } else {
            return VASettings.getStubActivityName(vpid);
        }
    }
```

这里需要特别注意，VA 占坑的方式和 DroidPlugin 有些小不同，VA 没有为每个 Process 注册多个 Activity，也没有为不同的启动方式注册多个 Activity，这里确实是有改进的。
这里根本原因是因为 VA 对 VAMS 实现的更为完整，实现了原版 AMS 的基本功能，包括完整的 Recorder 管理，Task Stack 管理等，这样的话 StubActivity 的唯一作用便是携带 Client App 真正的 Intent 交给 VAMS 处理。这套机制衍生到其他的组件也是一样的。

可以简单看一下 ActivityStack， ActivityRecorder，ActivityRecord

```java
 class ActivityStack {

    private final ActivityManager mAM;
    private final VActivityManagerService mService;

    /**
     * [Key] = TaskId [Value] = TaskRecord
     */
    private final SparseArray<TaskRecord> mHistory = new SparseArray<>();123456789
class TaskRecord {
    public final List<ActivityRecord> activities = Collections.synchronizedList(new ArrayList<ActivityRecord>());
    public int taskId;
    public int userId;
    public String affinity;
    public Intent taskRoot;123456
/* package */ class ActivityRecord {
    public TaskRecord task;
    public ComponentName component;
    public ComponentName caller;
    // Client App 中 Activity 的句柄
    public IBinder token;
    public int userId;
    public ProcessRecord process;
    public int launchMode;
    public int flags;
    public boolean marked;
    public String affinity;
```

StubActivityRecorder：

```java
public class StubActivityRecord  {
        public Intent intent;
        public ActivityInfo info;
        public ComponentName caller;
        public int userId;

        public StubActivityRecord(Intent intent, ActivityInfo info, ComponentName caller, int userId) {
            this.intent = intent;
            this.info = info;
            this.caller = caller;
            this.userId = userId;
        }

        // 获取原版 Intent 和一些其他信息
        public StubActivityRecord(Intent stub) {
            this.intent = stub.getParcelableExtra("_VA_|_intent_");
            this.info = stub.getParcelableExtra("_VA_|_info_");
            this.caller = stub.getParcelableExtra("_VA_|_caller_");
            this.userId = stub.getIntExtra("_VA_|_user_id_", 0);
        }

    // 将原版 Intent 塞到 Stub Intent
    public void saveToIntent(Intent stub) {
        stub.putExtra("_VA_|_intent_", intent);
        stub.putExtra("_VA_|_info_", info);
        stub.putExtra("_VA_|_caller_", caller);
        stub.putExtra("_VA_|_user_id_", userId);
    }
}
```

## 初始化 Application

还有一个非常重要的事情，注意到这一行：

```java
// 获得 Activity 对应的 ProcessRecorder，如果没有则表示这是 Process 第一个打开的组件，需要初始化 Application
        ProcessRecord targetApp = mService.startProcessIfNeedLocked(info.processName, userId, info.packageName);
```

这里会先去找对应 Client App 进程的 ProcessRecorder, 找不到代表 Application 刚启动尚未初始化:

```java
private ProcessRecord performStartProcessLocked(int vuid, int vpid, ApplicationInfo info, String processName) {
        ProcessRecord app = new ProcessRecord(info, processName, vuid, vpid);
        Bundle extras = new Bundle();
        BundleCompat.putBinder(extras, "_VA_|_binder_", app);
        extras.putInt("_VA_|_vuid_", vuid);
        extras.putString("_VA_|_process_", processName);
        extras.putString("_VA_|_pkg_", info.packageName);

        // 调用子程序包的 init_process 方法，并且得到子程序包 IBinder 句柄
        Bundle res = ProviderCall.call(VASettings.getStubAuthority(vpid), "_VA_|_init_process_", null, extras);
        if (res == null) {
            return null;
        }
        int pid = res.getInt("_VA_|_pid_");
        IBinder clientBinder = BundleCompat.getBinder(res, "_VA_|_client_");
        // attach 到 Client 的 VAM , 将子程序句柄推入服务端 VAMS
        attachClient(pid, clientBinder);
        return app;
    }
```

ProviderCall.call 向 Client App 的 StubContentProvider 发起远程调用：

```java
@Override
    public Bundle call(String method, String arg, Bundle extras) {
        if ("_VA_|_init_process_".equals(method)) {
            return initProcess(extras);
        }
        return null;
    }

    private Bundle initProcess(Bundle extras) {
        ConditionVariable lock = VirtualCore.get().getInitLock();
        if (lock != null) {
            lock.block();
        }
        IBinder token = BundleCompat.getBinder(extras, "_VA_|_binder_");
        int vuid = extras.getInt("_VA_|_vuid_");
        VClientImpl client = VClientImpl.get();
        client.initProcess(token, vuid);
        Bundle res = new Bundle();
        BundleCompat.putBinder(res, "_VA_|_client_", client.asBinder());
        res.putInt("_VA_|_pid_", Process.myPid());
        return res;
    }
```

Client App 的 IBinder 句柄(VClientImpl.asBinder) 被打包在了 Bundle 中返回给 VAMS。

最终, VAMS 调用原生 AM 的 startActivity 向真正的 AMS 发送替换成 StubActivity 的伪造 Intent。

```java
 mirror.android.app.IActivityManager.startActivity.call(ActivityManagerNative.getDefault.call(),
                (Object[]) args);
```

## 恢复原 Intent 重定向到原 Activity

当 AMS 收到伪装的 Intent 后，就会找到 StubActivity，这时流程回到 VA 里的主线程中的消息队列中。

Hook 过程就是用我们自己的 Handler 替换 android.os.Handler.mCallback 因为主线程在这里分发一些操作：

```java
    @Override
        public void inject() throws Throwable {
            otherCallback = getHCallback();
            mirror.android.os.Handler.mCallback.set(getH(), this);
        }
```

handlerMessage 判断是 LAUNCH_ACTIVITY Action 后直接调用了 handlerLaunchActivity 方法，和原版其实很像。

```java
private boolean handleLaunchActivity(Message msg) {
            Object r = msg.obj;
            Intent stubIntent = ActivityThread.ActivityClientRecord.intent.get(r);
            // 获取原版 Intent 信息
            StubActivityRecord saveInstance = new StubActivityRecord(stubIntent);
            if (saveInstance.intent == null) {
                return true;
            }
            // 原版 Intent
            Intent intent = saveInstance.intent;
            ComponentName caller = saveInstance.caller;
            IBinder token = ActivityThread.ActivityClientRecord.token.get(r);
            ActivityInfo info = saveInstance.info;

            // 如果 token 还没初始化，代表 App 刚刚启动第一个组件
            if (VClientImpl.get().getToken() == null) {
                VActivityManager.get().processRestarted(info.packageName, info.processName, saveInstance.userId);
                getH().sendMessageAtFrontOfQueue(Message.obtain(msg));
                return false;
            }
            // AppBindData 为空，则 App 信息不明
            if (!VClientImpl.get().isBound()) {
                // 初始化并绑定 Application
                VClientImpl.get().bindApplication(info.packageName, info.processName);
                getH().sendMessageAtFrontOfQueue(Message.obtain(msg));
                return false;
            }

            // 获取 TaskId
            int taskId = IActivityManager.getTaskForActivity.call(
                    ActivityManagerNative.getDefault.call(),
                    token,
                    false
            );

            // 1.将 ActivityRecorder 加入 mActivities 2.通知服务端 VAMS Activity 创建完成
            VActivityManager.get().onActivityCreate(ComponentUtils.toComponentName(info), caller, token, info, intent, ComponentUtils.getTaskAffinity(info), taskId, info.launchMode, info.flags);
            ClassLoader appClassLoader = VClientImpl.get().getClassLoader(info.applicationInfo);
            intent.setExtrasClassLoader(appClassLoader);
            // 将 Host Stub Activity Intent 替换为原版 Intent
            ActivityThread.ActivityClientRecord.intent.set(r, intent);
            // 同上
            ActivityThread.ActivityClientRecord.activityInfo.set(r, info);
            return true;
        }
```

需要注意的是，如果这个 Activity 是这个 Apk 启动的第一个组件，则需要 bindApplication 初始化 Application 操作:

```java
 private void bindApplicationNoCheck(String packageName, String processName, ConditionVariable lock) {
        mTempLock = lock;
        try {
            // 设置未捕获异常的 Callback
            setupUncaughtHandler();
        } catch (Throwable e) {
            e.printStackTrace();
        }
        try {
            // 修复 Provider 信息
            fixInstalledProviders();
        } catch (Throwable e) {
            e.printStackTrace();
        }
        mirror.android.os.Build.SERIAL.set(deviceInfo.serial);
        mirror.android.os.Build.DEVICE.set(Build.DEVICE.replace(" ", "_"));
        ActivityThread.mInitialApplication.set(
                VirtualCore.mainThread(),
                null
        );
        // 从 VPMS 获取 apk 信息
        AppBindData data = new AppBindData();
        InstalledAppInfo info = VirtualCore.get().getInstalledAppInfo(packageName, 0);
        if (info == null) {
            new Exception("App not exist!").printStackTrace();
            Process.killProcess(0);
            System.exit(0);
        }
        // dex 优化的开关，dalvik 和 art 处理不同
        if (!info.dependSystem && info.skipDexOpt) {
            VLog.d(TAG, "Dex opt skipped.");
            if (VirtualRuntime.isArt()) {
                ARTUtils.init(VirtualCore.get().getContext());
                ARTUtils.setIsDex2oatEnabled(false);
            } else {
                DalvikUtils.init();
                DalvikUtils.setDexOptMode(DalvikUtils.OPTIMIZE_MODE_NONE);
            }
        }
        data.appInfo = VPackageManager.get().getApplicationInfo(packageName, 0, getUserId(vuid));
        data.processName = processName;
        data.providers = VPackageManager.get().queryContentProviders(processName, getVUid(), PackageManager.GET_META_DATA);
        Log.i(TAG, "Binding application " + data.appInfo.packageName + " (" + data.processName + ")");
        mBoundApplication = data;
        // 主要设置进程的名字
        VirtualRuntime.setupRuntime(data.processName, data.appInfo);
        int targetSdkVersion = data.appInfo.targetSdkVersion;
        if (targetSdkVersion < Build.VERSION_CODES.GINGERBREAD) {
            StrictMode.ThreadPolicy newPolicy = new StrictMode.ThreadPolicy.Builder(StrictMode.getThreadPolicy()).permitNetwork().build();
            StrictMode.setThreadPolicy(newPolicy);
        }
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.N) {
            if (mirror.android.os.StrictMode.sVmPolicyMask != null) {
                mirror.android.os.StrictMode.sVmPolicyMask.set(0);
            }
        }
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.LOLLIPOP && targetSdkVersion < Build.VERSION_CODES.LOLLIPOP) {
            mirror.android.os.Message.updateCheckRecycle.call(targetSdkVersion);
        }
        if (VASettings.ENABLE_IO_REDIRECT) {
            // IO 重定向
            startIOUniformer();
        }
        // hook native 函数
        NativeEngine.hookNative();
        Object mainThread = VirtualCore.mainThread();
        // 准备 dex 列表
        NativeEngine.startDexOverride();
        // 获得子 pkg 的 Context 前提是必须在系统中安装的（疑问？）
        Context context = createPackageContext(data.appInfo.packageName);
        // 设置虚拟机系统环境 临时文件夹 codeCacheDir
        System.setProperty("java.io.tmpdir", context.getCacheDir().getAbsolutePath());
        // oat 的 cache 目录
        File codeCacheDir;
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.M) {
            codeCacheDir = context.getCodeCacheDir();
        } else {
            codeCacheDir = context.getCacheDir();
        }
        // 硬件加速的 cache 目录
        if (Build.VERSION.SDK_INT < Build.VERSION_CODES.N) {
            if (HardwareRenderer.setupDiskCache != null) {
                HardwareRenderer.setupDiskCache.call(codeCacheDir);
            }
        } else {
            if (ThreadedRenderer.setupDiskCache != null) {
                ThreadedRenderer.setupDiskCache.call(codeCacheDir);
            }
        }
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.M) {
            if (RenderScriptCacheDir.setupDiskCache != null) {
                RenderScriptCacheDir.setupDiskCache.call(codeCacheDir);
            }
        } else if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.JELLY_BEAN) {
            if (RenderScript.setupDiskCache != null) {
                RenderScript.setupDiskCache.call(codeCacheDir);
            }
        }

        // 修复子 App 中 ActivityThread.AppBinderData 的参数，因为之前用的是在 Host 程序中注册的 Stub 的信息
        Object boundApp = fixBoundApp(mBoundApplication);
        mBoundApplication.info = ContextImpl.mPackageInfo.get(context);
        mirror.android.app.ActivityThread.AppBindData.info.set(boundApp, data.info);

        // 同样修复 targetSdkVersion 原来也是可 Host 程序一样的
        VMRuntime.setTargetSdkVersion.call(VMRuntime.getRuntime.call(), data.appInfo.targetSdkVersion);

        boolean conflict = SpecialComponentList.isConflictingInstrumentation(packageName);
        if (!conflict) {
            InvocationStubManager.getInstance().checkEnv(AppInstrumentation.class);
        }

        // 开始构建子程序包的 Application 对象，并且替换原来通过 Host Stub 生成的 mInitialApplication
        mInitialApplication = LoadedApk.makeApplication.call(data.info, false, null);
        mirror.android.app.ActivityThread.mInitialApplication.set(mainThread, mInitialApplication);
        ContextFixer.fixContext(mInitialApplication);
        if (data.providers != null) {
            // 注册 Providers
            installContentProviders(mInitialApplication, data.providers);
        }
        // 初始化锁开，异步调用的初始化函数可以返回了
        if (lock != null) {
            lock.open();
            mTempLock = null;
        }
        try {
            // 调用 Application.onCreate
            mInstrumentation.callApplicationOnCreate(mInitialApplication);
            InvocationStubManager.getInstance().checkEnv(HCallbackStub.class);
            if (conflict) {
                InvocationStubManager.getInstance().checkEnv(AppInstrumentation.class);
            }
            Application createdApp = ActivityThread.mInitialApplication.get(mainThread);
            if (createdApp != null) {
                mInitialApplication = createdApp;
            }
        } catch (Exception e) {
            if (!mInstrumentation.onException(mInitialApplication, e)) {
                throw new RuntimeException(
                        "Unable to create application " + mInitialApplication.getClass().getName()
                                + ": " + e.toString(), e);
            }
        }
        VActivityManager.get().appDoneExecuting();
    }

    private void setupUncaughtHandler() {
        ThreadGroup root = Thread.currentThread().getThreadGroup();
        while (root.getParent() != null) {
            root = root.getParent();
        }
        ThreadGroup newRoot = new RootThreadGroup(root);
        if (Build.VERSION.SDK_INT < Build.VERSION_CODES.N) {
            final List<ThreadGroup> groups = mirror.java.lang.ThreadGroup.groups.get(root);
            //noinspection SynchronizationOnLocalVariableOrMethodParameter
            synchronized (groups) {
                List<ThreadGroup> newGroups = new ArrayList<>(groups);
                newGroups.remove(newRoot);
                mirror.java.lang.ThreadGroup.groups.set(newRoot, newGroups);
                groups.clear();
                groups.add(newRoot);
                mirror.java.lang.ThreadGroup.groups.set(root, groups);
                for (ThreadGroup group : newGroups) {
                    mirror.java.lang.ThreadGroup.parent.set(group, newRoot);
                }
            }
        } else {
            final ThreadGroup[] groups = ThreadGroupN.groups.get(root);
            //noinspection SynchronizationOnLocalVariableOrMethodParameter
            synchronized (groups) {
                ThreadGroup[] newGroups = groups.clone();
                ThreadGroupN.groups.set(newRoot, newGroups);
                ThreadGroupN.groups.set(root, new ThreadGroup[]{newRoot});
                for (Object group : newGroups) {
                    ThreadGroupN.parent.set(group, newRoot);
                }
                ThreadGroupN.ngroups.set(root, 1);
            }
        }
    }
```

bindApplication 主要做了以下几个事情：
1. 从 VPMS 获取 APK 的信息，根据设置控制 Dex 优化的开关。
2. 调用 mirror.android.os.Process.setArgV0.call(processName); 设置进程的名称，如果不设置则还是 p0 p1 这样。
3. 做 nativeHook 主要 Hook 一些 native 的函数，主要是一些 IO 函数，包括文件访问重定向等等。
4. 准备一些 cache 临时文件夹。
5. 设置 AppBinderData，AppBinderData 内部包含了 ApplicationInfo 和 provider 信息等重要的 apk 信息。可以理解为 framework 所需要的关键数据结构。
6. 安装 ContentProvider
7. 初始化用户的 Application 对象，并通过 Instrumentation 调用其 onCreate，代表着 Client App 的生命周期正式开始。

最后成功从 StubActivity Intent 还原出来的原版 Intent 被继续交给原生的 AM：

```java
// 将 Host Stub Activity Intent 替换为原版 Intent
ActivityThread.ActivityClientRecord.intent.set(r, intent);
// 同上
ActivityThread.ActivityClientRecord.activityInfo.set(r, info);
```

最后，最后一个 Hook 点在 Instrumentation.callActivityOnCreate：

因为 AMS 实际上启动的是 StubActivity 的关系，真正的 Activity 的一些信息还不是其真正的信息，比如主题之类的，所以需要在这个时机修复一下，选择这个时间修复的原因也是因为 Activity 已经被 new 出来了，而且资源已经准备完毕。

```java
public void callActivityOnCreate(Activity activity, Bundle icicle) {
        VirtualCore.get().getComponentDelegate().beforeActivityCreate(activity);
        IBinder token = mirror.android.app.Activity.mToken.get(activity);
        ActivityClientRecord r = VActivityManager.get().getActivityRecord(token);
        // 替换 Activity 对象
        if (r != null) {
            r.activity = activity;
        }
        ContextFixer.fixContext(activity);
        ActivityFixer.fixActivity(activity);
        ActivityInfo info = null;
        if (r != null) {
            info = r.info;
        }
        // 设置主题和屏幕纵横控制
        if (info != null) {
            if (info.theme != 0) {
                activity.setTheme(info.theme);
            }
            if (activity.getRequestedOrientation() == ActivityInfo.SCREEN_ORIENTATION_UNSPECIFIED
                    && info.screenOrientation != ActivityInfo.SCREEN_ORIENTATION_UNSPECIFIED) {
                activity.setRequestedOrientation(info.screenOrientation);
            }
        }
        super.callActivityOnCreate(activity, icicle);
        VirtualCore.get().getComponentDelegate().afterActivityCreate(activity);
    }
```

# 8 原生 Service 创建过程

首先有必要了解一下原生 framework 对 Service 的创建，因为在 VA 中启动 Service 和 Activity 有很大的区别。

首先入口 ContextWrapper.startService():

```java
@Override
    public ComponentName startService(Intent service) {
        return mBase.startService(service);
    }
```

mBase 是 ContextImpl，所以调用到 ContextImpl.startService():

```java
 @Override
    public ComponentName startService(Intent service) {
        warnIfCallingFromSystemProcess();
        return startServiceCommon(service, mUser);
    }
    private ComponentName startServiceCommon(Intent service, UserHandle user) {
        try {
            validateServiceIntent(service);
            service.prepareToLeaveProcess(this);
            ComponentName cn = ActivityManagerNative.getDefault().startService(
                mMainThread.getApplicationThread(), service, service.resolveTypeIfNeeded(
                            getContentResolver()), getOpPackageName(), user.getIdentifier());
            if (cn != null) {
                if (cn.getPackageName().equals("!")) {
                    throw new SecurityException(
                            "Not allowed to start service " + service
                            + " without permission " + cn.getClassName());
                } else if (cn.getPackageName().equals("!!")) {
                    throw new SecurityException(
                            "Unable to start service " + service
                            + ": " + cn.getClassName());
                }
            }
            return cn;
        } catch (RemoteException e) {
            throw e.rethrowFromSystemServer();
        }
    }
```

Client 的流程最后到 ActivityManagerNative.startService():

```java
public int startActivity(IApplicationThread caller, String callingPackage, Intent intent,
            String resolvedType, IBinder resultTo, String resultWho, int requestCode,
            int startFlags, ProfilerInfo profilerInfo, Bundle options) throws RemoteException {
        Parcel data = Parcel.obtain();
        Parcel reply = Parcel.obtain();
        data.writeInterfaceToken(IActivityManager.descriptor);
        data.writeStrongBinder(caller != null ? caller.asBinder() : null);
        data.writeString(callingPackage);
        intent.writeToParcel(data, 0);
        data.writeString(resolvedType);
        data.writeStrongBinder(resultTo);
        data.writeString(resultWho);
        data.writeInt(requestCode);
        data.writeInt(startFlags);
        if (profilerInfo != null) {
            data.writeInt(1);
            profilerInfo.writeToParcel(data, Parcelable.PARCELABLE_WRITE_RETURN_VALUE);
        } else {
            data.writeInt(0);
        }
        if (options != null) {
            data.writeInt(1);
            options.writeToParcel(data, 0);
        } else {
            data.writeInt(0);
        }
        // 远程调用 AMS
        mRemote.transact(START_ACTIVITY_TRANSACTION, data, reply, 0);
        reply.readException();
        int result = reply.readInt();
        reply.recycle();
        data.recycle();
        return result;
    }
```

不出所料，逻辑再一次转移到远程的 AMS 中，然后我们先忽略 AMS 中的一堆逻辑，最后 AMS 调用到这:

```java
    private final void realStartServiceLocked(ServiceRecord r,
            ProcessRecord app, boolean execInFg) throws RemoteException {
            app.thread.scheduleCreateService(r, r.serviceInfo,
                    mAm.compatibilityInfoForPackageLocked(r.serviceInfo.applicationInfo),
                    app.repProcState);
            sendServiceArgsLocked(r, execInFg, true);
    }
```

这里的 ProcessRecoder.thread 是 AMS 持有的 Client App 的 IBinder 句柄，通过他可以远程调用到 Client App 的 ApplicationThread 中的 scheduleCreateService 方法：

```java
public final void scheduleCreateService(IBinder token,
                ServiceInfo info, CompatibilityInfo compatInfo, int processState) {
            updateProcessState(processState, false);
            CreateServiceData s = new CreateServiceData();
            s.token = token;
            s.info = info;
            s.compatInfo = compatInfo;

            sendMessage(H.CREATE_SERVICE, s);
        }
```

这里大家可能发现了，Intent 在 AMS 绕了一圈又回来了，事实上 AMS 在其中好像没有发挥什么作用，其实在外部环境 AMS 还是很重要的，但是在 VA 中，AMS 在 Service 调度中其实没有发挥什么作用。原因有以下几点：

1. 首先 VA 内部的插件 Service 没有比较暴露给外部 App 调用，所以让 AMS 知晓 Service 的意义不大。

2. 其次和 Activity 必须有个 StubActivity 让 AMS 持有不一样，Service 生命周期和功能都极其简单，并且没有界面，没有交互，换句话说 Service 和其他 Framework Service(例如 WMS) 没有任何关系，所以其实并不需要 AMS 这一步存在。

   那么综上所诉，为了启动插件 Service 我们其实可以绕过 AMS，直接调用 ApplicationThread 中的 scheduleCreateService 方法，Service 的会话储存交给 VAMS 就行。

# 9 startService 的实现

和 startActivity 一样，首先是 Hook：

```java
static class StartService extends MethodProxy {

        @Override
        public String getMethodName() {
            return "startService";
        }

        @Override
        public Object call(Object who, Method method, Object... args) throws Throwable {
            IInterface appThread = (IInterface) args[0];
            Intent service = (Intent) args[1];
            String resolvedType = (String) args[2];
            if (service.getComponent() != null
                    && getHostPkg().equals(service.getComponent().getPackageName())) {
                // for server process
                return method.invoke(who, args);
            }
            int userId = VUserHandle.myUserId();
            // 如果是内部请求, 获取原来的 Service
            if (service.getBooleanExtra("_VA_|_from_inner_", false)) {
                userId = service.getIntExtra("_VA_|_user_id_", userId);
                service = service.getParcelableExtra("_VA_|_intent_");
            } else {
                if (isServerProcess()) {
                    userId = service.getIntExtra("_VA_|_user_id_", VUserHandle.USER_NULL);
                }
            }
            service.setDataAndType(service.getData(), resolvedType);
            ServiceInfo serviceInfo = VirtualCore.get().resolveServiceInfo(service, VUserHandle.myUserId());

            if (serviceInfo != null) {
                // 远程调用 VAMS.startService()
                return VActivityManager.get().startService(appThread, service, resolvedType, userId);
            }
            return method.invoke(who, args);
        }

        @Override
        public boolean isEnable() {
            return isAppProcess() || isServerProcess();
        }
    }
```

和 startActivity 一样将真正业务交给 VAMS.startService():

```java
 private ComponentName startServiceCommon(Intent service,
                                             boolean scheduleServiceArgs, int userId) {
        ServiceInfo serviceInfo = resolveServiceInfo(service, userId);
        if (serviceInfo == null) {
            return null;
        }
        ProcessRecord targetApp = startProcessIfNeedLocked(ComponentUtils.getProcessName(serviceInfo),
                userId,
                serviceInfo.packageName);

        if (targetApp == null) {
            VLog.e(TAG, "Unable to start new Process for : " + ComponentUtils.toComponentName(serviceInfo));
            return null;
        }
        IInterface appThread = targetApp.appThread;
        ServiceRecord r = findRecordLocked(userId, serviceInfo);
        boolean needCreateService = false;
        if (r == null) {
            r = new ServiceRecord();
            r.name = new ComponentName(serviceInfo.packageName, serviceInfo.name);
            r.startId = 0;
            r.activeSince = SystemClock.elapsedRealtime();
            r.process = targetApp;
            r.serviceInfo = serviceInfo;
            needCreateService = true;
        } else {
            if (r.process == null) {
                r.process = targetApp;
                needCreateService = true;
            }
        }

        // 如果 service 尚未创建
        if (needCreateService) {
            try {
                // 调用 ApplicationThread.scheduleCreateService 直接创建 Service
                IApplicationThreadCompat.scheduleCreateService(appThread, r, r.serviceInfo, 0);
            } catch (RemoteException e) {
                e.printStackTrace();
            }

            // Note: If the service has been called for not AUTO_CREATE binding, the corresponding
            // ServiceRecord is already in mHistory, so we use Set to replace List to avoid add
            // ServiceRecord twice
            // 将 ServiceRecorder 推入 history
            addRecord(r);

            // 等待 bindService，如果是通过 bindService 自动创建的 Service，在创建 Service 完成后会进入 bindService 流程
            requestServiceBindingsLocked(r);
        }

        r.lastActivityTime = SystemClock.uptimeMillis();
        if (scheduleServiceArgs) {
            r.startId++;
            boolean taskRemoved = serviceInfo.applicationInfo != null
                    && serviceInfo.applicationInfo.targetSdkVersion < Build.VERSION_CODES.ECLAIR;
            try {
                IApplicationThreadCompat.scheduleServiceArgs(appThread, r, taskRemoved, r.startId, 0, service);
            } catch (RemoteException e) {
                e.printStackTrace();
            }
        }
        return ComponentUtils.toComponentName(serviceInfo);
    }
```

这里主要做了以下几个工作：

1. 和 Activity 创建的时候一样，调用 startProcessIfNeedLocked 检查 Application 是否初始化，没有则开始初始化 Application 流程。
2. 准备 ServiceRecord 和 ServiceInfo。
3. 如果 service 还没有创建，则直接调用 ApplicationThread.scheduleCreateService 创建 Service，可以看出这里直接跳过了 AMS。
4. 将 ServiceRecord 记录到 Service 列表，等待 bindService，如果是通过 bindService 自动创建的 Service，在创建 Service 完成后会进入 bindService 流程。

同样的 bindService 也是直接调用系统的 ApplicationThread.scheduleBindService

好了，由于 Service 的特点，startService 看上去比 startActivity 简单多了。接下来要分析的是 BroacastReceiver。

# 10 方案猜测

同 Activity 一样，Client App 在 Menifest 中注册的静态广播外部 AMS 是无法知晓的，经过前几章的分析，相信大家已经是老司机了，我们可以先尝试提出自己的观点。
1. 和 Activity 一样使用 Stub 组件占坑？仔细想一想是无法实现的，因为你无法预先确定 Client App 中广播的 Intent Filter。
2. 使用动态注册即 context.registerBroadcastReceiver 代替静态注册，这确实是个好主意，但是重点在于注册的时机。我们需要在 VAService 启动时就预先把 VA 中所有安装的 Client App 的静态 Receiver 提前注册好，事实上外部 AMS 也是这么做的。不然的话，没有打开的 App 就无法收到广播了。

# 11 VA 静态广播注册

从前面我们知道，VAService 的启动时机实在 BinderProvider.onCreate():

```java
@Override
    public boolean onCreate() {
        .....................
        VAppManagerService.get().scanApps();
        .....................
        return true;
    }
```

看到 VAppManagerService.get().scanApps()————>PersistenceLayer.read()——————->PackagePersistenceLayer.readPersistenceData()——————>VAppManagerService.loadPackage()———->VAppManagerService.loadPackageInnerLocked()————–>BroadcastSystem.startUp();

```java
// 静态 Receiver 的注册
    public void startApp(VPackage p) {
        PackageSetting setting = (PackageSetting) p.mExtras;
        // 遍历 Client App 的 Receiver
        for (VPackage.ActivityComponent receiver : p.receivers) {
            ActivityInfo info = receiver.info;
            // 得到对应 Client App 在 VAService 中的记录列表
            List<BroadcastReceiver> receivers = mReceivers.get(p.packageName);
            if (receivers == null) {
                receivers = new ArrayList<>();
                mReceivers.put(p.packageName, receivers);
            }
            // 注册显式意图
            String componentAction = String.format("_VA_%s_%s", info.packageName, info.name);
            IntentFilter componentFilter = new IntentFilter(componentAction);
            BroadcastReceiver r = new StaticBroadcastReceiver(setting.appId, info, componentFilter);
            mContext.registerReceiver(r, componentFilter, null, mScheduler);
            // 推入记录
            receivers.add(r);
            // 遍历注册隐式意图
            for (VPackage.ActivityIntentInfo ci : receiver.intents) {
                IntentFilter cloneFilter = new IntentFilter(ci.filter);
                SpecialComponentList.protectIntentFilter(cloneFilter);
                r = new StaticBroadcastReceiver(setting.appId, info, cloneFilter);
                mContext.registerReceiver(r, cloneFilter, null, mScheduler);
                // 推入记录
                receivers.add(r);
            }
        }
    }
```

这里对每个 Client App 静态 Receiver 的信息使用统一的代理 StaticBroadcastReceiver 注册。
1. 首先注册 Receiver 的显式意图，每个显式意图被重定向成格式为 “_VA_PKGNAME_CLASSNAME”的 componentAction 这么做的理由实际是真正注册是 VAService 进程空间的 StaticBroadcastReceiver 代理 Receiver，而不是 VA Client App 进程空间，所以直接注册 VA Client App 中的真实类名是没有意义的，这样通过 VAService 代理然后再从 Intent 中取出的 “_VA_PKGNAME_CLASSNAME”到 VA Client 中找到真正的 Receiver，这个逻辑和 Activity 的处理有些相似。
2. 然后就是遍历 Intent-Filter，每个 Intent-Filter 注册一个 StaticBroadcastReceiver 代理。

这样我们的代理 Receiver 注册完毕了。

下面当代理 Receiver 收到广播时：

```java
 @Override
        public void onReceive(Context context, Intent intent) {
            if (mApp.isBooting()) {
                return;
            }
            if ((intent.getFlags() & FLAG_RECEIVER_REGISTERED_ONLY) != 0 || isInitialStickyBroadcast()) {
                return;
            }
            String privilegePkg = intent.getStringExtra("_VA_|_privilege_pkg_");
            if (privilegePkg != null && !info.packageName.equals(privilegePkg)) {
                return;
            }
            PendingResult result = goAsync();
            if (!mAMS.handleStaticBroadcast(appId, info, intent, new PendingResultData(result))) {
                result.finish();
            }
        }
```

然后看到 handleStaticBroadcast

```java
boolean handleStaticBroadcast(int appId, ActivityInfo info, Intent intent,
                                  PendingResultData result) {
        // 这里是取出真正的目标 Intent
        Intent realIntent = intent.getParcelableExtra("_VA_|_intent_");
        // 取出真正的目标 component
        ComponentName component = intent.getParcelableExtra("_VA_|_component_");
        // 用户 id
        int userId = intent.getIntExtra("_VA_|_user_id_", VUserHandle.USER_NULL);
        if (realIntent == null) {
            return false;
        }
        if (userId < 0) {
            VLog.w(TAG, "Sent a broadcast without userId " + realIntent);
            return false;
        }
        int vuid = VUserHandle.getUid(userId, appId);
        return handleUserBroadcast(vuid, info, component, realIntent, result);
    }
```

注意这里取出了真正的 Intent，和 Activity 类似，但是和 Activity 处理不同的是现在的逻辑还在 VAService 中：

然后 handleUserBroadcast()———–>handleStaticBroadcastAsUser()————>performScheduleReceiver():

```java
private void performScheduleReceiver(IVClient client, int vuid, ActivityInfo info, Intent intent,
                                         PendingResultData result) {

        ComponentName componentName = ComponentUtils.toComponentName(info);
        BroadcastSystem.get().broadcastSent(vuid, info, result);
        try {
            // 远程调用 client app 的 scheduleReceiver
            client.scheduleReceiver(info.processName, componentName, intent, result);
        } catch (Throwable e) {
            if (result != null) {
                result.finish();
            }
        }
    }
```

client.scheduleReceiver() 这时候远程调用了 Client App 的 scheduleReceiver。这样我们回到了 Client App 进程空间：

```java
@Override
    public void scheduleReceiver(String processName, ComponentName component, Intent intent, PendingResultData resultData) {
        ReceiverData receiverData = new ReceiverData();
        receiverData.resultData = resultData;
        receiverData.intent = intent;
        receiverData.component = component;
        receiverData.processName = processName;
        sendMessage(RECEIVER, receiverData);
    }
```

跟到消息队列中：

```java
case RECEIVER: {
     handleReceiver((ReceiverData) msg.obj);
}123
private void handleReceiver(ReceiverData data) {
        BroadcastReceiver.PendingResult result = data.resultData.build();
        try {
            // 依然是检测 Application 是否初始化，没有则初始化
            if (!isBound()) {
                bindApplication(data.component.getPackageName(), data.processName);
            }
            // 获取 Receiver 的 Context，这个context是一个ReceiverRestrictedContext实例，它有两个主要函数被禁掉：registerReceiver()和 bindService()。这两个函数在BroadcastReceiver.onReceive()不允许调用。每次Receiver处理一个广播，传递进来的context都是一个新的实例。
            Context context = mInitialApplication.getBaseContext();
            Context receiverContext = ContextImpl.getReceiverRestrictedContext.call(context);
            String className = data.component.getClassName();
            // 实例化目标 Receiver
            BroadcastReceiver receiver = (BroadcastReceiver) context.getClassLoader().loadClass(className).newInstance();
            mirror.android.content.BroadcastReceiver.setPendingResult.call(receiver, result);
            data.intent.setExtrasClassLoader(context.getClassLoader());
            // 手动调用 onCreate
            receiver.onReceive(receiverContext, data.intent);
            // 通知 Pending 结束
            if (mirror.android.content.BroadcastReceiver.getPendingResult.call(receiver) != null) {
                result.finish();
            }
        } catch (Exception e) {
            e.printStackTrace();
            throw new RuntimeException(
                    "Unable to start receiver " + data.component
                            + ": " + e.toString(), e);
        }
        // 这里需要远程通知 VAService 广播已送到
        VActivityManager.get().broadcastFinish(data.resultData);
    }

```

这里就是最关键的地方了，简单点概括就是 new 了真正的 Receiver 然后调用 onCreate 而已。Receiver 生命周期真的非常简单。

需要注意的是，broadCast 发送有个超时机制：

```java
void broadcastFinish(PendingResultData res) {
        synchronized (mBroadcastRecords) {
            BroadcastRecord record = mBroadcastRecords.remove(res.mToken);
            if (record == null) {
                VLog.e(TAG, "Unable to find the BroadcastRecord by token: " + res.mToken);
            }
        }
        mTimeoutHandler.removeMessages(0, res.mToken);
        res.finish();
    }12345678910
private final class TimeoutHandler extends Handler {
        @Override
        public void handleMessage(Message msg) {
            IBinder token = (IBinder) msg.obj;
            BroadcastRecord r = mBroadcastRecords.remove(token);
            if (r != null) {
                VLog.w(TAG, "Broadcast timeout, cancel to dispatch it.");
                r.pendingResult.finish();
            }
        }
    }
```

这里如果广播超时则会通知 PendingResult 结束，告诉发送方广播结束了。

# 12 发送广播的处理

其实上一部分已经讲了很多发送广播的处理了。
这里 Hook 了 broacastIntent 方法:

```java
static class BroadcastIntent extends MethodProxy {

        @Override
        public String getMethodName() {
            return "broadcastIntent";
        }

        @Override
        public Object call(Object who, Method method, Object... args) throws Throwable {
            Intent intent = (Intent) args[1];
            String type = (String) args[2];
            intent.setDataAndType(intent.getData(), type);
            if (VirtualCore.get().getComponentDelegate() != null) {
                VirtualCore.get().getComponentDelegate().onSendBroadcast(intent);
            }
            Intent newIntent = handleIntent(intent);
            if (newIntent != null) {
                args[1] = newIntent;
            } else {
                return 0;
            }

            if (args[7] instanceof String || args[7] instanceof String[]) {
                // clear the permission
                args[7] = null;
            }
            return method.invoke(who, args);
        }


        private Intent handleIntent(final Intent intent) {
            final String action = intent.getAction();
            if ("android.intent.action.CREATE_SHORTCUT".equals(action)
                    || "com.android.launcher.action.INSTALL_SHORTCUT".equals(action)) {

                return VASettings.ENABLE_INNER_SHORTCUT ? handleInstallShortcutIntent(intent) : null;

            } else if ("com.android.launcher.action.UNINSTALL_SHORTCUT".equals(action)) {

                handleUninstallShortcutIntent(intent);

            } else {
                return ComponentUtils.redirectBroadcastIntent(intent, VUserHandle.myUserId());
            }
            return intent;
        }

        private Intent handleInstallShortcutIntent(Intent intent) {
            Intent shortcut = intent.getParcelableExtra(Intent.EXTRA_SHORTCUT_INTENT);
            if (shortcut != null) {
                ComponentName component = shortcut.resolveActivity(VirtualCore.getPM());
                if (component != null) {
                    String pkg = component.getPackageName();
                    Intent newShortcutIntent = new Intent();
                    newShortcutIntent.setClassName(getHostPkg(), Constants.SHORTCUT_PROXY_ACTIVITY_NAME);
                    newShortcutIntent.addCategory(Intent.CATEGORY_DEFAULT);
                    newShortcutIntent.putExtra("_VA_|_intent_", shortcut);
                    newShortcutIntent.putExtra("_VA_|_uri_", shortcut.toUri(0));
                    newShortcutIntent.putExtra("_VA_|_user_id_", VUserHandle.myUserId());
                    intent.removeExtra(Intent.EXTRA_SHORTCUT_INTENT);
                    intent.putExtra(Intent.EXTRA_SHORTCUT_INTENT, newShortcutIntent);

                    Intent.ShortcutIconResource icon = intent.getParcelableExtra(Intent.EXTRA_SHORTCUT_ICON_RESOURCE);
                    if (icon != null && !TextUtils.equals(icon.packageName, getHostPkg())) {
                        try {
                            Resources resources = VirtualCore.get().getResources(pkg);
                            int resId = resources.getIdentifier(icon.resourceName, "drawable", pkg);
                            if (resId > 0) {
                                //noinspection deprecation
                                Drawable iconDrawable = resources.getDrawable(resId);
                                Bitmap newIcon = BitmapUtils.drawableToBitmap(iconDrawable);
                                if (newIcon != null) {
                                    intent.removeExtra(Intent.EXTRA_SHORTCUT_ICON_RESOURCE);
                                    intent.putExtra(Intent.EXTRA_SHORTCUT_ICON, newIcon);
                                }
                            }
                        } catch (Throwable e) {
                            e.printStackTrace();
                        }
                    }
                }
            }
            return intent;
        }
```

1. 这里拦截了创建快捷图标的 Intent，这是一个发给 Launcher 的隐式广播，VA 把这个请求拦截下来因为如果不拦截这个快捷方式就会指向外部的 App，并且如果外部 App 没有安装，此广播也不会发生作用。VA 把这个广播换成了自己的逻辑。
2. 注意 ComponentUtils.redirectBroadcastIntent()， 类似 Activity 用代理 Intent 包裹真正的 Intent：

```java
public static Intent redirectBroadcastIntent(Intent intent, int userId) {
        Intent newIntent = intent.cloneFilter();
        newIntent.setComponent(null);
        newIntent.setPackage(null);
        ComponentName component = intent.getComponent();
        String pkg = intent.getPackage();
        if (component != null) {
            newIntent.putExtra("_VA_|_user_id_", userId);
            // 这里显式意图被重定位成 _VA_PKGNAME_CLASSNAME 的格式，与前面注册的时候对应
            newIntent.setAction(String.format("_VA_%s_%s", component.getPackageName(), component.getClassName()));
            newIntent.putExtra("_VA_|_component_", component);
            newIntent.putExtra("_VA_|_intent_", new Intent(intent));
        } else if (pkg != null) {
            newIntent.putExtra("_VA_|_user_id_", userId);
            newIntent.putExtra("_VA_|_creator_", pkg);
            newIntent.putExtra("_VA_|_intent_", new Intent(intent));
            String protectedAction = SpecialComponentList.protectAction(intent.getAction());
            if (protectedAction != null) {
                newIntent.setAction(protectedAction);
            }
        } else {
            newIntent.putExtra("_VA_|_user_id_", userId);
            newIntent.putExtra("_VA_|_intent_", new Intent(intent));
            String protectedAction = SpecialComponentList.protectAction(intent.getAction());
            if (protectedAction != null) {
                newIntent.setAction(protectedAction);
            }
        }
        return newIntent;
    }
```

Ok, BroadcastReceiver 完毕，下一张分析最后一个组件 ContentProvider。

# 13 Provider 注册

回顾前面，Activity 启动的时候会检查 Application 是否初始化，会调用 bindApplication，里面执行了安装 Provider 的方法：

```java
private void installContentProviders(Context app, List<ProviderInfo> providers) {
        long origId = Binder.clearCallingIdentity();
        Object mainThread = VirtualCore.mainThread();
        try {
            for (ProviderInfo cpi : providers) {
                if (cpi.enabled) {
                    ActivityThread.installProvider(mainThread, app, cpi, null);
                }
            }
        } finally {
            Binder.restoreCallingIdentity(origId);
        }
    }
```

这里很简单，调用 ActivityThread.installProvider() 注册就这么完成了。

但是，仔细一想事情没那么简单，按照这个逻辑 ContentProvider 是在 Application 启动的时候注册的，那么如果 Application 没有启动，那么自然就没有注册了，这样的话其他 App 怎么找到 Provider 呢？

Activity 是因为有在 VA 的 Menifest 里注册的 StubActivity，这样启动 StubActivity 自然就启动了在“：p(n)”进程。那么对应的，VA 用了 StubContentProvider 么？确实是有的。

但是请不要误会了，这个注册在 “p(n)”进程的 StubContentProvider(继承自 StubContentProvider 的 C1，C2，Cn……) 并不是 StubActivity 那样的为了给插件 Provider 占坑的 Stub 组件。

StubContentProvider 的真正目的是为了让 AMS 通过 system_process 带起 “p(n)”进程，然后 VAMS 用过远程调用 StubProvider.call() 回插件 IClient 的 IBinder 句柄给 VAMS 持有。这样 VAMS 就可以远程调用插件进程 “p(n)”中的方法了。

事实上在前面第二张 App 启动时就有说过相关内容，但是为了避免大家误解还是有必要重复一下。

```java
public class StubContentProvider extends ContentProvider {

    @Override
    public boolean onCreate() {
        return true;
    }

    @Override
    public Bundle call(String method, String arg, Bundle extras) {
        if ("_VA_|_init_process_".equals(method)) {
            return initProcess(extras);
        }
        return null;
    }

    private Bundle initProcess(Bundle extras) {
        ConditionVariable lock = VirtualCore.get().getInitLock();
        if (lock != null) {
            lock.block();
        }
        IBinder token = BundleCompat.getBinder(extras, "_VA_|_binder_");
        int vuid = extras.getInt("_VA_|_vuid_");
        VClientImpl client = VClientImpl.get();
        client.initProcess(token, vuid);
        Bundle res = new Bundle();
        BundleCompat.putBinder(res, "_VA_|_client_", client.asBinder());
        res.putInt("_VA_|_pid_", Process.myPid());
        return res;
    }
```

# 14 getContentProvider

这里依然 Hook 了 getContentProvider 方法：

```java
 static class GetContentProvider extends MethodProxy {

        @Override
        public String getMethodName() {
            return "getContentProvider";
        }

        @Override
        public Object call(Object who, Method method, Object... args) throws Throwable {
            int nameIdx = getProviderNameIndex();
            String name = (String) args[nameIdx];
            int userId = VUserHandle.myUserId();
            // 远程调用 VPMS 从 VPackage 拿到 ProviderInfo
            ProviderInfo info = VPackageManager.get().resolveContentProvider(name, 0, userId);
            // 拿不到说明目标 App 未启动
            if (info != null && info.enabled && isAppPkg(info.packageName)) {
                // 远程调用 VAMS，然后 VAMS 再通过 AMS 远程调用注册在插件进程的 StubContentProvider.call 初始化插件进程
                int targetVPid = VActivityManager.get().initProcess(info.packageName, info.processName, userId);
                if (targetVPid == -1) {
                    return null;
                }
                args[nameIdx] = VASettings.getStubAuthority(targetVPid);
                Object holder = method.invoke(who, args);
                if (holder == null) {
                    return null;
                }
                // ContentProviderHolder 有两个成员变量provider、connection，provider 是目标 Provider 的 IBinder 句柄。
                // connection 则是 callback
                if (BuildCompat.isOreo()) {
                    IInterface provider = ContentProviderHolderOreo.provider.get(holder);
                    if (provider != null) {
                        // 这里是重点，远程调用了 VAMS 的 acquireProviderClient
                        provider = VActivityManager.get().acquireProviderClient(userId, info);
                    }
                    ContentProviderHolderOreo.provider.set(holder, provider);
                    ContentProviderHolderOreo.info.set(holder, info);
                } else {
                    IInterface provider = IActivityManager.ContentProviderHolder.provider.get(holder);
                    if (provider != null) {
                        provider = VActivityManager.get().acquireProviderClient(userId, info);
                    }
                    IActivityManager.ContentProviderHolder.provider.set(holder, provider);
                    IActivityManager.ContentProviderHolder.info.set(holder, info);
                }
                return holder;
            }
            Object holder = method.invoke(who, args);
            if (holder != null) {
                if (BuildCompat.isOreo()) {
                    IInterface provider = ContentProviderHolderOreo.provider.get(holder);
                    info = ContentProviderHolderOreo.info.get(holder);
                    if (provider != null) {
                        provider = ProviderHook.createProxy(true, info.authority, provider);
                    }
                    ContentProviderHolderOreo.provider.set(holder, provider);
                } else {
                    IInterface provider = IActivityManager.ContentProviderHolder.provider.get(holder);
                    info = IActivityManager.ContentProviderHolder.info.get(holder);
                    if (provider != null) {
                        provider = ProviderHook.createProxy(true, info.authority, provider);
                    }
                    IActivityManager.ContentProviderHolder.provider.set(holder, provider);
                }
                return holder;
            }
            return null;
        }


        public int getProviderNameIndex() {
            return 1;
        }

        @Override
        public boolean isEnable() {
            return isAppProcess();
        }
    }
```

这里主要做了几件事情：

1. 通过 VPMS 拿到解析好的目标 ProviderInfo，启动目标 Provider 所在的进程，怎么启动？远程调用 VAMS，然后 VAMS 再通过 AMS 远程调用注册在插件进程的 StubContentProvider.call 初始化插件进程。这时候 VAMS 就会持有目标插件进程的 IClient 句柄，以备后续调用。
2. 准备 ContentProviderHolder 相关的事情，ContentProviderHolder 有两个成员变量provider、connection，provider 是目标 Provider 的 IBinder 句柄。
3. 远程调用了 VAMS 的 acquireProviderClient(), 将任务抛给了远端的 VAMS。

下面就来看一下 VAMS.acquireProviderClient()

```java
@Override
    public IBinder acquireProviderClient(int userId, ProviderInfo info) {
        ProcessRecord callerApp;
        synchronized (mPidsSelfLocked) {
            callerApp = findProcessLocked(VBinder.getCallingPid());
        }
        if (callerApp == null) {
            throw new SecurityException("Who are you?");
        }
        String processName = info.processName;
        ProcessRecord r;
        synchronized (this) {
            r = startProcessIfNeedLocked(processName, userId, info.packageName);
        }
        if (r != null && r.client.asBinder().isBinderAlive()) {
            try {
                return r.client.acquireProviderClient(info);
            } catch (RemoteException e) {
                e.printStackTrace();
            }
        }
        return null;
    }
```

可以看到调用了 r.client.acquireProviderClient(info);
r.client 就是前面 initProcess 的时候保存下来的插件进程的 IClient 句柄，那么等于远程调用到了插件的 VClientImpl.acquireProviderClient():

注意现在开始流程是在目标 Provider 的 Client App 进程中

```java
@Override
    public IBinder acquireProviderClient(ProviderInfo info) {
        if (mTempLock != null) {
            mTempLock.block();
        }
        // 这里检查 Application 是否启动，注意注册 Provider 的逻辑也在里面
        if (!isBound()) {
            VClientImpl.get().bindApplication(info.packageName, info.processName);
        }
        // 准备 ContentProviderClient
        IInterface provider = null;
        String[] authorities = info.authority.split(";");
        String authority = authorities.length == 0 ? info.authority : authorities[0];
        ContentResolver resolver = VirtualCore.get().getContext().getContentResolver();
        ContentProviderClient client = null;
        try {
            if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.JELLY_BEAN) {
                client = resolver.acquireUnstableContentProviderClient(authority);
            } else {
                client = resolver.acquireContentProviderClient(authority);
            }
        } catch (Throwable e) {
            e.printStackTrace();
        }
        if (client != null) {
            // 反射获取 provider
            provider = mirror.android.content.ContentProviderClient.mContentProvider.get(client);
            client.release();
        }
        return provider != null ? provider.asBinder() : null;
    }
```

这里终于看到了调用 bindApplication 的地方，如果 provider 尚未注册，那么这里将会注册 provider。

最终通过反射 android.content.ContentProviderClient.mContentProvider 获取到了目标 provider 的句柄，然后 provider 沿着目标 Client App 进程——> VAMS 进程 ———-> 回到了调用 Client App 进程中，整个获取 provider 的过程正式完成。

到此，四大组件的代理全部完成，本次分析也基本结束，下一章会写一些总结和补充内容。