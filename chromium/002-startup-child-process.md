# Chromium Multiple Process Startup

## GPU Process的启动过程

```
Thread 1 "chrome" hit Breakpoint 3, (anonymous namespace)::BrowserMainLoop::CreateStartupTasks (this=0x24afb74c5920) at ../../content/browser/browser_main_loop.cc:917
917	  TRACE_EVENT0("startup", "BrowserMainLoop::CreateStartupTasks");
(gdb) bt
#0  0x00007ffff13d60c6 in (anonymous namespace)::BrowserMainLoop::CreateStartupTasks() (this=0x24afb74c5920) at ../../content/browser/browser_main_loop.cc:917
#1  0x00007ffff13e3593 in (anonymous namespace)::BrowserMainRunnerImpl::Initialize((anonymous namespace)::MainFunctionParams const&) (this=0x24afb74a3ec0, parameters=...)
    at ../../content/browser/browser_main_runner.cc:140
#2  0x00007ffff13cea3b in (anonymous namespace)::BrowserMain((anonymous namespace)::MainFunctionParams const&) (parameters=...) at ../../content/browser/browser_main.cc:42
#3  0x00007ffff3089017 in (anonymous namespace)::RunNamedProcessTypeMain((anonymous namespace)::(anonymous namespace)::string const&, (anonymous namespace)::MainFunctionParams const&, (anonymous namespace)::ContentMainDelegate*) (process_type=..., main_function_params=..., delegate=0x7fffffffdb40) at ../../content/app/content_main_runner.cc:427
#4  0x00007ffff308b6f4 in (anonymous namespace)::ContentMainRunnerImpl::Run() (this=0x24afb75e0a40) at ../../content/app/content_main_runner.cc:706
#5  0x00007ffff3081bc5 in (anonymous namespace)::ContentServiceManagerMainDelegate::RunEmbedderProcess() (this=0x7fffffffdaa0)
    at ../../content/app/content_service_manager_main_delegate.cc:51
#6  0x00007ffff7e8fdfc in (anonymous namespace)::Main((anonymous namespace)::MainParams const&) (params=...) at ../../services/service_manager/embedder/main.cc:453
#7  0x00007ffff3087dc5 in (anonymous namespace)::ContentMain((anonymous namespace)::ContentMainParams const&) (params=...) at ../../content/app/content_main.cc:19
#8  0x0000555556cde240 in ChromeMain(int, char const**) (argc=5, argv=0x7fffffffdcc8) at ../../chrome/app/chrome_main.cc:101
#9  0x0000555556cde152 in main(int, char const**) (argc=5, argv=0x7fffffffdcc8) at ../../chrome/app/chrome_exe_main_aura.cc:17
```

```c++
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


BrowserThreadsStarted会去做以下工作：

1. 初始化Mojo，用于IPC通信

2. 如果参数中开启Vulkan，GPU会去做初始化的动作

3. 初始化GPU shader cache

4. BrowserGpuChannelHostFactory的初始化

5. 如果支持WebRTC，初始化WebRTC

6. 初始化ResourceDispatcherHost， MediaStreamManager， SpeechRecognitionManager， UserInputMonitor， SaveFileManager

7. 设置哪些thread可以访问clipboard

8. launch GPU process


创建GPUChannel的时候，UI Thread会将task交由IO Thread处理。

```
Thread 1 "chrome" hit Breakpoint 4, (anonymous namespace)::BrowserGpuChannelHostFactory::EstablishRequest::Create (gpu_client_id=1, gpu_client_tracing_id=18446744073709551615)
    at ../../content/browser/gpu/browser_gpu_channel_host_factory.cc:89
89	  scoped_refptr<EstablishRequest> establish_request =
(gdb) bt
#0  0x00007ffff18959f2 in (anonymous namespace)::BrowserGpuChannelHostFactory::EstablishRequest::Create(int, uint64_t) (gpu_client_id=1, gpu_client_tracing_id=18446744073709551615)
    at ../../content/browser/gpu/browser_gpu_channel_host_factory.cc:89
#1  0x00007ffff1897b96 in (anonymous namespace)::BrowserGpuChannelHostFactory::EstablishGpuChannel((anonymous namespace)::GpuChannelEstablishedCallback) (this=0x15e3527794a0, callback=...) at ../../content/browser/gpu/browser_gpu_channel_host_factory.cc:289
#2  0x00007ffff189718e in (anonymous namespace)::BrowserGpuChannelHostFactory::Initialize(bool) (establish_gpu_channel=true)
    at ../../content/browser/gpu/browser_gpu_channel_host_factory.cc:231
#3  0x00007ffff13d7c27 in (anonymous namespace)::BrowserMainLoop::BrowserThreadsStarted() (this=0x15e351aff920) at ../../content/browser/browser_main_loop.cc:1319
#4  0x00007ffff01ffadd in (anonymous namespace)::(anonymous namespace)::FunctorTraits<void (content::ChildProcess::*)(), void>::Invoke<content::ChildProcess*>(void ((anonymous namespace)::ChildProcess::*)((anonymous namespace)::ChildProcess * const), <unknown type in /sa/src_code/chromium/src/out/chrome_x86_debug/./libcontent.so, CU 0x0, DIE 0x60ee>) (method=(void ((anonymous namespace)::ChildProcess::*)((anonymous namespace)::ChildProcess * const)) 0x7ffff13d7680 <(anonymous namespace)::BrowserMainLoop::BrowserThreadsStarted()>, receiver_ptr=<unknown type in /sa/src_code/chromium/src/out/chrome_x86_debug/./libcontent.so, CU 0x0, DIE 0x60ee>) at ../../base/bind_internal.h:447
#5  0x00007ffff01ffa54 in (anonymous namespace)::(anonymous namespace)::InvokeHelper<false, void>::MakeItSo<void (content::ChildProcess::*)(), content::ChildProcess*>(<unknown type in /sa/src_code/chromium/src/out/chrome_x86_debug/./libcontent.so, CU 0x0, DIE 0x607e>, <unknown type in /sa/src_code/chromium/src/out/chrome_x86_debug/./libcontent.so, CU 0x0, DIE 0x608b>) (functor=<unknown type in /sa/src_code/chromium/src/out/chrome_x86_debug/./libcontent.so, CU 0x0, DIE 0x607e>, args=<unknown type in /sa/src_code/chromium/src/out/chrome_x86_debug/./libcontent.so, CU 0x0, DIE 0x608b>) at ../../base/bind_internal.h:530
#6  0x00007ffff01ffa05 in (anonymous namespace)::(anonymous namespace)::Invoker<base::internal::BindState<void (content::ChildProcess::*)(), base::internal::UnretainedWrapper<content::ChildProcess> >, void ()>::RunImpl<void (content::ChildProcess::*)(), std::__1::tuple<base::internal::UnretainedWrapper<content::ChildProcess> >, 0>(<unknown type in /sa/src_code/chromium/src/out/chrome_x86_debug/./libcontent.so, CU 0x0, DIE 0x3865>, <unknown type in /sa/src_code/chromium/src/out/chrome_x86_debug/./libcontent.so, CU 0x0, DIE 0x3872>, (anonymous namespace)::(anonymous namespace)::index_sequence<0ul>) (functor=<unknown type in /sa/src_code/chromium/src/out/chrome_x86_debug/./libcontent.so, CU 0x0, DIE 0x3865>, bound=<unknown type in /sa/src_code/chromium/src/out/chrome_x86_debug/./libcontent.so, CU 0x0, DIE 0x3872>) at ../../base/bind_internal.h:604
#7  0x00007ffff02028ac in (anonymous namespace)::(anonymous namespace)::Invoker<base::internal::BindState<void (content::ServiceFactory::*)(), base::internal::UnretainedWrapper<content::ServiceFactory> >, void ()>::Run((anonymous namespace)::(anonymous namespace)::BindStateBase *) (base=0x15e352241870) at ../../base/bind_internal.h:586
#8  0x00007ffff011254d in (anonymous namespace)::RepeatingCallback<void ()>::Run(void) const (this=0x15e3526521c0) at ../../base/callback.h:124
#9  0x00007ffff2091ded in (anonymous namespace)::StartupTaskRunner::RunAllTasksNow() (this=0x15e352016140) at ../../content/browser/startup_task_runner.cc:45
#10 0x00007ffff13d6b2f in (anonymous namespace)::BrowserMainLoop::CreateStartupTasks() (this=0x15e351aff920) at ../../content/browser/browser_main_loop.cc:955
#11 0x00007ffff13e3593 in (anonymous namespace)::BrowserMainRunnerImpl::Initialize((anonymous namespace)::MainFunctionParams const&) (this=0x15e351c2d200, parameters=...)
    at ../../content/browser/browser_main_runner.cc:140
