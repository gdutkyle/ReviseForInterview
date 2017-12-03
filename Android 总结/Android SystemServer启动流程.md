#Android SystemServer 启动流程  

### 一 什么是SystemServer？  
简单来说，systemServer就是系统用来启动各种service的入口，安卓系统在启动的时候，会初始化两个重要的部分，一个是zygote进程，另一个是由zygote进程fork出来的SystemServer进程，SystemServer会启动我们系统中所需要的一系列service，下面会做分析。
### 二 从源码出发，分析Systemserver  
**2.1 SystemServer调用的入口**  

    public static void main(String[] args) {
        new SystemServer().run();
    }
SystemServer的初始化入口在main（）方法中，我们进入run（）方法中，可以看到以下代码：  

    private void run() {
        try {
            ......
            if (System.currentTimeMillis() < EARLIEST_SUPPORTED_TIME) {
                Slog.w(TAG, "System clock is before 1970; setting to 1970.");
                SystemClock.setCurrentTimeMillis(EARLIEST_SUPPORTED_TIME);
            }

           
            if (!SystemProperties.get("persist.sys.language").isEmpty()) {
                final String languageTag = Locale.getDefault().toLanguageTag();

                SystemProperties.set("persist.sys.locale", languageTag);
                SystemProperties.set("persist.sys.language", "");
                SystemProperties.set("persist.sys.country", "");
                SystemProperties.set("persist.sys.localevar", "");
            }

            // Here we go!
            Slog.i(TAG, "Entered the Android system server!");
            EventLog.writeEvent(EventLogTags.BOOT_PROGRESS_SYSTEM_RUN, SystemClock.uptimeMillis());

            
            SystemProperties.set("persist.sys.dalvik.vm.lib.2", VMRuntime.getRuntime().vmLibrary());

            VMRuntime.getRuntime().clearGrowthLimit();
            VMRuntime.getRuntime().setTargetHeapUtilization(0.8f);
            Build.ensureFingerprintProperty();
            Environment.setUserRequired(true);
            BaseBundle.setShouldDefuse(true);
            BinderInternal.disableBackgroundScheduling(true);

    
            android.os.Process.setThreadPriority(
                android.os.Process.THREAD_PRIORITY_FOREGROUND);
            android.os.Process.setCanSelfBackground(false);
            Looper.prepareMainLooper();

            // Initialize native services.
            System.loadLibrary("android_servers");

            // Check whether we failed to shut down last time we tried.
            // This call may not return.
            performPendingShutdown();

            // Initialize the system context.
            createSystemContext();

            // Create the system service manager.
            mSystemServiceManager = new SystemServiceManager(mSystemContext);
            LocalServices.addService(SystemServiceManager.class, mSystemServiceManager);
        } finally {
            Trace.traceEnd(Trace.TRACE_TAG_SYSTEM_SERVER);
        }

        // Start services.
        try {
            Trace.traceBegin(Trace.TRACE_TAG_SYSTEM_SERVER, "StartServices");
            startBootstrapServices();
            startCoreServices();
            startOtherServices();
        } catch (Throwable ex) {
            Slog.e("System", "******************************************");
            Slog.e("System", "************ Failure starting system services", ex);
            throw ex;
        } finally {
            Trace.traceEnd(Trace.TRACE_TAG_SYSTEM_SERVER);
        }
        // Loop forever.
        Looper.loop();
        throw new RuntimeException("Main thread loop unexpectedly exited");
    }
以上代码已经去除了部分无用代码，我们下面来一步一步的对上面的代码进行切割分析  

    if (System.currentTimeMillis() < EARLIEST_SUPPORTED_TIME) {
                Slog.w(TAG, "System clock is before 1970; setting to 1970.");
                SystemClock.setCurrentTimeMillis(EARLIEST_SUPPORTED_TIME);
            }
首先，SystemServer会去判断系统当前的时间，如果当前时间小于1970年那么，系统就会把当前的时间设置成1970年。
接下来，
    
            if (!SystemProperties.get("persist.sys.language").isEmpty()) {
                final String languageTag = Locale.getDefault().toLanguageTag();

                SystemProperties.set("persist.sys.locale", languageTag);
                SystemProperties.set("persist.sys.language", "");
                SystemProperties.set("persist.sys.country", "");
                SystemProperties.set("persist.sys.localevar", "");
            }
这代代码主要是初始化一些基本环境设置，比如说当前的语言，国家这些。  
接着， 

           SystemProperties.set("persist.sys.dalvik.vm.lib.2", VMRuntime.getRuntime().vmLibrary());

            VMRuntime.getRuntime().clearGrowthLimit();
            VMRuntime.getRuntime().setTargetHeapUtilization(0.8f);
            Build.ensureFingerprintProperty();
            Environment.setUserRequired(true);
            BaseBundle.setShouldDefuse(true);
            BinderInternal.disableBackgroundScheduling(true);

    
            android.os.Process.setThreadPriority(
                android.os.Process.THREAD_PRIORITY_FOREGROUND);
            android.os.Process.setCanSelfBackground(false);
            Looper.prepareMainLooper();

            // Initialize native services.
            System.loadLibrary("android_servers");
