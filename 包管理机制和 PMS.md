## 包管理机制和 PMS

### 一、 PackageManager 简介

![PackageManager](https://raw.githubusercontent.com/huanggenghg/huanggenghg/main/res/PackageManager.drawio.png)

### 二、PackageInstaller 入口

```java
// before Android 7.0 安装指定路径下的 apk
Intent intent = new Intent(Intent.ACTON_VIEW);
intent.addFlags(Intent.FLAG_ACTIVITY_NEW_TASK);
intent.setDataAndType(Uri.parse("file://" + path), "application/vnd.android.package-archive");
context.startActivity(intent);

// Android 7.0 and after should use FileProvider，可隐藏共享文件的真实路径，并将其转化成"content://Uri"
```

此上隐式匹配的 Activity 为 InstallerStart（Android 7.0 为 PackageInstallerActivity）

```java
            Uri packageUri = intent.getData();

            if (packageUri != null && packageUri.getScheme().equals(
                    ContentResolver.SCHEME_CONTENT)) {
                // [IMPORTANT] This path is deprecated, but should still work. Only necessary
                // features should be added.

                // Copy file to prevent it from being changed underneath this process
                nextActivity.setClass(this, InstallStaging.class);
            } else if (packageUri != null && packageUri.getScheme().equals(
                    PackageInstallerActivity.SCHEME_PACKAGE)) {
                nextActivity.setClass(this, PackageInstallerActivity.class);
            } else {
                Intent result = new Intent();
                result.putExtra(Intent.EXTRA_INSTALL_RESULT,
                        PackageManager.INSTALL_FAILED_INVALID_URI);
                setResult(RESULT_FIRST_USER, result);

                nextActivity = null;
            }
```

以以上应用场景，进入的是 InstallStaging，查看 InstallStaging 查看源码，详细读者可自行去查看，可知 InstallStaging 主要起转换作用，通过 StagingAsyncTask 将 content 协议的 Uri 转换成 file 协议，然后跳转到 PackageInstallerActivity 中，这样就可以像之前版本（before Android 7.0）一样启动安装流程。

![PackageInstaller 入口](https://raw.githubusercontent.com/huanggenghg/huanggenghg/main/res/PackageInstaller%20%E5%85%A5%E5%8F%A3.drawio.png)

由上可知，PackageInstallerActivity 才是安装器 PackageInstaller 真正的入口 Activity。

```java
    /**
     * Parse the Uri and set up the installer for this package.
     *
     * @param packageUri The URI to parse
     *
     * @return {@code true} iff the installer could be set up
     */
    private boolean processPackageUri(final Uri packageUri) {
        mPackageURI = packageUri;

        final String scheme = packageUri.getScheme();

        switch (scheme) {
            case SCHEME_PACKAGE: {
                try {
                    mPkgInfo = mPm.getPackageInfo(packageUri.getSchemeSpecificPart(),
                            PackageManager.GET_PERMISSIONS
                                    | PackageManager.MATCH_UNINSTALLED_PACKAGES);
                } catch (NameNotFoundException e) {
                }
                if (mPkgInfo == null) {
                    Log.w(TAG, "Requested package " + packageUri.getScheme()
                            + " not available. Discontinuing installation");
                    showDialogInner(DLG_PACKAGE_ERROR);
                    setPmResult(PackageManager.INSTALL_FAILED_INVALID_APK);
                    return false;
                }
                mAppSnippet = new PackageUtil.AppSnippet(mPm.getApplicationLabel(mPkgInfo.applicationInfo),
                        mPm.getApplicationIcon(mPkgInfo.applicationInfo));
            } break;

            case ContentResolver.SCHEME_FILE: {
                File sourceFile = new File(packageUri.getPath());
                PackageParser.Package parsed = PackageUtil.getPackageInfo(this, sourceFile); 

                // Check for parse errors
                if (parsed == null) {
                    Log.w(TAG, "Parse error when parsing manifest. Discontinuing installation");
                    showDialogInner(DLG_PACKAGE_ERROR);
                    setPmResult(PackageManager.INSTALL_FAILED_INVALID_APK);
                    return false;
                }
                mPkgInfo = PackageParser.generatePackageInfo(parsed, null,
                        PackageManager.GET_PERMISSIONS, 0, 0, null,
                        new PackageUserState());
                mAppSnippet = PackageUtil.getAppSnippet(this, mPkgInfo.applicationInfo, sourceFile);
            } break;

            default: {
                throw new IllegalArgumentException("Unexpected URI scheme " + packageUri);
            }
        }

        return true;
    }
```

内部解析 apk 文件信息的主要类是 PackageParser 及 PackageUtil，得到 apk 文件的所有信息。

在最后的 startInstallConfirm 中，也就是我们安装应用看到的界面，此时会提取 apk 中的权限信息并展示出来，至此 PackageInstaller 的初始化工作就完成了。

总结：

1. 根据 Uri 的 Scheme 协议不同，跳到不同的界面，content 协议跳到 InstallStart，其他跳到 PackageInstallerActivity；
2. InstallStart 将 content 协议的 Uri 装换成 file 协议，然后跳转到 PackageInstallerActivity；
3. PackageInstallerActivity 会分别对 package 协议和 file 协议的 Uri 进行处理，解析出 apk 文件得到包信息 PackageInfo。
4. PackageInstallerActivity 对未知来源的 apk 进行处理，展示解析出应用所需权限等。

### 三、PackageInstaller 安装 apk 的过程

在应用安装点击确定按钮，即 PackageInstallerActivity 的 onClick 方法，此为应用安装的入口，可由此查看源码，可知具体回到 InstallInstalling 中处理安装流程。具体读者自行查看，以下为调用时序图。

在 InstallInstalling 的 onCreate 中，获取 mSessionId（安装回话 id） 和 mInstallId（安装事件 id），并注册安装事件的回调。在 onResume 中，通过先前获取到的 mSessionId 拿到 sessionInfo 及由 PackageManager 拿到 PackageInstaller，调用安装任务 InstallingAsyncTask 开启安装，最终会调用到 PMS 中去进行安装处理。

![install](https://raw.githubusercontent.com/huanggenghg/huanggenghg/main/res/PackageInstaller%20%E5%AE%89%E8%A3%85%20apk%20%E8%BF%87%E7%A8%8B.drawio.png)

简单总结，PackageInstaller 安装 apk 过程有两步：

1. 将 apk 信息通过 I/O 流写入 PackageInstaller.Session 中；
2. 调用 PackageInstaller.Session 的 commit 方法，将 apk 信息交由 PMS 处理。

### 四、PMS 安装 apk 过程

具体调用流程可自行查看源码。

![pms install](https://raw.githubusercontent.com/huanggenghg/huanggenghg/main/res/PMS%20%E5%AE%89%E8%A3%85%20apk.drawio.png)

installPackageLI 及 installNewPackageLIF 流程

![install package](https://raw.githubusercontent.com/huanggenghg/huanggenghg/main/res/InstallPackageLI.drawio.png)

存储 apk 信息是将 apk 信息存储在 PackageParser.Package 类型的 newPackage 中，一个 Package 的信息包含了一个 base apk 及零个或者多个 split apk。

总结 PMS 安装的步骤：

1. PackageInstaller 安装 apk 时会将 apk 信息交由 PMS 处理，PMS 通过 PackageHandler 发送消息来驱动 apk 的复制和安装工作。
2. PMS 发送 INIT_COPY 和 MCS_BOUND 类型信息，控制 PackageHandler 来绑定 DefaultContainerService，完成复制 apk 等工作
3. 复制完 apk 后，开始进行安装 apk 的流程，包括安装前的检查、安装 apk 和安装后的收尾工作。

### 五、PMS 的创建过程

PMS 是在 SystemServer 进程中被创建的，是于`startBootstrapServices`启动引导服务中启动创建的

```java
    public static PackageManagerService main(Context context, Installer installer,
            boolean factoryTest, boolean onlyCore) {
        // Self-check for initial settings.
        PackageManagerServiceCompilerMapping.checkProperties();

        PackageManagerService m = new PackageManagerService(context, installer,
                factoryTest, onlyCore);
        m.enableSystemUserPackages();
        ServiceManager.addService("package", m);
        final PackageManagerNative pmn = m.new PackageManagerNative();
        ServiceManager.addService("package_native", pmn);
        return m;
    }
```

main 方法作用，一是创建 PMS 对象，二是将 PMS 注册到 SystemManager 中。

PMS 的构造过程分为 5 个阶段：

1. BOOT_PROGRESS_PMS_START（开始阶段）
2. BOOT_PROGRESS_PMS_SYSTEMT_SCAN_START（扫描系统阶段）
3. BOOT_PROGRESS_PMS_DATAS_SCAN_START（扫描 Data 分区阶段）
4. BOOT_PROGRESS_PMS_SCAN_END（扫描结束阶段）
5. BOOT_PROGRESS_PMS_READY（准备阶段）

```mermaid
graph LR
开始阶段 --> 扫描系统阶段 --> 扫描Data分区阶段 --> 扫描结束阶段 --> 准备阶段
```

#### 5.1 开始阶段

构造方法中获取一些包管理所需的属性：

- mSettings：用于保存多有包的动态设置。（sharedUserId 用于两个应用间共享数据）
- mInstaller：系统服务，PMS 很多操作的完成者，在其内部通过 IInstallId 和 installId 进行 Binder 通信，由位于 Native 层的 installId 来完成具体操作
- mPackageDexOptimizer：Dex 优化工具类
- mHandler(PackageHandler)：PMS 通过 PackageHandler 驱动 apk 的复制和安装工作
- Watchdog：定时检测系统关键服务是否可能发生死锁；定时检测线程的消息队列是否长时间处于工作状态。若出现上述问题，则 Watchdog 会将日志保存起来，必要时会杀掉自己所在进程（SystemServer 进程）
- ...

#### 5.2 扫描系统阶段

> Android 系统架构分为应用层、应用框架层、系统运行库层（Native 层）、硬件抽象层（HAL 层）和 Linux 内核层，除了 Linux 内核层在 Boot 分区，其他层代码都在 System 分区。

工作特点：

1. 创建 /system 的子目录，比如 /system/framework、/system/priv-app、/system/app 等
2. 扫描系统文件，比如 /vendor/overlay、/system/framework 和 /system/app 等目录下的文件
3. 对扫描到的系统文件进行后续处理（系统 app 更新、不更新、已删除？）

#### 5.3 扫描 Data 分区阶段

工作特点：

1. 扫描 /data/app 和 /data/app-private 目录下的文件
2. 遍历 possiblyDeletedUpdatedSystemApps 列表，若这个系统 app 的包信息不在 PMS 的变量 mPackages 中，则说明其是残留的 app 信息，后续会删除它的数据；若在 mPackages 中，则说明其存在于 Data 分区中，不属于系统 app，那么移除其系统权限
3. 遍历 mExpectingBetter 列表，处理系统 app 升级相关逻辑。

#### 5.4 扫描结束阶段

工作特点：

1. 第一次启动或当前平台 SDK 版本与上次启动时版本不一致，则会重新更新 apk 的授权
2. 如果是第一次启动或者 Android M 升级后的第一次启动，则需要初始化所有用户定义的默认首选 app
3. OTA 升级后的第一次启动，会清楚代码缓存目录
4. 把 Settings 的内容保存到 packages.xml 中，这样以后 PMS 再创建时会读到此前保存的 Settings 的内容

#### 5.5 准备阶段

创建 PackageInstallerService，用于管理安装会话的服务，它会为每次安装过程分配一个 SessionId；
进行垃圾收集；
将 PackageManagerInternalImpl（PackageManager 的本地服务）添加到 LocalServices 中，LocalServices 用于存储运行在当前进程中的本地服务。

### 六、APK 解析过程

PackageParser 将 Android 中的 apk、jar、so 库等转换为内存中的数据结构，供包管理在内存中进行处理。

> Android 5.0 引入了 Split APK 机制，其目的是解决 65536 上限及 apk 安装包越来越大的问题。

![PackageParser 解析 APK](https://raw.githubusercontent.com/huanggenghg/huanggenghg/main/res/PMS%20%E5%88%9B%E5%BB%BA%E6%B5%81%E7%A8%8B.drawio.png)

```java
    private Package parseClusterPackage(File packageDir, int flags) throws PackageParserException {
        final PackageLite lite = parseClusterPackageLite(packageDir, 0); // 轻量级解析目录文件
        if (mOnlyCoreApps && !lite.coreApp) {
            throw new PackageParserException(INSTALL_PARSE_FAILED_MANIFEST_MALFORMED,
                    "Not a coreApp: " + packageDir);
        }

        // Build the split dependency tree.
        SparseArray<int[]> splitDependencies = null;
        final SplitAssetLoader assetLoader;
        if (lite.isolatedSplits && !ArrayUtils.isEmpty(lite.splitNames)) {
            try {
                splitDependencies = SplitAssetDependencyLoader.createDependenciesFromPackage(lite);
                assetLoader = new SplitAssetDependencyLoader(lite, splitDependencies, flags);
            } catch (SplitAssetDependencyLoader.IllegalDependencyException e) {
                throw new PackageParserException(INSTALL_PARSE_FAILED_BAD_MANIFEST, e.getMessage());
            }
        } else {
            assetLoader = new DefaultSplitAssetLoader(lite, flags);
        }

        try {
            final AssetManager assets = assetLoader.getBaseAssetManager();
            final File baseApk = new File(lite.baseCodePath);
            final Package pkg = parseBaseApk(baseApk, assets, flags); // 解析 base apk
            if (pkg == null) {
                throw new PackageParserException(INSTALL_PARSE_FAILED_NOT_APK,
                        "Failed to parse base APK: " + baseApk);
            }

            if (!ArrayUtils.isEmpty(lite.splitNames)) {
                final int num = lite.splitNames.length; // split apk 的数量
                pkg.splitNames = lite.splitNames;
                pkg.splitCodePaths = lite.splitCodePaths;
                pkg.splitRevisionCodes = lite.splitRevisionCodes;
                pkg.splitFlags = new int[num];
                pkg.splitPrivateFlags = new int[num];
                pkg.applicationInfo.splitNames = pkg.splitNames;
                pkg.applicationInfo.splitDependencies = splitDependencies;
                pkg.applicationInfo.splitClassLoaderNames = new String[num];

                for (int i = 0; i < num; i++) {
                    final AssetManager splitAssets = assetLoader.getSplitAssetManager(i);
                    parseSplitApk(pkg, i, splitAssets, flags); // 解析 split apk
                }
            }

            pkg.setCodePath(packageDir.getCanonicalPath());
            pkg.setUse32bitAbi(lite.use32bitAbi);
            return pkg;
        } catch (IOException e) {
            throw new PackageParserException(INSTALL_PARSE_FAILED_UNEXPECTED_EXCEPTION,
                    "Failed to get path: " + lite.baseCodePath, e);
        } finally {
            IoUtils.closeQuietly(assetLoader);
        }
    }
```

```java
    private Package parseBaseApk(File apkFile, AssetManager assets, int flags)
            throws PackageParserException {
        final String apkPath = apkFile.getAbsolutePath();

        String volumeUuid = null;
        if (apkPath.startsWith(MNT_EXPAND)) { // 如果 apk 的路径是以 /mnt/expand/ 开头，以此截取 volumeUuid
            final int end = apkPath.indexOf('/', MNT_EXPAND.length());
            volumeUuid = apkPath.substring(MNT_EXPAND.length(), end);
        }

        mParseError = PackageManager.INSTALL_SUCCEEDED;
        mArchiveSourcePath = apkFile.getAbsolutePath();

        if (DEBUG_JAR) Slog.d(TAG, "Scanning base APK: " + apkPath);

        XmlResourceParser parser = null;
        try {
            final int cookie = assets.findCookieForPath(apkPath);
            if (cookie == 0) {
                throw new PackageParserException(INSTALL_PARSE_FAILED_BAD_MANIFEST,
                        "Failed adding asset path: " + apkPath);
            }
            parser = assets.openXmlResourceParser(cookie, ANDROID_MANIFEST_FILENAME);
            final Resources res = new Resources(assets, mMetrics, null);

            final String[] outError = new String[1];
            final Package pkg = parseBaseApk(apkPath, res, parser, flags, outError); // 重载方法
            if (pkg == null) {
                throw new PackageParserException(mParseError,
                        apkPath + " (at " + parser.getPositionDescription() + "): " + outError[0]);
            }

            pkg.setVolumeUuid(volumeUuid); // 标识解析后的 package
            pkg.setApplicationVolumeUuid(volumeUuid); // 标识 app 所在的存储卷 uuid
            pkg.setBaseCodePath(apkPath);
            pkg.setSigningDetails(SigningDetails.UNKNOWN);

            return pkg;

        } catch (PackageParserException e) {
            throw e;
        } catch (Exception e) {
            throw new PackageParserException(INSTALL_PARSE_FAILED_UNEXPECTED_EXCEPTION,
                    "Failed to read manifest from " + apkPath, e);
        } finally {
            IoUtils.closeQuietly(parser);
        }
    }
```

``` java
    private Package parseBaseApk(String apkPath, Resources res, XmlResourceParser parser, int flags,
            String[] outError) throws XmlPullParserException, IOException {
        ...

        final Package pkg = new Package(pkgName);

        TypedArray sa = res.obtainAttributes(parser,
                com.android.internal.R.styleable.AndroidManifest); // 提取自定义属性集

        pkg.mVersionCode = sa.getInteger(
                com.android.internal.R.styleable.AndroidManifest_versionCode, 0);
        pkg.mVersionCodeMajor = sa.getInteger(
                com.android.internal.R.styleable.AndroidManifest_versionCodeMajor, 0);
        pkg.applicationInfo.setVersionCode(pkg.getLongVersionCode());
        pkg.baseRevisionCode = sa.getInteger(
                com.android.internal.R.styleable.AndroidManifest_revisionCode, 0);
        pkg.mVersionName = sa.getNonConfigurationString(
                com.android.internal.R.styleable.AndroidManifest_versionName, 0);
        if (pkg.mVersionName != null) {
            pkg.mVersionName = pkg.mVersionName.intern();
        }

        ...

        sa.recycle();

        return parseBaseApkCommon(pkg, null, res, parser, flags, outError); // 解析 apk AndroidMainifest 中的各个标签，解析 application 标签在其中 parseBaseApplication 方法
    }
```

```java
private boolean parseBaseApplication(Package owner, Resources res, XmlResourceParser parser, int flags, String[] outError)
    throws XmlPullParserException, IOException {
    ...
        while ((type = parser.next()) != XmlPullParser.END_DOCUMENT
                && (type != XmlPullParser.END_TAG || parser.getDepth() > innerDepth)) {
            if (type == XmlPullParser.END_TAG || type == XmlPullParser.TEXT) {
                continue;
            }

            String tagName = parser.getName();
            // 解析四大组件
            if (tagName.equals("activity")) { 
                Activity a = parseActivity(owner, res, parser, flags, outError, cachedArgs, false,
                        owner.baseHardwareAccelerated);
                if (a == null) {
                    mParseError = PackageManager.INSTALL_PARSE_FAILED_MANIFEST_MALFORMED;
                    return false;
                }

                hasActivityOrder |= (a.order != 0);
                owner.activities.add(a);

            } else if (tagName.equals("receiver")) {
                Activity a = parseActivity(owner, res, parser, flags, outError, cachedArgs,
                        true, false);
                if (a == null) {
                    mParseError = PackageManager.INSTALL_PARSE_FAILED_MANIFEST_MALFORMED;
                    return false;
                }

                hasReceiverOrder |= (a.order != 0);
                owner.receivers.add(a);

            } else if (tagName.equals("service")) {
                Service s = parseService(owner, res, parser, flags, outError, cachedArgs);
                if (s == null) {
                    mParseError = PackageManager.INSTALL_PARSE_FAILED_MANIFEST_MALFORMED;
                    return false;
                }

                hasServiceOrder |= (s.order != 0);
                owner.services.add(s);

            } else if (tagName.equals("provider")) {
                Provider p = parseProvider(owner, res, parser, flags, outError, cachedArgs);
                if (p == null) {
                    mParseError = PackageManager.INSTALL_PARSE_FAILED_MANIFEST_MALFORMED;
                    return false;
                }

                owner.providers.add(p);

            }
            ...
        }
    ...
}
```

![parseBaseApk 方法主要的解析结构](https://raw.githubusercontent.com/huanggenghg/huanggenghg/main/res/parseBaseApk%20%E4%B8%BB%E8%A6%81%E8%A7%A3%E6%9E%90%E7%BB%93%E6%9E%84.drawio.png)

apk 包被解析后，最终在内存中的数据结构是 Package，Package 是 PackageParser 的内部类，对应的成员变量存储了不同的组件或应用等信息。