#12 0x00007ffff13cea3b in (anonymous namespace)::BrowserMain((anonymous namespace)::MainFunctionParams const&) (parameters=...) at ../../content/browser/browser_main.cc:42
#13 0x00007ffff3089017 in (anonymous namespace)::RunNamedProcessTypeMain((anonymous namespace)::(anonymous namespace)::string const&, (anonymous namespace)::MainFunctionParams const&, (anonymous namespace)::ContentMainDelegate*) (process_type=..., main_function_params=..., delegate=0x7fffffffdb40) at ../../content/app/content_main_runner.cc:427
#14 0x00007ffff308b6f4 in (anonymous namespace)::ContentMainRunnerImpl::Run() (this=0x15e351c2ba40) at ../../content/app/content_main_runner.cc:706
#15 0x00007ffff3081bc5 in (anonymous namespace)::ContentServiceManagerMainDelegate::RunEmbedderProcess() (this=0x7fffffffdaa0)
    at ../../content/app/content_service_manager_main_delegate.cc:51
#16 0x00007ffff7e8fdfc in (anonymous namespace)::Main((anonymous namespace)::MainParams const&) (params=...) at ../../services/service_manager/embedder/main.cc:453
#17 0x00007ffff3087dc5 in (anonymous namespace)::ContentMain((anonymous namespace)::ContentMainParams const&) (params=...) at ../../content/app/content_main.cc:19
#18 0x0000555556cde240 in ChromeMain(int, char const**) (argc=5, argv=0x7fffffffdcc8) at ../../chrome/app/chrome_main.cc:101
#19 0x0000555556cde152 in main(int, char const**) (argc=5, argv=0x7fffffffdcc8) at ../../chrome/app/chrome_exe_main_aura.cc:17
```

IO thread收到这个task以后，会调用GpuProcessHost::Get创建一个新的GpuProcessHost并做初始的工作。初始化过程中，GpuProcessHost会判断是不是single-process mode，如果不是，则启动GPU process。GPU process与render process有所差异，这里是使用BrowserChildProcessHostImpl::Launch调用ChildProcessLauncher完成的。具体的启动过程会交由单独的lancher thread完成。

```
Thread 6 "Chrome_IOThread" hit Breakpoint 3, (anonymous namespace)::(anonymous namespace)::ChildProcessLauncherHelper::StartLaunchOnClientThread (this=0x232fdfa5f520)
    at ../../content/browser/child_process_launcher_helper.cc:88
88	  DCHECK_CURRENTLY_ON(client_thread_id_);
(gdb) bt
#0  0x00007ffff14b3456 in (anonymous namespace)::(anonymous namespace)::ChildProcessLauncherHelper::StartLaunchOnClientThread() (this=0x232fdfa5f520)
    at ../../content/browser/child_process_launcher_helper.cc:88
#1  0x00007ffff14b17cd in (anonymous namespace)::ChildProcessLauncher::ChildProcessLauncher((anonymous namespace)::(anonymous namespace)::unique_ptr<content::SandboxedProcessLauncherDelegate, std::__1::default_delete<content::SandboxedProcessLauncherDelegate> >, (anonymous namespace)::(anonymous namespace)::unique_ptr<base::CommandLine, std::__1::default_delete<base::CommandLine> >, int, (anonymous namespace)::ChildProcessLauncher::Client*, (anonymous namespace)::(anonymous namespace)::unique_ptr<mojo::edk::OutgoingBrokerClientInvitation, std::__1::default_delete<mojo::edk::OutgoingBrokerClientInvitation> >, (anonymous namespace)::(anonymous namespace)::ProcessErrorCallback const&, bool) (this=0x232fdf0c69e0, delegate=..., command_line=..., child_process_id=2, client=0x232fdeffd440, broker_client_invitation=..., process_error_callback=..., terminate_on_shutdown=true)
    at ../../content/browser/child_process_launcher.cc:50
#2  0x00007ffff13bce7c in (anonymous namespace)::BrowserChildProcessHostImpl::Launch((anonymous namespace)::(anonymous namespace)::unique_ptr<content::SandboxedProcessLauncherDelegate, std::__1::default_delete<content::SandboxedProcessLauncherDelegate> >, (anonymous namespace)::(anonymous namespace)::unique_ptr<base::CommandLine, std::__1::default_delete<base::CommandLine> >, bool) (this=0x232fdeffd430, delegate=..., cmd_line=..., terminate_on_shutdown=true) at ../../content/browser/browser_child_process_host_impl.cc:283
#3  0x00007ffff18dd3f6 in (anonymous namespace)::GpuProcessHost::LaunchGpuProcess() (this=0x232fdf401460) at ../../content/browser/gpu/gpu_process_host.cc:1270
#4  0x00007ffff18d8c68 in (anonymous namespace)::GpuProcessHost::Init() (this=0x232fdf401460) at ../../content/browser/gpu/gpu_process_host.cc:782
#5  0x00007ffff18d813f in (anonymous namespace)::GpuProcessHost::Get((anonymous namespace)::GpuProcessHost::GpuProcessKind, bool) (kind=(anonymous namespace)::GpuProcessHost::GPU_PROCESS_KIND_SANDBOXED, force_create=true) at ../../content/browser/gpu/gpu_process_host.cc:489
#6  0x00007ffff1895aeb in (anonymous namespace)::BrowserGpuChannelHostFactory::EstablishRequest::EstablishOnIO() (this=0x232fdfa3f530)
    at ../../content/browser/gpu/browser_gpu_channel_host_factory.cc:117
#7  0x00007ffff0579b3f in (anonymous namespace)::(anonymous namespace)::FunctorTraits<void (content::ServiceManagerConnectionImpl::IOThreadContext::*)(), void>::Invoke<scoped_refptr<content::ServiceManagerConnectionImpl::IOThreadContext>>(void ((anonymous namespace)::ServiceManagerConnectionImpl::IOThreadContext::*)((anonymous namespace)::ServiceManagerConnectionImpl::IOThreadContext * const), <unknown type in /sa/src_code/chromium/src/out/chrome_x86_debug/./libcontent.so, CU 0x0, DIE 0x36941>) (method=(void ((anonymous namespace)::ServiceManagerConnectionImpl::IOThreadContext::*)((anonymous namespace)::ServiceManagerConnectionImpl::IOThreadContext * const)) 0x7ffff1895ab0 <(anonymous namespace)::BrowserGpuChannelHostFactory::EstablishRequest::EstablishOnIO()>, receiver_ptr=<unknown type in /sa/src_code/chromium/src/out/chrome_x86_debug/./libcontent.so, CU 0x0, DIE 0x36941>) at ../../base/bind_internal.h:447
#8  0x00007ffff0579ab4 in (anonymous namespace)::(anonymous namespace)::InvokeHelper<false, void>::MakeItSo<void (content::ServiceManagerConnectionImpl::IOThreadContext::*)(), scoped_refptr<content::ServiceManagerConnectionImpl::IOThreadContext> >(<unknown type in /sa/src_code/chromium/src/out/chrome_x86_debug/./libcontent.so, CU 0x0, DIE 0x368cd>, <unknown type in /sa/src_code/chromium/src/out/chrome_x86_debug/./libcontent.so, CU 0x0, DIE 0x368da>) (functor=<unknown type in /sa/src_code/chromium/src/out/chrome_x86_debug/./libcontent.so, CU 0x0, DIE 0x368cd>, args=<unknown type in /sa/src_code/chromium/src/out/chrome_x86_debug/./libcontent.so, CU 0x0, DIE 0x368da>) at ../../base/bind_internal.h:530
#9  0x00007ffff0579a60 in (anonymous namespace)::(anonymous namespace)::Invoker<base::internal::BindState<void (content::ServiceManagerConnectionImpl::IOThreadContext::*)(), scoped_refptr<content::ServiceManagerConnectionImpl::IOThreadContext> >, void ()>::RunImpl<void (content::ServiceManagerConnectionImpl::IOThreadContext::*)(), std::__1::tuple<scoped_refptr<content::ServiceManagerConnectionImpl::IOThreadContext> >, 0>(<unknown type in /sa/src_code/chromium/src/out/chrome_x86_debug/./libcontent.so, CU 0x0, DIE 0x2703f>, <unknown type in /sa/src_code/chromium/src/out/chrome_x86_debug/./libcontent.so, CU 0x0, DIE 0x2704c>, (anonymous namespace)::(anonymous namespace)::index_sequence<0ul>) (functor=<unknown type in /sa/src_code/chromium/src/out/chrome_x86_debug/./libcontent.so, CU 0x0, DIE 0x2703f>, bound=<unknown type in /sa/src_code/chromium/src/out/chrome_x86_debug/./libcontent.so, CU 0x0, DIE 0x2704c>)
    at ../../base/bind_internal.h:604