这一部分主要是加载一些虚拟机环境和一些运行库，这些都不是本文所要分析的重点，接下来还有一个方法： 
 
      // Initialize the system context.
            createSystemContext();

     private void createSystemContext() {  
        ActivityThread activityThread = ActivityThread.systemMain();
        mSystemContext = activityThread.getSystemContext();
        mSystemContext.setTheme(DEFAULT_SYSTEM_THEME);
    }

这个方法表示SystemServer需要获取当前系统的上下文  
接下来，就会初始化系统的system Service Manager  

    mSystemServiceManager = new SystemServiceManager(mSystemContext);
            LocalServices.addService(SystemServiceManager.class, mSystemServiceManager);  
systemManagerService的作用是用来管理service的创建、开始或者SystemService下的其他生命周期事件。好了，接下来就是重头戏来了  

    try {
            startBootstrapServices();
            startCoreServices();
            startOtherServices();
        } catch (Throwable ex) {
            ......
            throw ex;
        } finally {
              ......
        }  
从谷歌给我们的注释我们给出以上三个方法简单的解释  
1 startBootstrapServices() 主要是用来开启一些系统级别的service，这些service具有高度的复杂的相互依赖关系，所以我们需要把他们的初始化放在同一个地方  
2 startCoreServices() 主要是启动一些核心的Service，但是这些service跟我们的bootservice没有相互依赖关系的，是相对独立的服务
2 startOtherServices() 正如英文所示，这是一些费关键非核心的service

**Step 1 分析startBootstrapService()方法**  

    private void startBootstrapServices() {
        Installer installer = mSystemServiceManager.startService(Installer.class);

        traceBeginAndSlog("DeviceIdentifiersPolicyService");
        mSystemServiceManager.startService(DeviceIdentifiersPolicyService.class);
        traceEnd();

        // Activity manager runs the show.
        mActivityManagerService = mSystemServiceManager.startService(
                ActivityManagerService.Lifecycle.class).getService();
        mActivityManagerService.setSystemServiceManager(mSystemServiceManager);
        mActivityManagerService.setInstaller(installer);

        // Power manager needs to be started early because other services need it.
        // Native daemons may be watching for it to be registered so it must be ready
        // to handle incoming binder calls immediately (including being able to verify
        // the permissions for those calls).
        mPowerManagerService = mSystemServiceManager.startService(PowerManagerService.class);

        // Now that the power manager has been started, let the activity manager
        // initialize power management features.
        Trace.traceBegin(Trace.TRACE_TAG_SYSTEM_SERVER, "InitPowerManagement");
        mActivityManagerService.initPowerManagement();
        Trace.traceEnd(Trace.TRACE_TAG_SYSTEM_SERVER);

        // Manages LEDs and display backlight so we need it to bring up the display.
        mSystemServiceManager.startService(LightsService.class);

        // Display manager is needed to provide display metrics before package manager
        // starts up.
        mDisplayManagerService = mSystemServiceManager.startService(DisplayManagerService.class);

        // We need the default display before we can initialize the package manager.
        mSystemServiceManager.startBootPhase(SystemService.PHASE_WAIT_FOR_DEFAULT_DISPLAY);

        // Only run "core" apps if we're encrypting the device.
        String cryptState = SystemProperties.get("vold.decrypt");
        if (ENCRYPTING_STATE.equals(cryptState)) {
            Slog.w(TAG, "Detected encryption in progress - only parsing core apps");
            mOnlyCore = true;
        } else if (ENCRYPTED_STATE.equals(cryptState)) {
            Slog.w(TAG, "Device encrypted - only parsing core apps");
            mOnlyCore = true;
        }

        // Start the package manager.
        traceBeginAndSlog("StartPackageManagerService");
        mPackageManagerService = PackageManagerService.main(mSystemContext, installer,
                mFactoryTestMode != FactoryTest.FACTORY_TEST_OFF, mOnlyCore);
        mFirstBoot = mPackageManagerService.isFirstBoot();
        mPackageManager = mSystemContext.getPackageManager();
        Trace.traceEnd(Trace.TRACE_TAG_SYSTEM_SERVER);

        // Manages A/B OTA dexopting. This is a bootstrap service as we need it to rename
        // A/B artifacts after boot, before anything else might touch/need them.
        // Note: this isn't needed during decryption (we don't have /data anyways).
        if (!mOnlyCore) {
            boolean disableOtaDexopt = SystemProperties.getBoolean("config.disable_otadexopt",
                    false);
            if (!disableOtaDexopt) {
                traceBeginAndSlog("StartOtaDexOptService");
                try {
                    OtaDexoptService.main(mSystemContext, mPackageManagerService);
                } catch (Throwable e) {
                    reportWtf("starting OtaDexOptService", e);
                } finally {
                    Trace.traceEnd(Trace.TRACE_TAG_SYSTEM_SERVER);
                }
            }
        }

        traceBeginAndSlog("StartUserManagerService");
        mSystemServiceManager.startService(UserManagerService.LifeCycle.class);
        Trace.traceEnd(Trace.TRACE_TAG_SYSTEM_SERVER);

        // Initialize attribute cache used to cache resources from packages.
        AttributeCache.init(mSystemContext);

        // Set up the Application instance for the system process and get started.
        mActivityManagerService.setSystemProcess();

        // The sensor service needs access to package manager service, app ops
        // service, and permissions service, therefore we start it after them.
        startSensorService();
    }  
