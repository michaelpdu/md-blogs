# 多进程启动过程

在Windows环境下，打开Chrome浏览器，通过进程管理工具可以看到下面类似的进程树。

```cpp
chrome.exe                                  // 0. browser process
    |- chrome.exe --type=crashpad-handler   // 1. crashpad
    |- chrome.exe --type=watcher            // 2. watcher
    |- chrome.exe --type=gpu-process        // 3. gpu process
    |- chrome.exe --type=renderer           // 4. render process
    |- ...
```

在Linux环境下，chrome启动以后，会有如下的process tree：

```cpp
chrome,32357,chrome
  ├─chrome,32360,zygote-process
  │   └─chrome,32362,zygote-process
  │       ├─chrome,32595,render-process
  │       └─chrome,32653,render-process
  ├─chrome,32386,gpu-process
```

那么，这些进程是做什么的？已经是如何启动的？下面我们会根据这个思路不断剖析多进程的启动过程。

## crashpad

crashpad process是在browser process进入main函数以后，作为第一个child process启动，主要是用于监控其他进程的crash，收集dump并上传到指定的server上。从下面的command line可以看出一些信息：

```cpp
"C:\Program Files (x86)\Google\Chrome\Application\chrome.exe" 
--type=crashpad-handler 
"--user-data-dir=C:\Users\michael_du\AppData\Local\Google\Chrome\User Data" 
/prefetch:7 
--monitor-self-annotation=ptype=crashpad-handler 
"--database=C:\Users\michael_du\AppData\Local\Google\Chrome\User Data\Crashpad" 
"--metrics-dir=C:\Users\michael_du\AppData\Local\Google\Chrome\User Data" 
--url=https://clients2.google.com/cr/report         // server URL
--annotation=channel= 
--annotation=plat=Win32 
--annotation=prod=Chrome 
--annotation=ver=65.0.3325.181 
--initial-client-data=0x234,0x238,0x23c,0x230,0x240,0x5e14d060,0x5e14d070,0x5e14d07c
```

crashpad process的启动是在`SignalInitializeCrashReporting`函数中完成的。

```cpp
#if !defined(WIN_CONSOLE_APP)
int APIENTRY wWinMain(HINSTANCE instance, HINSTANCE prev, wchar_t*, int) {
#else
int main() {
  HINSTANCE instance = GetModuleHandle(nullptr);
#endif
  install_static::InitializeFromPrimaryModule();
  SignalInitializeCrashReporting(); // start crashpad process here
  ...
}
```

```cpp
// browser process启动crashpad process的过程
wWinMain
..> SignalInitializeCrashReporting       //这个函数太简单，被优化掉了
--> elf_crash::InitializeCrashReporting
  --> ChromeCrashReporterClient::InitializeCrashReportingForProcess
    --> crash_reporter::InitializeCrashpadWithEmbeddedHandler
      --> crash_reporter::`anonymous namespace'::InitializeCrashpadImpl
        --> crash_reporter::internal::PlatformCrashpadInitialization
          --> crashpad::CrashpadClient::StartHandler
            --> crashpad::`anonymous namespace'::StartHandlerProcess
```

在chrome的main函数中，直接通过crash_reporter::RunAsCrashpadHandler启动crashpad process。

另外，crashpad是一个单独的project，用于监控crash，生成dump，并上报到server。