#10 0x00007ffff05799b9 in (anonymous namespace)::(anonymous namespace)::Invoker<base::internal::BindState<void (content::ServiceManagerConnectionImpl::IOThreadContext::*)(), scoped_refptr<content::ServiceManagerConnectionImpl::IOThreadContext> >, void ()>::RunOnce((anonymous namespace)::(anonymous namespace)::BindStateBase *) (base=0x232fdf4efe20)
    at ../../base/bind_internal.h:572
#11 0x00007ffff76161ae in (anonymous namespace)::OnceCallback<void ()>::Run(void) (this=0x7fffc7f94fa8) at ../../base/callback.h:95
#12 0x00007ffff766b43f in (anonymous namespace)::(anonymous namespace)::TaskAnnotator::RunTask(char const*, (anonymous namespace)::PendingTask*) (this=0x232fdf11de68, queue_function=0x7ffff750de53 "MessageLoop::PostTask", pending_task=0x7fffc7f94fa8) at ../../base/debug/task_annotator.cc:61
#13 0x00007ffff770b109 in (anonymous namespace)::(anonymous namespace)::IncomingTaskQueue::RunTask((anonymous namespace)::PendingTask*) (this=0x232fdf11de20, pending_task=0x7fffc7f94fa8) at ../../base/message_loop/incoming_task_queue.cc:124
#14 0x00007ffff77142c5 in (anonymous namespace)::MessageLoop::RunTask((anonymous namespace)::PendingTask*) (this=0x232fdf4d23a0, pending_task=0x7fffc7f94fa8)
    at ../../base/message_loop/message_loop.cc:391
#15 0x00007ffff7714548 in (anonymous namespace)::MessageLoop::DeferOrRunPendingTask((anonymous namespace)::PendingTask) (this=0x232fdf4d23a0, pending_task=...)
    at ../../base/message_loop/message_loop.cc:403
#16 0x00007ffff7714879 in (anonymous namespace)::MessageLoop::DoWork() (this=0x232fdf4d23a0) at ../../base/message_loop/message_loop.cc:447
#17 0x00007ffff771a04e in (anonymous namespace)::MessagePumpLibevent::Run((anonymous namespace)::MessagePump::Delegate*) (this=0x232fdf478a70, delegate=0x232fdf4d23a0)
    at ../../base/message_loop/message_pump_libevent.cc:212
#18 0x00007ffff7713a8c in (anonymous namespace)::MessageLoop::Run(bool) (this=0x232fdf4d23a0, application_tasks_allowed=true) at ../../base/message_loop/message_loop.cc:342
#19 0x00007ffff77cabed in (anonymous namespace)::RunLoop::Run() (this=0x7fffc7f95fd0) at ../../base/run_loop.cc:130
#20 0x00007ffff7889388 in (anonymous namespace)::Thread::Run((anonymous namespace)::RunLoop*) (this=0x232fdf4d0e20, run_loop=0x7fffc7f95fd0) at ../../base/threading/thread.cc:255
#21 0x00007ffff13f5d4f in (anonymous namespace)::BrowserProcessSubThread::IOThreadRun((anonymous namespace)::RunLoop*) (this=0x232fdf4d0e20, run_loop=0x7fffc7f95fd0)
    at ../../content/browser/browser_process_sub_thread.cc:155
#22 0x00007ffff13f5c4a in (anonymous namespace)::BrowserProcessSubThread::Run((anonymous namespace)::RunLoop*) (this=0x232fdf4d0e20, run_loop=0x7fffc7f95fd0)
    at ../../content/browser/browser_process_sub_thread.cc:105
#23 0x00007ffff7889ff5 in (anonymous namespace)::Thread::ThreadMain() (this=0x232fdf4d0e20) at ../../base/threading/thread.cc:338
#24 0x00007ffff787fa1d in (anonymous namespace)::(anonymous namespace)::ThreadFunc(void*) (params=0x232fdf6619d0) at ../../base/threading/platform_thread_posix.cc:76
#25 0x00007ffff7bc16ba in start_thread (arg=0x7fffc7f97700) at pthread_create.c:333
#26 0x00007fffd918341d in clone () at ../sysdeps/unix/sysv/linux/x86_64/clone.S:109
```

```c++
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

对于BeforeLaunchOnLauncherThread函数，Android/Linux/Win平台都是直接返回true。

对于PostLaunchOnLauncherThread，除了Android平台，其他都是会执行到的。

LaunchProcessOnLauncherThread会根据平台不同，具体实现细节有所差异。

```
Thread 25 "TaskSchedulerSi" hit Breakpoint 2, (anonymous namespace)::(anonymous namespace)::ChildProcessLauncherHelper::PostLaunchOnLauncherThread (this=0x1819ae6b840, process=..., 
    launch_result=1002) at ../../content/browser/child_process_launcher_helper.cc:136
136	  mojo_client_handle_.reset();
(gdb) ATTENTION: default value of option force_s3tc_enable overridden by environment.
[1274:1274:0413/103734.555064:WARNING:gpu_info.cc(98)] No active GPU found, returning primary GPU.
[1274:1274:0413/103734.555192:ERROR:sandbox_linux.cc(379)] InitializeSandbox() called with multiple threads in process gpu-process.
bt
#0  0x00007ffff14b3d9c in (anonymous namespace)::(anonymous namespace)::ChildProcessLauncherHelper::PostLaunchOnLauncherThread((anonymous namespace)::(anonymous namespace)::ChildProcessLauncherHelper::Process, int) (this=0x1819ae6b840, process=..., launch_result=1002) at ../../content/browser/child_process_launcher_helper.cc:136
#1  0x00007ffff14b3c0c in (anonymous namespace)::(anonymous namespace)::ChildProcessLauncherHelper::LaunchOnLauncherThread() (this=0x1819ae6b840)
    at ../../content/browser/child_process_launcher_helper.cc:126
#2  0x00007ffff0579b3f in (anonymous namespace)::(anonymous namespace)::FunctorTraits<void (content::ServiceManagerConnectionImpl::IOThreadContext::*)(), void>::Invoke<scoped_refptr<content::ServiceManagerConnectionImpl::IOThreadContext>>(void ((anonymous namespace)::ServiceManagerConnectionImpl::IOThreadContext::*)((anonymous namespace)::ServiceManagerConnectionImpl::IOThreadContext * const), <unknown type in /sa/src_code/chromium/src/out/chrome_x86_debug/./libcontent.so, CU 0x0, DIE 0x36941>) (method=(void ((anonymous namespace)::ServiceManagerConnectionImpl::IOThreadContext::*)((anonymous namespace)::ServiceManagerConnectionImpl::IOThreadContext * const)) 0x7ffff14b3790 <(anonymous namespace)::(anonymous namespace)::ChildProcessLauncherHelper::LaunchOnLauncherThread()>, receiver_ptr=<unknown type in /sa/src_code/chromium/src/out/chrome_x86_debug/./libcontent.so, CU 0x0, DIE 0x36941>)
    at ../../base/bind_internal.h:447
#3  0x00007ffff0579ab4 in (anonymous namespace)::(anonymous namespace)::InvokeHelper<false, void>::MakeItSo<void (content::ServiceManagerConnectionImpl::IOThreadContext::*)(), scoped_refptr<content::ServiceManagerConnectionImpl::IOThreadContext> >(<unknown type in /sa/src_code/chromium/src/out/chrome_x86_debug/./libcontent.so, CU 0x0, DIE 0x368cd>, <unknown type in /sa/src_code/chromium/src/out/chrome_x86_debug/./libcontent.so, CU 0x0, DIE 0x368da>) (functor=<unknown type in /sa/src_code/chromium/src/out/chrome_x86_debug/./libcontent.so, CU 0x0, DIE 0x368cd>, args=<unknown type in /sa/src_code/chromium/src/out/chrome_x86_debug/./libcontent.so, CU 0x0, DIE 0x368da>) at ../../base/bind_internal.h:530
#4  0x00007ffff0579a60 in (anonymous namespace)::(anonymous namespace)::Invoker<base::internal::BindState<void (content::ServiceManagerConnectionImpl::IOThreadContext::*)(), scoped_refptr<content::ServiceManagerConnectionImpl::IOThreadContext> >, void ()>::RunImpl<void (content::ServiceManagerConnectionImpl::IOThreadContext::*)(), std::__1::tuple<scoped_refptr<content::ServiceManagerConnectionImpl::IOThreadContext> >, 0>(<unknown type in /sa/src_code/chromium/src/out/chrome_x86_debug/./libcontent.so, CU 0x0, DIE 0x2703f>, <unknown type in /sa/src_code/chromium/src/out/chrome_x86_debug/./libcontent.so, CU 0x0, DIE 0x2704c>, (anonymous namespace)::(anonymous namespace)::index_sequence<0ul>) (functor=<unknown type in /sa/src_code/chromium/src/out/chrome_x86_debug/./libcontent.so, CU 0x0, DIE 0x2703f>, bound=<unknown type in /sa/src_code/chromium/src/out/chrome_x86_debug/./libcontent.so, CU 0x0, DIE 0x2704c>)
    at ../../base/bind_internal.h:604
#5  0x00007ffff05799b9 in (anonymous namespace)::(anonymous namespace)::Invoker<base::internal::BindState<void (content::ServiceManagerConnectionImpl::IOThreadContext::*)(), scoped_refptr<content::ServiceManagerConnectionImpl::IOThreadContext> >, void ()>::RunOnce((anonymous namespace)::(anonymous namespace)::BindStateBase *) (base=0x1819a75c410)
    at ../../base/bind_internal.h:572
#6  0x00007ffff76161ae in (anonymous namespace)::OnceCallback<void ()>::Run(void) (this=0x7fffbda95118) at ../../base/callback.h:95
#7  0x00007ffff766b43f in (anonymous namespace)::(anonymous namespace)::TaskAnnotator::RunTask(char const*, (anonymous namespace)::PendingTask*) (this=0x1819a2d6c28, queue_function=0x0, pending_task=0x7fffbda95118) at ../../base/debug/task_annotator.cc:61
#8  0x00007ffff786fb52 in (anonymous namespace)::(anonymous namespace)::TaskTracker::RunOrSkipTask((anonymous namespace)::(anonymous namespace)::Task, (anonymous namespace)::(anonymous namespace)::Sequence*, bool) (this=0x1819a2d6c20, task=..., sequence=0x1819a4531c0, can_run_task=true) at ../../base/task_scheduler/task_tracker.cc:460
#9  0x00007ffff7873aa6 in (anonymous namespace)::(anonymous namespace)::TaskTrackerPosix::RunOrSkipTask((anonymous namespace)::(anonymous namespace)::Task, (anonymous namespace)::(anonymous namespace)::Sequence*, bool) (this=0x1819a2d6c20, task=..., sequence=0x1819a4531c0, can_run_task=true) at ../../base/task_scheduler/task_tracker_posix.cc:25
#10 0x00007ffff786dc23 in (anonymous namespace)::(anonymous namespace)::TaskTracker::RunAndPopNextTask(scoped_refptr<base::internal::Sequence>, (anonymous namespace)::(anonymous namespace)::CanScheduleSequenceObserver*) (this=0x1819a2d6c20, sequence=..., observer=0x1819ae62e30) at ../../base/task_scheduler/task_tracker.cc:353
#11 0x00007ffff7859c03 in (anonymous namespace)::(anonymous namespace)::SchedulerWorker::Thread::ThreadMain() (this=0x1819ae811a0) at ../../base/task_scheduler/scheduler_worker.cc:85
#12 0x00007ffff787fa1d in (anonymous namespace)::(anonymous namespace)::ThreadFunc(void*) (params=0x1819ae61110) at ../../base/threading/platform_thread_posix.cc:76
#13 0x00007ffff7bc16ba in start_thread (arg=0x7fffbda96700) at pthread_create.c:333
#14 0x00007fffd918341d in clone () at ../sysdeps/unix/sysv/linux/x86_64/clone.S:109

```