在这段代码中，我们先分析核心的代码：  

        traceBeginAndSlog("StartInstaller");
        Installer installer = mSystemServiceManager.startService(Installer.class);
        traceEnd();

        // In some cases after launching an app we need to access device identifiers,
        // therefore register the device identifier policy before the activity manager.
        traceBeginAndSlog("DeviceIdentifiersPolicyService");
        mSystemServiceManager.startService(DeviceIdentifiersPolicyService.class);
        traceEnd();
这段代码的执行，就是为了先启动installer，这样android才有机会去创建一些关键的路径（data/user）,这些都需要在其他Service启动前完成。其次，通过`mSystemServiceManager.startService(DeviceIdentifiersPolicyService.class);`这段代码，android需要注册当前的设备表示，以防有一些特殊的时候需要用到。  
我们进入install类的onStart()方法一看  

    @Override
    public void onStart() {
        if (mIsolated) {
            mInstalld = null;
        } else {
            connect();
        }
    }  
    private void connect() {
        IBinder binder = ServiceManager.getService("installd");
        if (binder != null) {
            try {
                binder.linkToDeath(new DeathRecipient() {
                    @Override
                    public void binderDied() {
                        Slog.w(TAG, "installd died; reconnecting");
                        connect();
                    }
                }, 0);
            } catch (RemoteException e) {
                binder = null;
            }
        }

        if (binder != null) {
            mInstalld = IInstalld.Stub.asInterface(binder);
            try {
                invalidateMounts();
            } catch (InstallerException ignored) {
            }
        } else {
            Slog.w(TAG, "installd not found; trying again");
            BackgroundThread.getHandler().postDelayed(() -> {
                connect();
            }, DateUtils.SECOND_IN_MILLIS);
        }
    }
我们可以看到，其实这就是一个递归。install通过调用connect，去判断installer是否被初始化。只有installer被初始化了，才会继续往下掉用，初始化其他的服务。  
接下来，启动ActivityManagerService。我们通过代码分析：  

        mActivityManagerService = mSystemServiceManager.startService(
                ActivityManagerService.Lifecycle.class).getService();
        mActivityManagerService.setSystemServiceManager(mSystemServiceManager);
        mActivityManagerService.setInstaller(installer);
我们开启ams传入的是 ActivityManagerService.Lifecycle.class，同时把installer传入给ams。我们进入AMS查看代码，可以看到  

    final void finishBooting() {
        ...
        for (String abi : Build.SUPPORTED_ABIS) {
            zygoteProcess.establishZygoteConnectionForAbi(abi);
            final String instructionSet = VMRuntime.getInstructionSet(abi);
            if (!completedIsas.contains(instructionSet)) {
                try {
                    mInstaller.markBootComplete(VMRuntime.getInstructionSet(abi));
                } catch (InstallerException e) {
                    Slog.w(TAG, "Unable to mark boot complete for abi: " + abi + " (" +
                            e.getMessage() +")");
                }
                completedIsas.add(instructionSet);
            }
        }
      ...
     }
所以finishBooting是ams和zygote进行通讯的入口。  
startBootstrapServices()方法中，还启动了很多其他的services，包括PowerManagerService、RecoverySystemService、LightsService、DisplayManagerService等等，这里就不重复做出分析了。  

**Step 2 分析startCoreServices()方法**  

    private void startCoreServices() {
        // Records errors and logs, for example wtf()
        traceBeginAndSlog("StartDropBoxManager");
        mSystemServiceManager.startService(DropBoxManagerService.class);
        traceEnd();

        traceBeginAndSlog("StartBatteryService");
        // Tracks the battery level.  Requires LightService.
        mSystemServiceManager.startService(BatteryService.class);
        traceEnd();

        // Tracks application usage stats.
        traceBeginAndSlog("StartUsageService");
        mSystemServiceManager.startService(UsageStatsService.class);
        mActivityManagerService.setUsageStatsManager(
                LocalServices.getService(UsageStatsManagerInternal.class));
        traceEnd();

        // Tracks whether the updatable WebView is in a ready state and watches for update installs.
        traceBeginAndSlog("StartWebViewUpdateService");
        mWebViewUpdateService = mSystemServiceManager.startService(WebViewUpdateService.class);
        traceEnd();
    }  
这个方法主要启动一些跟bootstrap进程无关的service，这里比较好玩的是，android会在这里去检测WebView是处于就绪状态和手动更新安装。  
**Step 3 分析startOtherServices()方法**  
这个方法过长，就不贴上源码的，这个方法主要是为了整理或者重构一些杂七杂八的包，不太重要，不做分析。
### 总结
至此，systemServer启动流程分析完毕，可能篇幅太长，大家需要自己好好整理，没事多复习