有兴趣可以进一步研究这个project的实现：[https://github.com/chromium/crashpad](https://github.com/chromium/crashpad)

## watcher

作用是什么？

从code上看，是在监控browser process，用于将进程的exit code写入register，方便下次启动时，知道上次是不是异常退出，是否需要恢复。

```cpp
// browser process启动watcher process的过程
wWinMain
--> MainDllLoader::Launch
  --> ChromeDllLoader::OnBeforeLaunch
    --> ChromeWatcherClient::LaunchWatcher
      --> browser_watcher::WatcherClient::LaunchWatcher
        --> base::LaunchProcess
```

watcher process的入口函数是chrome_watcher!WatcherMain，调用过程是这样的：

wWinMain --> MainDllLoader::Launch --> chrome_watcher!WatcherMain

如何起作用的？

## GPU Process

从main函数开始，启动不同进程，前面的几层调用栈类似下面这样，差异是会根据参数的不同，选择不同的entry，如：BrowserMain|RendererMain|GpuMain|UtilityMain|PpapiPluginMain|PpapiBrokerMain

在MainRunner的initialize函数中，会创建一些startup task，然后，在这些task中在做后续的初始化动作。

`为什么这么做？` 从代码层面理解，应该是Android平台有特殊的需求，这些startup task未必需要立刻执行起来。

```cpp
main
--> ChromeMain 
--> ContentMain 
--> Main 
--> ContentServiceManagerMainDelegate::RunEmbedderProcess 
--> ContentMainRunnerImpl::Run
--> RunNamedProcessTypeMain
--> BrowserMain(RendererMain|GpuMain|UtilityMain|PpapiPluginMain|PpapiBrokerMain)
    --> BrowserMainRunnerImpl::Initialize
        --> BrowserMainLoop::CreateStartupTasks 
            --> pre_create_threads
            --> create_threads
            --> post_create_threads
            --> browser_thread_started
            --> pre_main_message_loop_run

```

下面的这段code是RunNamedProcessTypeMain的具体实现：

```cpp
int RunNamedProcessTypeMain(
    const std::string& process_type,
    const MainFunctionParams& main_function_params,
    ContentMainDelegate* delegate) {
  static const MainFunction kMainFunctions[] = {
#if !defined(CHROME_MULTIPLE_DLL_CHILD)
    { "",                            BrowserMain },
#endif
#if !defined(CHROME_MULTIPLE_DLL_BROWSER)
#if BUILDFLAG(ENABLE_PLUGINS)
    { switches::kPpapiPluginProcess, PpapiPluginMain },
    { switches::kPpapiBrokerProcess, PpapiBrokerMain },
#endif  // ENABLE_PLUGINS
    { switches::kUtilityProcess,     UtilityMain },
    { switches::kRendererProcess,    RendererMain },
    { switches::kGpuProcess,         GpuMain },
#endif  // !CHROME_MULTIPLE_DLL_BROWSER
  };

  RegisterMainThreadFactories();

  for (size_t i = 0; i < arraysize(kMainFunctions); ++i) {
    if (process_type == kMainFunctions[i].name) {
      if (delegate) {
        int exit_code = delegate->RunProcess(process_type,
            main_function_params);
#if defined(OS_ANDROID)
        // In Android's browser process, the negative exit code doesn't mean the
        // default behavior should be used as the UI message loop is managed by
        // the Java and the browser process's default behavior is always
        // overridden.
        if (process_type.empty())
          return exit_code;
#endif
        if (exit_code >= 0)
          return exit_code;
      }
      return kMainFunctions[i].function(main_function_params);
    }
  }
  ...
}
```

下面这段code是CreateStartupTasks的具体实现：

```cpp
void BrowserMainLoop::CreateStartupTasks() {
  TRACE_EVENT0("startup", "BrowserMainLoop::CreateStartupTasks");

  DCHECK(!startup_task_runner_);
#if defined(OS_ANDROID)
  startup_task_runner_ = std::make_unique<StartupTaskRunner>(
      base::Bind(&BrowserStartupComplete), base::ThreadTaskRunnerHandle::Get());
#else
  startup_task_runner_ = std::make_unique<StartupTaskRunner>(
      base::Callback<void(int)>(), base::ThreadTaskRunnerHandle::Get());
#endif
  StartupTask pre_create_threads =
      base::Bind(&BrowserMainLoop::PreCreateThreads, base::Unretained(this));
  startup_task_runner_->AddTask(std::move(pre_create_threads));

  StartupTask create_threads =
      base::Bind(&BrowserMainLoop::CreateThreads, base::Unretained(this));
  startup_task_runner_->AddTask(std::move(create_threads));

  StartupTask post_create_threads =
      base::Bind(&BrowserMainLoop::PostCreateThreads, base::Unretained(this));
  startup_task_runner_->AddTask(std::move(post_create_threads));

  StartupTask browser_thread_started = base::Bind(
      &BrowserMainLoop::BrowserThreadsStarted, base::Unretained(this));
  startup_task_runner_->AddTask(std::move(browser_thread_started));

  StartupTask pre_main_message_loop_run = base::Bind(
      &BrowserMainLoop::PreMainMessageLoopRun, base::Unretained(this));
  startup_task_runner_->AddTask(std::move(pre_main_message_loop_run));

#if defined(OS_ANDROID)
  if (parameters_.ui_task) {
    // Running inside browser tests, which relies on synchronous start.
    startup_task_runner_->RunAllTasksNow();
  } else {
    startup_task_runner_->StartRunningTasksAsync();
  }
#else
  startup_task_runner_->RunAllTasksNow();
#endif
}

```

### BrowserThreadsStarted Task

BrowserThreadsStarted会去做以下工作：

- 初始化Mojo，用于IPC通信

- 如果参数中开启Vulkan，GPU会去做初始化的动作

- 初始化GPU shader cache

- BrowserGpuChannelHostFactory的初始化

- 如果支持WebRTC，初始化WebRTC

- 初始化ResourceDispatcherHost， MediaStreamManager， SpeechRecognitionManager， UserInputMonitor， SaveFileManager

- 设置哪些thread可以访问clipboard

- launch GPU process


创建GPUChannel的时候，UI Thread会将task交由IO Thread处理。

```cpp
BrowserThreadsStarted
    --> ...
    --> BrowserGpuChannelHostFactory::Initialize
        --> BrowserGpuChannelHostFactory::EstablishGpuChannel
            --> BrowserGpuChannelHostFactory::EstablishRequest::Create 
                ----------> (tast to io thread) BrowserGpuChannelHostFactory::EstablishRequest::EstablishOnIO

```

IO thread收到这个task以后，会调用GpuProcessHost::Get创建一个新的GpuProcessHost并做初始的工作。初始化过程中，GpuProcessHost会判断是不是single-process mode，如果不是，则启动GPU process。GPU process与render process有所差异，这里是使用BrowserChildProcessHostImpl::Launch调用ChildProcessLauncher完成的。具体的启动过程会交由单独的lancher thread完成。

```cpp
BrowserGpuChannelHostFactory::EstablishRequest::EstablishOnIO
--> GpuProcessHost::Get
    --> GpuProcessHost::Init
        --> GpuProcessHost::LaunchGpuProcess
            --> BrowserChildProcessHostImpl::Launch
                --> ChildProcessLauncher::ChildProcessLauncher
                    --> ChildProcessLauncherHelper::StartLaunchOnClientThread
                        ----------> (task to launch thread) ChildProcessLauncherHelper::LaunchOnLauncherThread
```

```cpp
void ChildProcessLauncherHelper::LaunchOnLauncherThread() {
  DCHECK(CurrentlyOnProcessLauncherTaskRunner());

  begin_launch_time_ = base::TimeTicks::Now();

  std::unique_ptr<FileMappedForLaunch> files_to_register = GetFilesToMap();

  bool is_synchronous_launch = true;
  int launch_result = LAUNCH_RESULT_FAILURE;
  base::LaunchOptions options;

  Process process;
  if (BeforeLaunchOnLauncherThread(*files_to_register, &options)) {
    process =
        LaunchProcessOnLauncherThread(options, std::move(files_to_register),
                                      &is_synchronous_launch, &launch_result);

    AfterLaunchOnLauncherThread(process, options);
  }

  if (is_synchronous_launch) {
    PostLaunchOnLauncherThread(std::move(process), launch_result);
  }
}

```

```cpp
LaunchOnLauncherThread
    --> 对于BeforeLaunchOnLauncherThread函数，Android/Linux/Win平台都是直接返回true。
    --> 对于PostLaunchOnLauncherThread，除了Android平台，其他都是会执行到的。
    --> LaunchProcessOnLauncherThread会根据平台不同，具体实现细节有所差异。
```


## Render Process

在上面介绍的GPU Process启动过程中，主线程会给自己喂几个startup tasks，譬如：`PreCreateThreads`，`CreateThreads`，`PostCreateThreads`，`BrowserThreadsStarted`和`PreMainMessageLoopRun`。GPU Process的创建过程是在`BrowserThreadsStarted`中完成的，而下面要介绍的Render Process则是在`PreMainMessageLoopRun`中完成。

```cpp
PreMainMessageLoopRun
--> ChromeBrowserMainParts::PreMainMessageLoopRun
  --> ChromeBrowserMainParts::PreMainMessageLoopRunImpl
    --> StartupBrowserCreator::Start
      --> StartupBrowserCreator::ProcessCmdLineImpl
        --> StartupBrowserCreator::LaunchBrowserForLastProfiles
          --> StartupBrowserCreator::ProcessLastOpenedProfiles
            --> StartupBrowserCreator::LaunchBrowser
              --> StartupBrowserCreatorImpl::Launch
                --> StartupBrowserCreatorImpl::DetermineURLsAndLaunch
                  --> StartupBrowserCreatorImpl::RestoreOrCreateBrowser
                    --> StartupBrowserCreatorImpl::OpenTabsInBrowser
                      --> Navigate
                        --> LoadURLInContents
                          --> NavigationControllerImpl::LoadURLWithParams
                            --> NavigationControllerImpl::LoadEntry
                              --> NavigationControllerImpl::NavigateToPendingEntry
                                --> NavigationControllerImpl::NavigateToPendingEntryInternal
                                  --> NavigatorImpl::NavigateToPendingEntry
                                    --> NavigatorImpl::NavigateToEntry
                                      --> NavigatorImpl::RequestNavigation
                                        --> FrameTreeNode::CreatedNavigationRequest
                                          --> RenderFrameHostManager::DidCreateNavigationRequest
                                            --> RenderFrameHostManager::GetFrameHostForNavigation
                                              --> RenderFrameHostManager::ReinitializeRenderFrame
                                                --> RenderFrameHostManager::InitRenderView
                                                  --> RenderProcessHostImpl::Init()
                                                    --> ChildProcessLauncher::ChildProcessLauncher
                                                      --> ChildProcessLauncherHelper::StartLaunchOnClientThread
                                                        --> ChildProcessLauncherHelper::StartLaunchOnClientThread
```

从上面的调用过程可以看出，创建Render Process的时机是浏览器做navigate动作的时候，由NavigationController来控制Navigator完成。最终的实现，是由RenderFrameHostManager去做InitRenderView的时候，会对RenderProcessHost做初始化动作。在RenderProcessHost初始化的时候，会启动一个LaunchThread完成相关的创建工作，如同GPU Process一样。

在Chromium中，browser process中有一些host object是用于管理child process的，分别有下面几种：

```cpp
Utility_process_host        --> Utility_process
Gpu_process_host            --> GPU_process
Ppapi_plugin_process_host   --> Ppapi_plugin_process
Render_process_host         --> Render_process

// 下面几种Process Host的作用是什么？
Child_process_host
Brower_child_process_host
Mock_render_process_host
Profiling_process_host
Native_message_process_host
```

Linux下面的Chromium默认是会启动Zygote Process，再由Zygote Process负责fork出render process，详细介绍可以参看：[Linux Zygote Process](https://chromium.googlesource.com/chromium/src/+/lkcr/docs/linux_zygote.md)

```cpp
// 创建zygote process的过程
main
--> ChromeMain
  --> ContentMain
    --> Main
      --> ContentServiceManagerMainDelegate::RunEmbedderProcess
        --> ContentMainRunnerImpl::Run
          --> RunNamedProcessTypeMain
            --> BrowserMain
              --> BrowserMainRunnerImpl::Initialize
                --> BrowserMainLoop::EarlyInitialization
                  --> SetupSandbox
                    --> CreateGenericZygote             // 返回一个全局的ZygoteCommunication指针，后面会用这个指针通信
                      --> ZygoteCommunication::Init     // 调用LaunchZygoteHelper启动Zygote进程
                        --> LaunchZygoteHelper
                          --> ZygoteHostImpl::LaunchZygote

// 发送fork request的过程
ChildProcessLauncherHelper::LaunchOnLauncherThread
--> ChildProcessLauncherHelper::LaunchProcessOnLauncherThread
  --> ZygoteCommunication::ForkRequest
    --> ZygoteCommunication::SendMessage
      --> UnixDomainSocket::SendMsg
        ----------> send message to Zygote process to fork a child process

// 终止child process的过程
ChildProcessLauncherHelper::ForceNormalProcessTerminationSync
--> ZygoteCommunication::EnsureProcessTerminated
  --> ZygoteCommunication::SendMessage
    --> UnixDomainSocket::SendMsg
      ----------> send message to Zygote process to terminate child process
```

下面这段code是Linux版本的LaunchProcessOnLauncherThread，逻辑很简单。先判断有没有zygote process，如果有，就通过zygote_handle发送fork request，如果没有，直接创建进程。

```cpp
ChildProcessLauncherHelper::Process
ChildProcessLauncherHelper::LaunchProcessOnLauncherThread(
    const base::LaunchOptions& options,
    std::unique_ptr<FileMappedForLaunch> files_to_register,
    bool* is_synchronous_launch,
    int* launch_result) {
  *is_synchronous_launch = true;

  ZygoteHandle zygote_handle =
      base::CommandLine::ForCurrentProcess()->HasSwitch(switches::kNoZygote)
          ? nullptr
          : delegate_->GetZygote();
  if (zygote_handle) {
    // TODO(crbug.com/569191): If chrome supported multiple zygotes they could
    // be created lazily here, or in the delegate GetZygote() implementations.
    // Additionally, the delegate could provide a UseGenericZygote() method.
    base::ProcessHandle handle = zygote_handle->ForkRequest(
        command_line()->argv(), files_to_register->GetMapping(),
        GetProcessType());
    *launch_result = LAUNCH_RESULT_SUCCESS;

#if !defined(OS_OPENBSD)
    if (handle) {
      // This is just a starting score for a renderer or extension (the
      // only types of processes that will be started this way).  It will
      // get adjusted as time goes on.  (This is the same value as
      // chrome::kLowestRendererOomScore in chrome/chrome_constants.h, but
      // that's not something we can include here.)
      const int kLowestRendererOomScore = 300;
      ZygoteHostImpl::GetInstance()->AdjustRendererOOMScore(
          handle, kLowestRendererOomScore);
    }
#endif

    Process process;
    process.process = base::Process(handle);
    process.zygote = zygote_handle;
    return process;
  }

  Process process;
  process.process = base::LaunchProcess(*command_line(), options);
  *launch_result = process.process.IsValid() ? LAUNCH_RESULT_SUCCESS
                                             : LAUNCH_RESULT_FAILURE;
  return process;
}
```

在zygote process中，如何处理fork request？可以参考下面Zygote处理request的过程。

```cpp
ZygoteMain
--> Zygote.ProcessRequests
  --> Zygote.HandleRequestFromBrowser
    --> Zygote.HandleForkRequest
```