## Render Process的启动过程

在上面介绍的GPU Process启动过程中，主线程会给自己喂几个startup tasks，譬如：PreCreateThreads，CreateThreads，PostCreateThreads，BrowserThreadsStarted和PreMainMessageLoopRun。GPU Process的创建过程是在BrowserThreadsStarted task中完成的，而下面要介绍的Render Process则是在PreMainMessageLoopRun task中完成。

```
Thread 1 "chrome" hit Breakpoint 3, (anonymous namespace)::(anonymous namespace)::ChildProcessLauncherHelper::StartLaunchOnClientThread (this=0x232fdf1c0f20)
    at ../../content/browser/child_process_launcher_helper.cc:88
88	  DCHECK_CURRENTLY_ON(client_thread_id_);
(gdb) bt
#0  0x00007ffff14b3456 in (anonymous namespace)::(anonymous namespace)::ChildProcessLauncherHelper::StartLaunchOnClientThread() (this=0x232fdf1c0f20)
    at ../../content/browser/child_process_launcher_helper.cc:88
#1  0x00007ffff14b17cd in (anonymous namespace)::ChildProcessLauncher::ChildProcessLauncher((anonymous namespace)::(anonymous namespace)::unique_ptr<content::SandboxedProcessLauncherDelegate, std::__1::default_delete<content::SandboxedProcessLauncherDelegate> >, (anonymous namespace)::(anonymous namespace)::unique_ptr<base::CommandLine, std::__1::default_delete<base::CommandLine> >, int, (anonymous namespace)::ChildProcessLauncher::Client*, (anonymous namespace)::(anonymous namespace)::unique_ptr<mojo::edk::OutgoingBrokerClientInvitation, std::__1::default_delete<mojo::edk::OutgoingBrokerClientInvitation> >, (anonymous namespace)::(anonymous namespace)::ProcessErrorCallback const&, bool) (this=0x232fe027d260, delegate=..., command_line=..., child_process_id=3, client=0x232fe049b688, broker_client_invitation=..., process_error_callback=..., terminate_on_shutdown=true)
    at ../../content/browser/child_process_launcher.cc:50
#2  0x00007ffff1dd7006 in (anonymous namespace)::RenderProcessHostImpl::Init() (this=0x232fe049b620) at ../../content/browser/renderer_host/render_process_host_impl.cc:1629
#3  0x00007ffff186e0ea in (anonymous namespace)::RenderFrameHostManager::InitRenderView((anonymous namespace)::RenderViewHostImpl*, (anonymous namespace)::RenderFrameProxyHost*) (this=0x232fdf82aab0, render_view_host=0x232fe010bf40, proxy=0x0) at ../../content/browser/frame_host/render_frame_host_manager.cc:1871
#4  0x00007ffff1867115 in (anonymous namespace)::RenderFrameHostManager::ReinitializeRenderFrame((anonymous namespace)::RenderFrameHostImpl*) (this=0x232fdf82aab0, render_frame_host=0x232fe0397020) at ../../content/browser/frame_host/render_frame_host_manager.cc:2018
#5  0x00007ffff1865f8f in (anonymous namespace)::RenderFrameHostManager::GetFrameHostForNavigation((anonymous namespace)::NavigationRequest const&) (this=0x232fdf82aab0, request=...)
    at ../../content/browser/frame_host/render_frame_host_manager.cc:624
#6  0x00007ffff1864ee3 in (anonymous namespace)::RenderFrameHostManager::DidCreateNavigationRequest((anonymous namespace)::NavigationRequest*) (this=0x232fdf82aab0, request=0x232fe034ac20) at ../../content/browser/frame_host/render_frame_host_manager.cc:469
#7  0x00007ffff1789dac in (anonymous namespace)::FrameTreeNode::CreatedNavigationRequest((anonymous namespace)::(anonymous namespace)::unique_ptr<content::NavigationRequest, std::__1::default_delete<content::NavigationRequest> >) (this=0x232fdf82aaa0, navigation_request=...) at ../../content/browser/frame_host/frame_tree_node.cc:498
#8  0x00007ffff17f8d21 in (anonymous namespace)::NavigatorImpl::RequestNavigation((anonymous namespace)::FrameTreeNode*, GURL const&, (anonymous namespace)::Referrer const&, (anonymous namespace)::FrameNavigationEntry const&, (anonymous namespace)::NavigationEntryImpl const&, (anonymous namespace)::ReloadType, (anonymous namespace)::PreviewsState, bool, bool, scoped_refptr<network::ResourceRequestBody> const&, (anonymous namespace)::TimeTicks, (anonymous namespace)::(anonymous namespace)::unique_ptr<content::NavigationUIData, std::__1::default_delete<content::NavigationUIData> >) (this=0x232fe01b44f0, frame_tree_node=0x232fdf82aaa0, dest_url=..., dest_referrer=..., frame_entry=..., entry=..., reload_type=(anonymous namespace)::ReloadType::NONE, previews_state=0, is_same_document_history_load=false, is_history_navigation_in_new_child=false, post_body=..., navigation_start=..., navigation_ui_data=...)
    at ../../content/browser/frame_host/navigator_impl.cc:1086
#9  0x00007ffff17f7fd3 in (anonymous namespace)::NavigatorImpl::NavigateToEntry((anonymous namespace)::FrameTreeNode*, (anonymous namespace)::FrameNavigationEntry const&, (anonymous namespace)::NavigationEntryImpl const&, (anonymous namespace)::ReloadType, bool, bool, bool, scoped_refptr<network::ResourceRequestBody> const&, (anonymous namespace)::(anonymous namespace)::unique_ptr<content::NavigationUIData, std::__1::default_delete<content::NavigationUIData> >) (this=0x232fe01b44f0, frame_tree_node=0x232fdf82aaa0, frame_entry=..., entry=..., reload_type=(anonymous namespace)::ReloadType::NONE, is_same_document_history_load=false, is_history_navigation_in_new_child=false, is_pending_entry=true, post_body=..., navigation_ui_data=...) at ../../content/browser/frame_host/navigator_impl.cc:341
#10 0x00007ffff17f93b9 in (anonymous namespace)::NavigatorImpl::NavigateToPendingEntry((anonymous namespace)::FrameTreeNode*, (anonymous namespace)::FrameNavigationEntry const&, (anonymous namespace)::ReloadType, bool, (anonymous namespace)::(anonymous namespace)::unique_ptr<content::NavigationUIData, std::__1::default_delete<content::NavigationUIData> >) (this=0x232fe01b44f0, frame_tree_node=0x232fdf82aaa0, frame_entry=..., reload_type=(anonymous namespace)::ReloadType::NONE, is_same_document_history_load=false, navigation_ui_data=...)
    at ../../content/browser/frame_host/navigator_impl.cc:390
#11 0x00007ffff17b7a09 in (anonymous namespace)::NavigationControllerImpl::NavigateToPendingEntryInternal((anonymous namespace)::ReloadType, (anonymous namespace)::(anonymous namespace)::unique_ptr<content::NavigationUIData, std::__1::default_delete<content::NavigationUIData> >) (this=0x232fe03872c8, reload_type=(anonymous namespace)::ReloadType::NONE, navigation_ui_data=...) at ../../content/browser/frame_host/navigation_controller_impl.cc:2157
#12 0x00007ffff17a72c3 in (anonymous namespace)::NavigationControllerImpl::NavigateToPendingEntry((anonymous namespace)::ReloadType, (anonymous namespace)::(anonymous namespace)::unique_ptr<content::NavigationUIData, std::__1::default_delete<content::NavigationUIData> >) (this=0x232fe03872c8, reload_type=(anonymous namespace)::ReloadType::NONE, navigation_ui_data=...) at ../../content/browser/frame_host/navigation_controller_impl.cc:2111
#13 0x00007ffff17a7bbe in (anonymous namespace)::NavigationControllerImpl::LoadEntry((anonymous namespace)::(anonymous namespace)::unique_ptr<content::NavigationEntryImpl, std::__1::default_delete<content::NavigationEntryImpl> >, (anonymous namespace)::(anonymous namespace)::unique_ptr<content::NavigationUIData, std::__1::default_delete<content::NavigationUIData> >) (this=0x232fe03872c8, entry=..., navigation_ui_data=...) at ../../content/browser/frame_host/navigation_controller_impl.cc:512
#14 0x00007ffff17abd53 in (anonymous namespace)::NavigationControllerImpl::LoadURLWithParams((anonymous namespace)::NavigationController::LoadURLParams const&) (this=0x232fe03872c8, params=...) at ../../content/browser/frame_host/navigation_controller_impl.cc:870
#15 0x000055555c457974 in (anonymous namespace)::LoadURLInContents((anonymous namespace)::WebContents*, GURL const&, NavigateParams*) (target_contents=0x232fe0387220, url=..., params=0x7fffffff8d20) at ../../chrome/browser/ui/browser_navigator.cc:341
#16 0x000055555c45483a in Navigate(NavigateParams*) (params=0x7fffffff8d20) at ../../chrome/browser/ui/browser_navigator.cc:633
#17 0x000055555c4b6e43 in StartupBrowserCreatorImpl::OpenTabsInBrowser(Browser*, bool, StartupTabs const&) (this=0x7fffffff9af0, browser=0x232fe0250de0, process_startup=true, tabs=...)
    at ../../chrome/browser/ui/startup/startup_browser_creator_impl.cc:532
#18 0x000055555c4b8c1a in StartupBrowserCreatorImpl::RestoreOrCreateBrowser(StartupTabs const&, StartupBrowserCreatorImpl::BrowserOpenBehavior, SessionRestore::BehaviorBitmask, bool, bool) (this=0x7fffffff9af0, tabs=..., behavior=StartupBrowserCreatorImpl::BrowserOpenBehavior::NEW, restore_options=0, process_startup=true, is_post_crash_launch=true)
    at ../../chrome/browser/ui/startup/startup_browser_creator_impl.cc:854
#19 0x000055555c4b625f in StartupBrowserCreatorImpl::DetermineURLsAndLaunch(bool, (anonymous namespace)::(anonymous namespace)::vector<GURL, std::__1::allocator<GURL> > const&) (this=0x7fffffff9af0, process_startup=true, cmd_line_urls=...) at ../../chrome/browser/ui/startup/startup_browser_creator_impl.cc:730
#20 0x000055555c4b5184 in StartupBrowserCreatorImpl::Launch(Profile*, (anonymous namespace)::(anonymous namespace)::vector<GURL, std::__1::allocator<GURL> > const&, bool) (this=0x7fffff---Type <return> to continue, or q <return> to quit---
ff9af0, profile=0x232fdf04bde0, urls_to_open=..., process_startup=true) at ../../chrome/browser/ui/startup/startup_browser_creator_impl.cc:427
#21 0x000055555c4aecea in StartupBrowserCreator::LaunchBrowser((anonymous namespace)::CommandLine const&, Profile*, (anonymous namespace)::FilePath const&, (anonymous namespace)::(anonymous namespace)::IsProcessStartup, (anonymous namespace)::(anonymous namespace)::IsFirstRun) (this=0x232fdf9503e0, command_line=..., profile=0x232fdf04bde0, cur_dir=..., process_startup=(anonymous namespace)::(anonymous namespace)::IS_PROCESS_STARTUP, is_first_run=(anonymous namespace)::(anonymous namespace)::IS_NOT_FIRST_RUN)
    at ../../chrome/browser/ui/startup/startup_browser_creator.cc:353
#22 0x000055555c4b198c in StartupBrowserCreator::ProcessLastOpenedProfiles((anonymous namespace)::CommandLine const&, (anonymous namespace)::FilePath const&, (anonymous namespace)::(anonymous namespace)::IsProcessStartup, (anonymous namespace)::(anonymous namespace)::IsFirstRun, Profile*, StartupBrowserCreator::Profiles const&) (this=0x232fdf9503e0, command_line=..., cur_dir=..., is_process_startup=(anonymous namespace)::(anonymous namespace)::IS_PROCESS_STARTUP, is_first_run=(anonymous namespace)::(anonymous namespace)::IS_NOT_FIRST_RUN, last_used_profile=0x232fdf04bde0, last_opened_profiles=...) at ../../chrome/browser/ui/startup/startup_browser_creator.cc:833
#23 0x000055555c4b0ec7 in StartupBrowserCreator::LaunchBrowserForLastProfiles((anonymous namespace)::CommandLine const&, (anonymous namespace)::FilePath const&, bool, Profile*, StartupBrowserCreator::Profiles const&) (this=0x232fdf9503e0, command_line=..., cur_dir=..., process_startup=true, last_used_profile=0x232fdf04bde0, last_opened_profiles=...)
    at ../../chrome/browser/ui/startup/startup_browser_creator.cc:763
#24 0x000055555c4ae89d in StartupBrowserCreator::ProcessCmdLineImpl((anonymous namespace)::CommandLine const&, (anonymous namespace)::FilePath const&, bool, Profile*, StartupBrowserCreator::Profiles const&) (this=0x232fdf9503e0, command_line=..., cur_dir=..., process_startup=true, last_used_profile=0x232fdf04bde0, last_opened_profiles=...)
    at ../../chrome/browser/ui/startup/startup_browser_creator.cc:725
#25 0x000055555c4ad002 in StartupBrowserCreator::Start((anonymous namespace)::CommandLine const&, (anonymous namespace)::FilePath const&, Profile*, StartupBrowserCreator::Profiles const&) (this=0x232fdf9503e0, cmd_line=..., cur_dir=..., last_used_profile=0x232fdf04bde0, last_opened_profiles=...) at ../../chrome/browser/ui/startup/startup_browser_creator.cc:308
#26 0x000055555885341f in ChromeBrowserMainParts::PreMainMessageLoopRunImpl() (this=0x232fdef006e0) at ../../chrome/browser/chrome_browser_main.cc:2002
#27 0x000055555885158e in ChromeBrowserMainParts::PreMainMessageLoopRun() (this=0x232fdef006e0) at ../../chrome/browser/chrome_browser_main.cc:1444
#28 0x00007ffff13da98c in (anonymous namespace)::BrowserMainLoop::PreMainMessageLoopRun() (this=0x232fdeddc920) at ../../content/browser/browser_main_loop.cc:1042
#29 0x00007ffff01ffadd in (anonymous namespace)::(anonymous namespace)::FunctorTraits<void (content::ChildProcess::*)(), void>::Invoke<content::ChildProcess*>(void ((anonymous namespace)::ChildProcess::*)((anonymous namespace)::ChildProcess * const), <unknown type in /sa/src_code/chromium/src/out/chrome_x86_debug/./libcontent.so, CU 0x0, DIE 0x60ee>) (method=(void ((anonymous namespace)::ChildProcess::*)((anonymous namespace)::ChildProcess * const)) 0x7ffff13da850 <(anonymous namespace)::BrowserMainLoop::PreMainMessageLoopRun()>, receiver_ptr=<unknown type in /sa/src_code/chromium/src/out/chrome_x86_debug/./libcontent.so, CU 0x0, DIE 0x60ee>) at ../../base/bind_internal.h:447
#30 0x00007ffff01ffa54 in (anonymous namespace)::(anonymous namespace)::InvokeHelper<false, void>::MakeItSo<void (content::ChildProcess::*)(), content::ChildProcess*>(<unknown type in /sa/src_code/chromium/src/out/chrome_x86_debug/./libcontent.so, CU 0x0, DIE 0x607e>, <unknown type in /sa/src_code/chromium/src/out/chrome_x86_debug/./libcontent.so, CU 0x0, DIE 0x608b>) (functor=<unknown type in /sa/src_code/chromium/src/out/chrome_x86_debug/./libcontent.so, CU 0x0, DIE 0x607e>, args=<unknown type in /sa/src_code/chromium/src/out/chrome_x86_debug/./libcontent.so, CU 0x0, DIE 0x608b>) at ../../base/bind_internal.h:530
#31 0x00007ffff01ffa05 in (anonymous namespace)::(anonymous namespace)::Invoker<base::internal::BindState<void (content::ChildProcess::*)(), base::internal::UnretainedWrapper<content::ChildProcess> >, void ()>::RunImpl<void (content::ChildProcess::*)(), std::__1::tuple<base::internal::UnretainedWrapper<content::ChildProcess> >, 0>(<unknown type in /sa/src_code/chromium/src/out/chrome_x86_debug/./libcontent.so, CU 0x0, DIE 0x3865>, <unknown type in /sa/src_code/chromium/src/out/chrome_x86_debug/./libcontent.so, CU 0x0, DIE 0x3872>, (anonymous namespace)::(anonymous namespace)::index_sequence<0ul>) (functor=<unknown type in /sa/src_code/chromium/src/out/chrome_x86_debug/./libcontent.so, CU 0x0, DIE 0x3865>, bound=<unknown type in /sa/src_code/chromium/src/out/chrome_x86_debug/./libcontent.so, CU 0x0, DIE 0x3872>) at ../../base/bind_internal.h:604
#32 0x00007ffff02028ac in (anonymous namespace)::(anonymous namespace)::Invoker<base::internal::BindState<void (content::ServiceFactory::*)(), base::internal::UnretainedWrapper<content::ServiceFactory> >, void ()>::Run((anonymous namespace)::(anonymous namespace)::BindStateBase *) (base=0x232fdf513170) at ../../base/bind_internal.h:586
#33 0x00007ffff011254d in (anonymous namespace)::RepeatingCallback<void ()>::Run(void) const (this=0x232fdf661440) at ../../base/callback.h:124
#34 0x00007ffff2091ded in (anonymous namespace)::StartupTaskRunner::RunAllTasksNow() (this=0x232fdf2f1f20) at ../../content/browser/startup_task_runner.cc:45
#35 0x00007ffff13d6b2f in (anonymous namespace)::BrowserMainLoop::CreateStartupTasks() (this=0x232fdeddc920) at ../../content/browser/browser_main_loop.cc:955
#36 0x00007ffff13e3593 in (anonymous namespace)::BrowserMainRunnerImpl::Initialize((anonymous namespace)::MainFunctionParams const&) (this=0x232fdedbaec0, parameters=...)
    at ../../content/browser/browser_main_runner.cc:140
#37 0x00007ffff13cea3b in (anonymous namespace)::BrowserMain((anonymous namespace)::MainFunctionParams const&) (parameters=...) at ../../content/browser/browser_main.cc:42
#38 0x00007ffff3089017 in (anonymous namespace)::RunNamedProcessTypeMain((anonymous namespace)::(anonymous namespace)::string const&, (anonymous namespace)::MainFunctionParams const&, (anonymous namespace)::ContentMainDelegate*) (process_type=..., main_function_params=..., delegate=0x7fffffffdb40) at ../../content/app/content_main_runner.cc:427
#39 0x00007ffff308b6f4 in (anonymous namespace)::ContentMainRunnerImpl::Run() (this=0x232fdeef7a40) at ../../content/app/content_main_runner.cc:706
#40 0x00007ffff3081bc5 in (anonymous namespace)::ContentServiceManagerMainDelegate::RunEmbedderProcess() (this=0x7fffffffdaa0)
    at ../../content/app/content_service_manager_main_delegate.cc:51
#41 0x00007ffff7e8fdfc in (anonymous namespace)::Main((anonymous namespace)::MainParams const&) (params=...) at ../../services/service_manager/embedder/main.cc:453
#42 0x00007ffff3087dc5 in (anonymous namespace)::ContentMain((anonymous namespace)::ContentMainParams const&) (params=...) at ../../content/app/content_main.cc:19
#43 0x0000555556cde240 in ChromeMain(int, char const**) (argc=5, argv=0x7fffffffdcc8) at ../../chrome/app/chrome_main.cc:101
#44 0x0000555556cde152 in main(int, char const**) (argc=5, argv=0x7fffffffdcc8) at ../../chrome/app/chrome_exe_main_aura.cc:17
```



Utility_process_host
Child_process_host
Gpu_process_host
Ppapi_plugin_process_host
Render_process_host
Brower_child_process_host
Mock_render_process_host
Profiling_process_host
Native_message_process_host



```c++
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

Linux下面的Chromium默认是会启动Zygote Process，再由Zygote Process负责fork出render process，详细介绍可以参看：[Linux Zygote Process](https://chromium.googlesource.com/chromium/src/+/lkcr/docs/linux_zygote.md)


```
(gdb) bt
#0  0x00007ffff7965795 in (anonymous namespace)::UnixDomainSocket::SendMsg(int, void const*, size_t, (anonymous namespace)::(anonymous namespace)::vector<int, std::__1::allocator<int> > const&) (fd=312, buf=0x1b41fd737420, length=8, fds=...) at ../../base/posix/unix_domain_socket.cc:74
#1  0x00007ffff21bac9c in (anonymous namespace)::ZygoteCommunication::SendMessage((anonymous namespace)::Pickle const&, (anonymous namespace)::(anonymous namespace)::vector<int, std::__1::allocator<int> > const*) (this=0x1b41fd6cd910, data=..., fds=0x0) at ../../content/browser/zygote_host/zygote_communication_linux.cc:51
#2  0x00007ffff21bd2de in (anonymous namespace)::ZygoteCommunication::Init((anonymous namespace)::OnceCallback<int (base::CommandLine *, base::ScopedGeneric<int, base::internal::ScopedFDCloseTraits> *)>) (this=0x1b41fd6cd910, launcher=...) at ../../content/browser/zygote_host/zygote_communication_linux.cc:256
#3  0x00007ffff2346fc0 in (anonymous namespace)::CreateGenericZygote((anonymous namespace)::OnceCallback<int (base::CommandLine *, base::ScopedGeneric<int, base::internal::ScopedFDCloseTraits> *)>) (launcher=...) at ../../content/browser/zygote_host/zygote_handle_linux.cc:21
#4  0x00007ffff13d2e11 in (anonymous namespace)::(anonymous namespace)::SetupSandbox((anonymous namespace)::CommandLine const&) (parsed_command_line=...)
    at ../../content/browser/browser_main_loop.cc:312
#5  0x00007ffff13d2786 in (anonymous namespace)::BrowserMainLoop::EarlyInitialization() (this=0x1b41fd4cd920) at ../../content/browser/browser_main_loop.cc:619
#6  0x00007ffff13e3310 in (anonymous namespace)::BrowserMainRunnerImpl::Initialize((anonymous namespace)::MainFunctionParams const&) (this=0x1b41fd4abda0, parameters=...)
    at ../../content/browser/browser_main_runner.cc:119
#7  0x00007ffff13cea3b in (anonymous namespace)::BrowserMain((anonymous namespace)::MainFunctionParams const&) (parameters=...) at ../../content/browser/browser_main.cc:42
#8  0x00007ffff3089017 in (anonymous namespace)::RunNamedProcessTypeMain((anonymous namespace)::(anonymous namespace)::string const&, (anonymous namespace)::MainFunctionParams const&, (anonymous namespace)::ContentMainDelegate*) (process_type=..., main_function_params=..., delegate=0x7fffffffdbc0) at ../../content/app/content_main_runner.cc:427
#9  0x00007ffff308b6f4 in (anonymous namespace)::ContentMainRunnerImpl::Run() (this=0x1b41fd5e8890) at ../../content/app/content_main_runner.cc:706
#10 0x00007ffff3081bc5 in (anonymous namespace)::ContentServiceManagerMainDelegate::RunEmbedderProcess() (this=0x7fffffffdb20)
    at ../../content/app/content_service_manager_main_delegate.cc:51
#11 0x00007ffff7e8fdfc in (anonymous namespace)::Main((anonymous namespace)::MainParams const&) (params=...) at ../../services/service_manager/embedder/main.cc:453
#12 0x00007ffff3087dc5 in (anonymous namespace)::ContentMain((anonymous namespace)::ContentMainParams const&) (params=...) at ../../content/app/content_main.cc:19
#13 0x0000555556cde240 in ChromeMain(int, char const**) (argc=2, argv=0x7fffffffdd48) at ../../chrome/app/chrome_main.cc:101
#14 0x0000555556cde152 in main(int, char const**) (argc=2, argv=0x7fffffffdd48) at ../../chrome/app/chrome_exe_main_aura.cc:17
```

```
Thread 26 "TaskSchedulerSi" hit Breakpoint 2, (anonymous namespace)::UnixDomainSocket::SendMsg (fd=312, buf=0x1b41fec6bc20, length=700, fds=...)
    at ../../base/posix/unix_domain_socket.cc:74
74	  struct msghdr msg = {};
(gdb) bt
#0  0x00007ffff7965795 in (anonymous namespace)::UnixDomainSocket::SendMsg(int, void const*, size_t, (anonymous namespace)::(anonymous namespace)::vector<int, std::__1::allocator<int> > const&) (fd=312, buf=0x1b41fec6bc20, length=700, fds=...) at ../../base/posix/unix_domain_socket.cc:74
#1  0x00007ffff21bac9c in (anonymous namespace)::ZygoteCommunication::SendMessage((anonymous namespace)::Pickle const&, (anonymous namespace)::(anonymous namespace)::vector<int, std::__1::allocator<int> > const*) (this=0x1b41fd6cd910, data=..., fds=0x7fffbd291a48) at ../../content/browser/zygote_host/zygote_communication_linux.cc:51
#2  0x00007ffff21bc098 in (anonymous namespace)::ZygoteCommunication::ForkRequest((anonymous namespace)::(anonymous namespace)::vector<std::__1::basic_string<char>, std::__1::allocator<std::__1::basic_string<char> > > const&, (anonymous namespace)::FileHandleMappingVector const&, (anonymous namespace)::(anonymous namespace)::string const&) (this=0x1b41fd6cd910, argv=..., mapping=..., process_type=...) at ../../content/browser/zygote_host/zygote_communication_linux.cc:132
#3  0x00007ffff14b658b in (anonymous namespace)::(anonymous namespace)::ChildProcessLauncherHelper::LaunchProcessOnLauncherThread((anonymous namespace)::LaunchOptions const&, (anonymous namespace)::(anonymous namespace)::unique_ptr<content::PosixFileDescriptorInfo, std::__1::default_delete<content::PosixFileDescriptorInfo> >, bool*, int*) (this=0x1b41feb9d700, options=..., files_to_register=..., is_synchronous_launch=0x7fffbd29326f, launch_result=0x7fffbd293268) at ../../content/browser/child_process_launcher_helper_linux.cc:82
#4  0x00007ffff14b3a91 in (anonymous namespace)::(anonymous namespace)::ChildProcessLauncherHelper::LaunchOnLauncherThread() (this=0x1b41feb9d700)
    at ../../content/browser/child_process_launcher_helper.cc:119
#5  0x00007ffff0579b3f in (anonymous namespace)::(anonymous namespace)::FunctorTraits<void (content::ServiceManagerConnectionImpl::IOThreadContext::*)(), void>::Invoke<scoped_refptr<content::ServiceManagerConnectionImpl::IOThreadContext>>(void ((anonymous namespace)::ServiceManagerConnectionImpl::IOThreadContext::*)((anonymous namespace)::ServiceManagerConnectionImpl::IOThreadContext * const), <unknown type in /sa/src_code/chromium/src/out/chrome_x86_debug/./libcontent.so, CU 0x0, DIE 0x36941>) (method=(void ((anonymous namespace)::ServiceManagerConnectionImpl::IOThreadContext::*)((anonymous namespace)::ServiceManagerConnectionImpl::IOThreadContext * const)) 0x7ffff14b3790 <(anonymous namespace)::(anonymous namespace)::ChildProcessLauncherHelper::LaunchOnLauncherThread()>, receiver_ptr=<unknown type in /sa/src_code/chromium/src/out/chrome_x86_debug/./libcontent.so, CU 0x0, DIE 0x36941>)
    at ../../base/bind_internal.h:447
#6  0x00007ffff0579ab4 in (anonymous namespace)::(anonymous namespace)::InvokeHelper<false, void>::MakeItSo<void (content::ServiceManagerConnectionImpl::IOThreadContext::*)(), scoped_refptr<content::ServiceManagerConnectionImpl::IOThreadContext> >(<unknown type in /sa/src_code/chromium/src/out/chrome_x86_debug/./libcontent.so, CU 0x0, DIE 0x368cd>, <unknown type in /sa/src_code/chromium/src/out/chrome_x86_debug/./libcontent.so, CU 0x0, DIE 0x368da>) (functor=<unknown type in /sa/src_code/chromium/src/out/chrome_x86_debug/./libcontent.so, CU 0x0, DIE 0x368cd>, args=<unknown type in /sa/src_code/chromium/src/out/chrome_x86_debug/./libcontent.so, CU 0x0, DIE 0x368da>) at ../../base/bind_internal.h:530
#7  0x00007ffff0579a60 in (anonymous namespace)::(anonymous namespace)::Invoker<base::internal::BindState<void (content::ServiceManagerConnectionImpl::IOThreadContext::*)(), scoped_refptr<content::ServiceManagerConnectionImpl::IOThreadContext> >, void ()>::RunImpl<void (content::ServiceManagerConnectionImpl::IOThreadContext::*)(), std::__1::tuple<scoped_refptr<content::ServiceManagerConnectionImpl::IOThreadContext> >, 0>(<unknown type in /sa/src_code/chromium/src/out/chrome_x86_debug/./libcontent.so, CU 0x0, DIE 0x2703f>, <unknown type in /sa/src_code/chromium/src/out/chrome_x86_debug/./libcontent.so, CU 0x0, DIE 0x2704c>, (anonymous namespace)::(anonymous namespace)::index_sequence<0ul>) (functor=<unknown type in /sa/src_code/chromium/src/out/chrome_x86_debug/./libcontent.so, CU 0x0, DIE 0x2703f>, bound=<unknown type in /sa/src_code/chromium/src/out/chrome_x86_debug/./libcontent.so, CU 0x0, DIE 0x2704c>)
    at ../../base/bind_internal.h:604
#8  0x00007ffff05799b9 in (anonymous namespace)::(anonymous namespace)::Invoker<base::internal::BindState<void (content::ServiceManagerConnectionImpl::IOThreadContext::*)(), scoped_refptr<content::ServiceManagerConnectionImpl::IOThreadContext> >, void ()>::RunOnce((anonymous namespace)::(anonymous namespace)::BindStateBase *) (base=0x1b41febf95d0)
    at ../../base/bind_internal.h:572
#9  0x00007ffff76161ae in (anonymous namespace)::OnceCallback<void ()>::Run(void) (this=0x7fffbd294118) at ../../base/callback.h:95
#10 0x00007ffff766b43f in (anonymous namespace)::(anonymous namespace)::TaskAnnotator::RunTask(char const*, (anonymous namespace)::PendingTask*) (this=0x1b41fd584c28, queue_function=0x0, pending_task=0x7fffbd294118) at ../../base/debug/task_annotator.cc:61
#11 0x00007ffff786fb52 in (anonymous namespace)::(anonymous namespace)::TaskTracker::RunOrSkipTask((anonymous namespace)::(anonymous namespace)::Task, (anonymous namespace)::(anonymous namespace)::Sequence*, bool) (this=0x1b41fd584c20, task=..., sequence=0x1b41fd6edab0, can_run_task=true) at ../../base/task_scheduler/task_tracker.cc:460
#12 0x00007ffff7873aa6 in (anonymous namespace)::(anonymous namespace)::TaskTrackerPosix::RunOrSkipTask((anonymous namespace)::(anonymous namespace)::Task, (anonymous namespace)::(anonymous namespace)::Sequence*, bool) (this=0x1b41fd584c20, task=..., sequence=0x1b41fd6edab0, can_run_task=true) at ../../base/task_scheduler/task_tracker_posix.cc:25
#13 0x00007ffff786dc23 in (anonymous namespace)::(anonymous namespace)::TaskTracker::RunAndPopNextTask(scoped_refptr<base::internal::Sequence>, (anonymous namespace)::(anonymous namespace)::CanScheduleSequenceObserver*) (this=0x1b41fd584c20, sequence=..., observer=0x1b41fe132020) at ../../base/task_scheduler/task_tracker.cc:353
#14 0x00007ffff7859c03 in (anonymous namespace)::(anonymous namespace)::SchedulerWorker::Thread::ThreadMain() (this=0x1b41fe0fcf20) at ../../base/task_scheduler/scheduler_worker.cc:85
#15 0x00007ffff787fa1d in (anonymous namespace)::(anonymous namespace)::ThreadFunc(void*) (params=0x1b41fdfee390) at ../../base/threading/platform_thread_posix.cc:76
#16 0x00007ffff7bc16ba in start_thread (arg=0x7fffbd295700) at pthread_create.c:333
#17 0x00007fffd918341d in clone () at ../sysdeps/unix/sysv/linux/x86_64/clone.S:109

```

```
#0  0x00007ffff7965795 in (anonymous namespace)::UnixDomainSocket::SendMsg(int, void const*, size_t, (anonymous namespace)::(anonymous namespace)::vector<int, std::__1::allocator<int> > const&) (fd=312, buf=0x1b41fe569fa0, length=12, fds=...) at ../../base/posix/unix_domain_socket.cc:74
#1  0x00007ffff21bac9c in (anonymous namespace)::ZygoteCommunication::SendMessage((anonymous namespace)::Pickle const&, (anonymous namespace)::(anonymous namespace)::vector<int, std::__1::allocator<int> > const*) (this=0x1b41fd6cd910, data=..., fds=0x0) at ../../content/browser/zygote_host/zygote_communication_linux.cc:51
#2  0x00007ffff21bccb0 in (anonymous namespace)::ZygoteCommunication::EnsureProcessTerminated(pid_t) (this=0x1b41fd6cd910, process=46062)
    at ../../content/browser/zygote_host/zygote_communication_linux.cc:207
#3  0x00007ffff14b68fb in (anonymous namespace)::(anonymous namespace)::ChildProcessLauncherHelper::ForceNormalProcessTerminationSync((anonymous namespace)::(anonymous namespace)::ChildProcessLauncherHelper::Process) (process=...) at ../../content/browser/child_process_launcher_helper_linux.cc:150
#4  0x00007ffff14b5615 in (anonymous namespace)::(anonymous namespace)::FunctorTraits<void (*)(content::internal::ChildProcessLauncherHelper::Process), void>::Invoke<content::internal::ChildProcessLauncherHelper::Process>(void (*)((anonymous namespace)::(anonymous namespace)::ChildProcessLauncherHelper::Process), <unknown type in /sa/src_code/chromium/src/out/chrome_x86_debug/./libcontent.so, CU 0x0, DIE 0x17ad9>) (function=0x7ffff14b6800 <(anonymous namespace)::(anonymous namespace)::ChildProcessLauncherHelper::ForceNormalProcessTerminationSync((anonymous namespace)::(anonymous namespace)::ChildProcessLauncherHelper::Process)>, args=<unknown type in /sa/src_code/chromium/src/out/chrome_x86_debug/./libcontent.so, CU 0x0, DIE 0x17ad9>) at ../../base/bind_internal.h:402
#5  0x00007ffff14b55c0 in (anonymous namespace)::(anonymous namespace)::InvokeHelper<false, void>::MakeItSo<void (*)(content::internal::ChildProcessLauncherHelper::Process), content::internal::ChildProcessLauncherHelper::Process>(<unknown type in /sa/src_code/chromium/src/out/chrome_x86_debug/./libcontent.so, CU 0x0, DIE 0x17a65>, <unknown type in /sa/src_code/chromium/src/out/chrome_x86_debug/./libcontent.so, CU 0x0, DIE 0x17a72>) (functor=<unknown type in /sa/src_code/chromium/src/out/chrome_x86_debug/./libcontent.so, CU 0x0, DIE 0x17a65>, args=<unknown type in /sa/src_code/chromium/src/out/chrome_x86_debug/./libcontent.so, CU 0x0, DIE 0x17a72>) at ../../base/bind_internal.h:530
#6  0x00007ffff14b5580 in (anonymous namespace)::(anonymous namespace)::Invoker<base::internal::BindState<void (*)(content::internal::ChildProcessLauncherHelper::Process), content::internal::ChildProcessLauncherHelper::Process>, void ()>::RunImpl<void (*)(content::internal::ChildProcessLauncherHelper::Process), std::__1::tuple<content::internal::ChildProcessLauncherHelper::Process>, 0>(<unknown type in /sa/src_code/chromium/src/out/chrome_x86_debug/./libcontent.so, CU 0x0, DIE 0x11870>, <unknown type in /sa/src_code/chromium/src/out/chrome_x86_debug/./libcontent.so, CU 0x0, DIE 0x1187d>, (anonymous namespace)::(anonymous namespace)::index_sequence<0ul>) (functor=<unknown type in /sa/src_code/chromium/src/out/chrome_x86_debug/./libcontent.so, CU 0x0, DIE 0x11870>, bound=<unknown type in /sa/src_code/chromium/src/out/chrome_x86_debug/./libcontent.so, CU 0x0, DIE 0x1187d>) at ../../base/bind_internal.h:604
#7  0x00007ffff14b54d9 in (anonymous namespace)::(anonymous namespace)::Invoker<base::internal::BindState<void (*)(content::internal::ChildProcessLauncherHelper::Process), content::internal::ChildProcessLauncherHelper::Process>, void ()>::RunOnce((anonymous namespace)::(anonymous namespace)::BindStateBase *) (base=0x1b41fec8f5d0) at ../../base/bind_internal.h:572
#8  0x00007ffff76161ae in (anonymous namespace)::OnceCallback<void ()>::Run(void) (this=0x7fffbd294118) at ../../base/callback.h:95
#9  0x00007ffff766b43f in (anonymous namespace)::(anonymous namespace)::TaskAnnotator::RunTask(char const*, (anonymous namespace)::PendingTask*) (this=0x1b41fd584c28, queue_function=0x0, pending_task=0x7fffbd294118) at ../../base/debug/task_annotator.cc:61
#10 0x00007ffff786fb52 in (anonymous namespace)::(anonymous namespace)::TaskTracker::RunOrSkipTask((anonymous namespace)::(anonymous namespace)::Task, (anonymous namespace)::(anonymous namespace)::Sequence*, bool) (this=0x1b41fd584c20, task=..., sequence=0x1b41fd6edab0, can_run_task=true) at ../../base/task_scheduler/task_tracker.cc:460
#11 0x00007ffff7873aa6 in (anonymous namespace)::(anonymous namespace)::TaskTrackerPosix::RunOrSkipTask((anonymous namespace)::(anonymous namespace)::Task, (anonymous namespace)::(anonymous namespace)::Sequence*, bool) (this=0x1b41fd584c20, task=..., sequence=0x1b41fd6edab0, can_run_task=true) at ../../base/task_scheduler/task_tracker_posix.cc:25
#12 0x00007ffff786dc23 in (anonymous namespace)::(anonymous namespace)::TaskTracker::RunAndPopNextTask(scoped_refptr<base::internal::Sequence>, (anonymous namespace)::(anonymous namespace)::CanScheduleSequenceObserver*) (this=0x1b41fd584c20, sequence=..., observer=0x1b41fe132020) at ../../base/task_scheduler/task_tracker.cc:353
#13 0x00007ffff7859c03 in (anonymous namespace)::(anonymous namespace)::SchedulerWorker::Thread::ThreadMain() (this=0x1b41fe0fcf20) at ../../base/task_scheduler/scheduler_worker.cc:85
#14 0x00007ffff787fa1d in (anonymous namespace)::(anonymous namespace)::ThreadFunc(void*) (params=0x1b41fdfee390) at ../../base/threading/platform_thread_posix.cc:76
#15 0x00007ffff7bc16ba in start_thread (arg=0x7fffbd295700) at pthread_create.c:333
#16 0x00007fffd918341d in clone () at ../sysdeps/unix/sysv/linux/x86_64/clone.S:109

```

