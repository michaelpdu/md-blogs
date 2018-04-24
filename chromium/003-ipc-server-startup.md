# IPC通信的连接建立过程

读过Chromium的document以后，知道在Linux环境下面，IPC是通过socketpair实现的。


Linux环境下，Chrome在BrowserMainLoop::EarlyInitialization中会准备sandboxed的socket。

```c++
void SetupSandbox(const base::CommandLine& parsed_command_line) {
  TRACE_EVENT0("startup", "SetupSandbox");
  // SandboxHostLinux needs to be initialized even if the sandbox and
  // zygote are both disabled. It initializes the sandboxed process socket.
  SandboxHostLinux::GetInstance()->Init();

  if (parsed_command_line.HasSwitch(switches::kNoZygote) &&
      !parsed_command_line.HasSwitch(switches::kNoSandbox)) {
    LOG(ERROR) << "--no-sandbox should be used together with --no--zygote";
    exit(EXIT_FAILURE);
  }

  // Tickle the zygote host so it forks now.
  ZygoteHostImpl::GetInstance()->Init(parsed_command_line);
  ZygoteHandle generic_zygote =
      CreateGenericZygote(base::BindOnce(LaunchZygoteHelper));

  // TODO(kerrnel): Investigate doing this without the ZygoteHostImpl as a
  // proxy. It is currently done this way due to concerns about race
  // conditions.
  ZygoteHostImpl::GetInstance()->SetRendererSandboxStatus(
      generic_zygote->GetSandboxStatus());
}
```

```
(gdb) bt
#0  0x00007fffd91845d0 in socketpair () at ../sysdeps/unix/syscall-template.S:84
#1  0x00007ffff1e9f572 in (anonymous namespace)::SandboxHostLinux::Init() (this=0x1c924e4a71a0) at ../../content/browser/sandbox_host_linux.cc:33
#2  0x00007ffff13d2d05 in (anonymous namespace)::(anonymous namespace)::SetupSandbox((anonymous namespace)::CommandLine const&) (parsed_command_line=...)
    at ../../content/browser/browser_main_loop.cc:301
#3  0x00007ffff13d2786 in (anonymous namespace)::BrowserMainLoop::EarlyInitialization() (this=0x1c924e24f920) at ../../content/browser/browser_main_loop.cc:619
#4  0x00007ffff13e3310 in (anonymous namespace)::BrowserMainRunnerImpl::Initialize((anonymous namespace)::MainFunctionParams const&) (this=0x1c924e22dec0, parameters=...)
    at ../../content/browser/browser_main_runner.cc:119
#5  0x00007ffff13cea3b in (anonymous namespace)::BrowserMain((anonymous namespace)::MainFunctionParams const&) (parameters=...) at ../../content/browser/browser_main.cc:42
#6  0x00007ffff3089017 in (anonymous namespace)::RunNamedProcessTypeMain((anonymous namespace)::(anonymous namespace)::string const&, (anonymous namespace)::MainFunctionParams const&, (anonymous namespace)::ContentMainDelegate*) (process_type=..., main_function_params=..., delegate=0x7fffffffdb40) at ../../content/app/content_main_runner.cc:427
#7  0x00007ffff308b6f4 in (anonymous namespace)::ContentMainRunnerImpl::Run() (this=0x1c924e36aa40) at ../../content/app/content_main_runner.cc:706
#8  0x00007ffff3081bc5 in (anonymous namespace)::ContentServiceManagerMainDelegate::RunEmbedderProcess() (this=0x7fffffffdaa0)
    at ../../content/app/content_service_manager_main_delegate.cc:51
#9  0x00007ffff7e8fdfc in (anonymous namespace)::Main((anonymous namespace)::MainParams const&) (params=...) at ../../services/service_manager/embedder/main.cc:453
#10 0x00007ffff3087dc5 in (anonymous namespace)::ContentMain((anonymous namespace)::ContentMainParams const&) (params=...) at ../../content/app/content_main.cc:19
#11 0x0000555556cde240 in ChromeMain(int, char const**) (argc=5, argv=0x7fffffffdcc8) at ../../chrome/app/chrome_main.cc:101
#12 0x0000555556cde152 in main(int, char const**) (argc=5, argv=0x7fffffffdcc8) at ../../chrome/app/chrome_exe_main_aura.cc:17

```

```c++
void SandboxHostLinux::Init() {
  DCHECK(!initialized_);
  initialized_ = true;

  int fds[2];
  // We use SOCK_SEQPACKET rather than SOCK_DGRAM to prevent the sandboxed
  // processes from sending datagrams to other sockets on the system. The
  // sandbox may prevent the sandboxed process from calling socket() to create
  // new sockets, but it'll still inherit some sockets. With AF_UNIX+SOCK_DGRAM,
  // it can call sendmsg to send a datagram to any (abstract) socket on the same
  // system. With SOCK_SEQPACKET, this is prevented.
  CHECK(socketpair(AF_UNIX, SOCK_SEQPACKET, 0, fds) == 0);

  child_socket_ = fds[0];
  // The SandboxIPC client is not expected to read from |child_socket_|.
  // Instead, it reads from a temporary socket sent with the request.
  PCHECK(0 == shutdown(child_socket_, SHUT_RD)) << "shutdown";

  const int browser_socket = fds[1];
  // The SandboxIPC handler is not expected to write to |browser_socket|.
  // Instead, it replies on a temporary socket provided by the caller.
  PCHECK(0 == shutdown(browser_socket, SHUT_WR)) << "shutdown";

  int pipefds[2];
  CHECK(0 == pipe(pipefds));
  const int child_lifeline_fd = pipefds[0];
  childs_lifeline_fd_ = pipefds[1];

  ipc_handler_.reset(new SandboxIPCHandler(child_lifeline_fd, browser_socket));
  ipc_thread_.reset(
      new base::DelegateSimpleThread(ipc_handler_.get(), "sandbox_ipc_thread"));
  ipc_thread_->Start();
}
```

在SandboxHostLinux::Init函数中，会启动sandbox_ipc_thread去专门处理sandboxes IPC请求，相应的处理函数是在SandboxIPCHandler::Run()中。

```
(gdb) i threads 
  Id   Target Id         Frame 
* 1    Thread 0x7fffce84ea80 (LWP 9124) "chrome" socketpair () at ../sysdeps/unix/syscall-template.S:84
  2    Thread 0x7fffcd599700 (LWP 9914) "sandbox_ipc_thr" 0x00007fffd917774d in poll () at ../sysdeps/unix/syscall-template.S:84
(gdb) thread 2
[Switching to thread 2 (Thread 0x7fffcd599700 (LWP 9914))]
#0  0x00007fffd917774d in poll () at ../sysdeps/unix/syscall-template.S:84
84	../sysdeps/unix/syscall-template.S: No such file or directory.
(gdb) bt
#0  0x00007fffd917774d in poll () at ../sysdeps/unix/syscall-template.S:84
#1  0x00007ffff1ea029d in (anonymous namespace)::SandboxIPCHandler::Run() (this=0x5848e034200) at ../../content/browser/sandbox_ipc_linux.cc:100
#2  0x00007ffff7885e8f in (anonymous namespace)::DelegateSimpleThread::Run() (this=0x5848dfe5700) at ../../base/threading/simple_thread.cc:92
#3  0x00007ffff7885b87 in (anonymous namespace)::SimpleThread::ThreadMain() (this=0x5848dfe5700) at ../../base/threading/simple_thread.cc:68
#4  0x00007ffff787fa1d in (anonymous namespace)::(anonymous namespace)::ThreadFunc(void*) (params=0x5848df9b700) at ../../base/threading/platform_thread_posix.cc:76
#5  0x00007ffff7bc16ba in start_thread (arg=0x7fffcd599700) at pthread_create.c:333
#6  0x00007fffd918341d in clone () at ../sysdeps/unix/sysv/linux/x86_64/clone.S:109

```

```c++
void SandboxIPCHandler::Run() {
  struct pollfd pfds[2];
  pfds[0].fd = lifeline_fd_;
  pfds[0].events = POLLIN;
  pfds[1].fd = browser_socket_;
  pfds[1].events = POLLIN;

  int failed_polls = 0;
  for (;;) {
    const int r =
        HANDLE_EINTR(poll(pfds, arraysize(pfds), -1 /* no timeout */));
    // '0' is not a possible return value with no timeout.
    DCHECK_NE(0, r);
    if (r < 0) {
      PLOG(WARNING) << "poll";
      if (failed_polls++ == 3) {
        LOG(FATAL) << "poll(2) failing. SandboxIPCHandler aborting.";
        return;
      }
      continue;
    }

    failed_polls = 0;

    // The browser process will close the other end of this pipe on shutdown,
    // so we should exit.
    if (pfds[0].revents) {
      break;
    }

    // If poll(2) reports an error condition in this fd,
    // we assume the zygote is gone and we exit the loop.
    if (pfds[1].revents & (POLLERR | POLLHUP)) {
      break;
    }

    if (pfds[1].revents & POLLIN) {
      HandleRequestFromChild(browser_socket_);
    }
  }

  VLOG(1) << "SandboxIPCHandler stopping.";
}

```

在LaunchZygoteHelper中会获得child socket，作为参数的一部分，启动Zygote。在ZygoteHostImpl::LaunchZygote函数中，会进一步将其作为参数传入到启动process的函数中。

```
(gdb) bt
#0  0x00007fffd91845d0 in socketpair () at ../sysdeps/unix/syscall-template.S:84
#1  0x00007ffff21be88f in (anonymous namespace)::ZygoteHostImpl::LaunchZygote((anonymous namespace)::CommandLine*, (anonymous namespace)::ScopedFD*, (anonymous namespace)::FileHandleMappingVector) (this=0x5848e158860, cmd_line=0x7fffffffbd70, control_fd=0x5848e12bc50, additional_remapped_fds=...) at ../../content/browser/zygote_host/zygote_host_impl_linux.cc:158
#2  0x00007ffff13e0640 in (anonymous namespace)::(anonymous namespace)::LaunchZygoteHelper((anonymous namespace)::CommandLine*, (anonymous namespace)::ScopedFD*) (cmd_line=0x7fffffffbd70, control_fd=0x5848e12bc50) at ../../content/browser/browser_main_loop.cc:293
#3  0x00007ffff13e11d8 in (anonymous namespace)::(anonymous namespace)::FunctorTraits<int (*)(base::CommandLine*, base::ScopedGeneric<int, base::internal::ScopedFDCloseTraits>*), void>::Invoke<base::CommandLine*, base::ScopedGeneric<int, base::internal::ScopedFDCloseTraits>*>(int (*)((anonymous namespace)::CommandLine *, (anonymous namespace)::ScopedGeneric<int, base::internal::ScopedFDCloseTraits> *), <unknown type in /sa/src_code/chromium/src/out/chrome_x86_debug/./libcontent.so, CU 0x0, DIE 0x50bb5>, <unknown type in /sa/src_code/chromium/src/out/chrome_x86_debug/./libcontent.so, CU 0x0, DIE 0x50bc2>) (function=
    0x7ffff13e0050 <(anonymous namespace)::(anonymous namespace)::LaunchZygoteHelper((anonymous namespace)::CommandLine*, (anonymous namespace)::ScopedFD*)>, args=<unknown type in /sa/src_code/chromium/src/out/chrome_x86_debug/./libcontent.so, CU 0x0, DIE 0x50bb5>, args=<unknown type in /sa/src_code/chromium/src/out/chrome_x86_debug/./libcontent.so, CU 0x0, DIE 0x50bc2>) at ../../base/bind_internal.h:402
#4  0x00007ffff13e1190 in (anonymous namespace)::(anonymous namespace)::InvokeHelper<false, int>::MakeItSo<int (*)(base::CommandLine*, base::ScopedGeneric<int, base::internal::ScopedFDCloseTraits>*), base::CommandLine*, base::ScopedGeneric<int, base::internal::ScopedFDCloseTraits>*>(<unknown type in /sa/src_code/chromium/src/out/chrome_x86_debug/./libcontent.so, CU 0x0, DIE 0x50b18>, <unknown type in /sa/src_code/chromium/src/out/chrome_x86_debug/./libcontent.so, CU 0x0, DIE 0x50b25>, <unknown type in /sa/src_code/chromium/src/out/chrome_x86_debug/./libcontent.so, CU 0x0, DIE 0x50b32>) (functor=<unknown type in /sa/src_code/chromium/src/out/chrome_x86_debug/./libcontent.so, CU 0x0, DIE 0x50b18>, args=<unknown type in /sa/src_code/chromium/src/out/chrome_x86_debug/./libcontent.so, CU 0x0, DIE 0x50b25>, args=<unknown type in /sa/src_code/chromium/src/out/chrome_x86_debug/./libcontent.so, CU 0x0, DIE 0x50b32>)
    at ../../base/bind_internal.h:530
#5  0x00007ffff13e1141 in (anonymous namespace)::(anonymous namespace)::Invoker<base::internal::BindState<int (*)(base::CommandLine *, base::ScopedGeneric<int, base::internal::ScopedFDCloseTraits> *)>, int (base::CommandLine *, base::ScopedGeneric<int, base::internal::ScopedFDCloseTraits> *)>::RunImpl<int (*)(base::CommandLine *, base::ScopedGeneric<int, base::internal::ScopedFDCloseTraits> *), std::__1::tuple<>>(<unknown type in /sa/src_code/chromium/src/out/chrome_x86_debug/./libcontent.so, CU 0x0, DIE 0x45cb8>, <unknown type in /sa/src_code/chromium/src/out/chrome_x86_debug/./libcontent.so, CU 0x0, DIE 0x45cc5>, (anonymous namespace)::(anonymous namespace)::index_sequence<>, <unknown type in /sa/src_code/chromium/src/out/chrome_x86_debug/./libcontent.so, CU 0x0, DIE 0x45cde>, <unknown type in /sa/src_code/chromium/src/out/chrome_x86_debug/./libcontent.so, CU 0x0, DIE 0x45ceb>) (functor=<unknown type in /sa/src_code/chromium/src/out/chrome_x86_debug/./libcontent.so, CU 0x0, DIE 0x45cb8>, bound=<unknown type in /sa/src_code/chromium/src/out/chrome_x86_debug/./libcontent.so, CU 0x0, DIE 0x45cc5>, unbound_args=<unknown type in /sa/src_code/chromium/src/out/chrome_x86_debug/./libcontent.so, CU 0x0, DIE 0x45cde>, unbound_args=<unknown type in /sa/src_code/chromium/src/out/chrome_x86_debug/./libcontent.so, CU 0x0, DIE 0x45ceb>) at ../../base/bind_internal.h:604
#6  0x00007ffff13e10f9 in (anonymous namespace)::(anonymous namespace)::Invoker<base::internal::BindState<int (*)(base::CommandLine *, base::ScopedGeneric<int, base::internal::ScopedFDCloseTraits> *)>, int (base::CommandLine *, base::ScopedGeneric<int, base::internal::ScopedFDCloseTraits> *)>::RunOnce((anonymous namespace)::(anonymous namespace)::BindStateBase *, (anonymous namespace)::(anonymous namespace)::PassingTraitsType<base::CommandLine*>, (anonymous namespace)::(anonymous namespace)::PassingTraitsType<base::ScopedGeneric<int, base::internal::ScopedFDCloseTraits>*>) (base=0x5848df32db0, unbound_args=0x7fffffffbd70, unbound_args=0x5848e12bc50) at ../../base/bind_internal.h:572
#7  0x00007ffff21bdc04 in (anonymous namespace)::OnceCallback<int (base::CommandLine *, base::ScopedGeneric<int, base::internal::ScopedFDCloseTraits> *)>::Run((anonymous namespace)::CommandLine *, (anonymous namespace)::ScopedGeneric<int, base::internal::ScopedFDCloseTraits> *) (this=0x7fffffffc210, args=0x7fffffffbd70, args=0x5848e12bc50) at ../../base/callback.h:95
#8  0x00007ffff21bd2a0 in (anonymous namespace)::ZygoteCommunication::Init((anonymous namespace)::OnceCallback<int (base::CommandLine *, base::ScopedGeneric<int, base::internal::ScopedFDCloseTraits> *)>) (this=0x5848e12bc50, launcher=...) at ../../content/browser/zygote_host/zygote_communication_linux.cc:252
#9  0x00007ffff2346fc0 in (anonymous namespace)::CreateGenericZygote((anonymous namespace)::OnceCallback<int (base::CommandLine *, base::ScopedGeneric<int, base::internal::ScopedFDCloseTraits> *)>) (launcher=...) at ../../content/browser/zygote_host/zygote_handle_linux.cc:21
#10 0x00007ffff13d2e11 in (anonymous namespace)::(anonymous namespace)::SetupSandbox((anonymous namespace)::CommandLine const&) (parsed_command_line=...)
    at ../../content/browser/browser_main_loop.cc:312
#11 0x00007ffff13d2786 in (anonymous namespace)::BrowserMainLoop::EarlyInitialization() (this=0x5848df22920) at ../../content/browser/browser_main_loop.cc:619
#12 0x00007ffff13e3310 in (anonymous namespace)::BrowserMainRunnerImpl::Initialize((anonymous namespace)::MainFunctionParams const&) (this=0x5848deefec0, parameters=...)
    at ../../content/browser/browser_main_runner.cc:119
#13 0x00007ffff13cea3b in (anonymous namespace)::BrowserMain((anonymous namespace)::MainFunctionParams const&) (parameters=...) at ../../content/browser/browser_main.cc:42
#14 0x00007ffff3089017 in (anonymous namespace)::RunNamedProcessTypeMain((anonymous namespace)::(anonymous namespace)::string const&, (anonymous namespace)::MainFunctionParams const&, (anonymous namespace)::ContentMainDelegate*) (process_type=..., main_function_params=..., delegate=0x7fffffffdb40) at ../../content/app/content_main_runner.cc:427
#15 0x00007ffff308b6f4 in (anonymous namespace)::ContentMainRunnerImpl::Run() (this=0x5848e03da40) at ../../content/app/content_main_runner.cc:706
#16 0x00007ffff3081bc5 in (anonymous namespace)::ContentServiceManagerMainDelegate::RunEmbedderProcess() (this=0x7fffffffdaa0)
    at ../../content/app/content_service_manager_main_delegate.cc:51
#17 0x00007ffff7e8fdfc in (anonymous namespace)::Main((anonymous namespace)::MainParams const&) (params=...) at ../../services/service_manager/embedder/main.cc:453
#18 0x00007ffff3087dc5 in (anonymous namespace)::ContentMain((anonymous namespace)::ContentMainParams const&) (params=...) at ../../content/app/content_main.cc:19
#19 0x0000555556cde240 in ChromeMain(int, char const**) (argc=5, argv=0x7fffffffdcc8) at ../../chrome/app/chrome_main.cc:101
#20 0x0000555556cde152 in main(int, char const**) (argc=5, argv=0x7fffffffdcc8) at ../../chrome/app/chrome_exe_main_aura.cc:17

```


```c++
pid_t LaunchZygoteHelper(base::CommandLine* cmd_line,
                         base::ScopedFD* control_fd) {
  // Append any switches from the browser process that need to be forwarded on
  // to the zygote/renderers.
  static const char* const kForwardSwitches[] = {
      switches::kAndroidFontsPath, switches::kClearKeyCdmPathForTesting,
      switches::kEnableHeapProfiling,
      switches::kEnableLogging,  // Support, e.g., --enable-logging=stderr.
      // Need to tell the zygote that it is headless so that we don't try to use
      // the wrong type of main delegate.
      switches::kHeadless,
      // Zygote process needs to know what resources to have loaded when it
      // becomes a renderer process.
      switches::kForceDeviceScaleFactor, switches::kLoggingLevel,
      switches::kPpapiInProcess, switches::kRegisterPepperPlugins, switches::kV,
      switches::kVModule,
  };
  cmd_line->CopySwitchesFrom(*base::CommandLine::ForCurrentProcess(),
                             kForwardSwitches, arraysize(kForwardSwitches));

  GetContentClient()->browser()->AppendExtraCommandLineSwitches(cmd_line, -1);

  // Start up the sandbox host process and get the file descriptor for the
  // sandboxed processes to talk to it.
  base::FileHandleMappingVector additional_remapped_fds;
  additional_remapped_fds.emplace_back(
      SandboxHostLinux::GetInstance()->GetChildSocket(), GetSandboxFD());

  return ZygoteHostImpl::GetInstance()->LaunchZygote(
      cmd_line, control_fd, std::move(additional_remapped_fds));
}

pid_t ZygoteHostImpl::LaunchZygote(
    base::CommandLine* cmd_line,
    base::ScopedFD* control_fd,
    base::FileHandleMappingVector additional_remapped_fds) {
  ...
  base::LaunchOptions options;
  options.fds_to_remap = std::move(additional_remapped_fds);
  options.fds_to_remap.emplace_back(fds[1], kZygoteSocketPairFd);

  base::ScopedFD dummy_fd;
  if (use_suid_sandbox_) {
    std::unique_ptr<sandbox::SetuidSandboxHost> sandbox_host(
        sandbox::SetuidSandboxHost::Create());
    sandbox_host->PrependWrapper(cmd_line);
    sandbox_host->SetupLaunchOptions(&options, &dummy_fd);
    sandbox_host->SetupLaunchEnvironment();
  }

  base::Process process =
      use_namespace_sandbox_
          ? sandbox::NamespaceSandbox::LaunchProcess(*cmd_line, options)
          : base::LaunchProcess(*cmd_line, options);
  CHECK(process.IsValid()) << "Failed to launch zygote process";
  ...
}

```



child process是如何连上parent process的？

在render process的启动过程中，会创建RenderProcess

```
Breakpoint 2, (anonymous namespace)::RenderProcessImpl::Create () at ../../content/renderer/render_process_impl.cc:205
205	      content::GetContentClient()->renderer()->GetTaskSchedulerInitParams();
(gdb) bt
#0  0x00007ffff2bd984e in (anonymous namespace)::RenderProcessImpl::Create() () at ../../content/renderer/render_process_impl.cc:205
#1  0x00007ffff2c5bf70 in (anonymous namespace)::RendererMain((anonymous namespace)::MainFunctionParams const&) (parameters=...) at ../../content/renderer/renderer_main.cc:234
#2  0x00007ffff3089017 in (anonymous namespace)::RunNamedProcessTypeMain((anonymous namespace)::(anonymous namespace)::string const&, (anonymous namespace)::MainFunctionParams const&, (anonymous namespace)::ContentMainDelegate*) (process_type=..., main_function_params=..., delegate=0x7fffffffd900) at ../../content/app/content_main_runner.cc:427
#3  0x00007ffff308b6f4 in (anonymous namespace)::ContentMainRunnerImpl::Run() (this=0x3e4e29831800) at ../../content/app/content_main_runner.cc:706
#4  0x00007ffff3081bc5 in (anonymous namespace)::ContentServiceManagerMainDelegate::RunEmbedderProcess() (this=0x7fffffffd860)
    at ../../content/app/content_service_manager_main_delegate.cc:51
#5  0x00007ffff7e8fdfc in (anonymous namespace)::Main((anonymous namespace)::MainParams const&) (params=...) at ../../services/service_manager/embedder/main.cc:453
#6  0x00007ffff3087dc5 in (anonymous namespace)::ContentMain((anonymous namespace)::ContentMainParams const&) (params=...) at ../../content/app/content_main.cc:19
#7  0x0000555556cde240 in ChromeMain(int, char const**) (argc=17, argv=0x7fffffffda88) at ../../chrome/app/chrome_main.cc:101
#8  0x0000555556cde152 in main(int, char const**) (argc=17, argv=0x7fffffffda88) at ../../chrome/app/chrome_exe_main_aura.cc:17

```






```
4:050:x86> kbn
 # ChildEBP RetAddr  Args to Child
00 0039f44c 5807a285 04da2940 0039f46c 0039f460 KERNEL32!CreateFileW
01 0039f4a8 58078f46 0039f75c 0000007f 0000001f chrome_elf!crashpad::CrashpadClient::SetHandlerIPCPipe+0x89 [C:\b\c\b\win_clang\src\third_party\crashpad\crashpad\client\crashpad_client_win.cc @ 670] 
02 0039f79c 5807706d 0039f7d8 00000000 00000000 chrome_elf!crash_reporter::internal::PlatformCrashpadInitialization+0x3b4 [C:\b\c\b\win_clang\src\components\crash\content\app\crashpad_win.cc @ 154] 
03 0039f800 5807717c 0039f83c 00000001 0039f894 chrome_elf!crash_reporter::`anonymous namespace'::InitializeCrashpadImpl+0x41 [C:\b\c\b\win_clang\src\components\crash\content\app\crashpad.cc @ 119] 
04 0039f810 5805320b 00000000 0039f824 0039f83c chrome_elf!crash_reporter::InitializeCrashpadWithEmbeddedHandler+0x16 [C:\b\c\b\win_clang\src\components\crash\content\app\crashpad.cc @ 192] 
05 0039f894 58052fca 00a4c878 0039f9fc 00931027 chrome_elf!ChromeCrashReporterClient::InitializeCrashReportingForProcess+0x133 [C:\b\c\b\win_clang\src\chrome\app\chrome_crash_reporter_client_win.cc @ 55] 
06 0039f8a0 00931027 b579bf52 fffffffe 0039f8f0 chrome_elf!elf_crash::InitializeCrashReporting+0x3e [C:\b\c\b\win_clang\src\chrome_elf\crash\crash_helper.cc @ 77] 
07 0039f9fc 009ef2f8 00930000 00000000 006429a0 chrome!wWinMain+0x27 [C:\b\c\b\win_clang\src\chrome\app\chrome_exe_main_win.cc @ 179] 
08 (Inline) -------- -------- -------- -------- chrome!invoke_main+0x1a [f:\dd\vctools\crt\vcstartup\src\startup\exe_common.inl @ 118] 
09 0039fa48 74af62c4 00452000 74af62a0 cf827f32 chrome!__scrt_common_main_seh+0xf6 [f:\dd\vctools\crt\vcstartup\src\startup\exe_common.inl @ 283] 
0a 0039fa5c 77940f79 00452000 c2ddc6ee 00000000 KERNEL32!BaseThreadInitThunk+0x24
0b 0039faa4 77940f44 ffffffff 77962ed0 00000000 ntdll_778e0000!__RtlUserThreadStart+0x2f
0c 0039fab4 00000000 009ef370 00452000 00000000 ntdll_778e0000!_RtlUserThreadStart+0x1b

4:050:x86> dd esp
0039f358  5807e18e 04da2d78 c0000000 00000000

4:050:x86> du 04da2d78 
04da2d78  "\\.\pipe\crashpad_6524_VDPBGAXUF"
04da2db8  "DELOSWG"







Breakpoint 0 hit
chrome!content::RenderProcessHostImpl::OnChannelConnected:
54f7fee0 55              push    ebp
0:000:x86> kbn
 # ChildEBP RetAddr  Args to Child              
00 0515f5cc 55449bc9 00000930 0515f618 56ea66d0 chrome!content::RenderProcessHostImpl::OnChannelConnected [C:\b\c\b\win_clang\src\content\browser\renderer_host\render_process_host_impl.cc @ 2928] 
01 0515f5e4 54731a49 11977c50 00000000 00000000 chrome!IPC::ChannelProxy::Context::OnDispatchConnected+0x33 [C:\b\c\b\win_clang\src\ipc\ipc_channel_proxy.cc @ 343] 
02 (Inline) -------- -------- -------- -------- chrome!base::OnceCallback<void ()>::Run+0x10 [C:\b\c\b\win_clang\src\base\callback.h @ 65] 
03 0515f654 54743133 569a2928 0515f710 0515f6e8 chrome!base::debug::TaskAnnotator::RunTask+0x99 [C:\b\c\b\win_clang\src\base\debug\task_annotator.cc @ 53] 
04 0515f664 54742c26 0515f710 56a15794 07073920 chrome!base::internal::IncomingTaskQueue::RunTask+0x13 [C:\b\c\b\win_clang\src\base\message_loop\incoming_task_queue.cc @ 125] 
05 0515f6e8 54742a47 0515f710 546bb87c f726de21 chrome!base::MessageLoop::RunTask+0x1b6 [C:\b\c\b\win_clang\src\base\message_loop\message_loop.cc @ 399] 
06 0515f708 546c70be 00000000 569100a9 56a15794 chrome!base::MessageLoop::DeferOrRunPendingTask+0x57 [C:\b\c\b\win_clang\src\base\message_loop\message_loop.cc @ 411] 
07 0515f7c0 5478d7ed 07072c58 000b0856 00000401 chrome!base::MessageLoop::DoWork+0xde [C:\b\c\b\win_clang\src\base\message_loop\message_loop.cc @ 455] 
08 0515f7f8 546c6e5e 070738b0 546c6c00 00000001 chrome!base::MessagePumpForUI::DoRunLoop+0x7d [C:\b\c\b\win_clang\src\base\message_loop\message_pump_win.cc @ 174] 
09 0515f828 546c6dcf 070738b0 0515f858 0515f848 chrome!base::MessagePumpWin::Run+0x6e [C:\b\c\b\win_clang\src\base\message_loop\message_pump_win.cc @ 58] 
0a 0515f838 546c688e 00000001 0515f858 0515f894 chrome!base::MessageLoop::Run+0x1f [C:\b\c\b\win_clang\src\base\message_loop\message_loop.cc @ 350] 
0b 0515f848 549e6769 0db95168 ffffffff 070738b4 chrome!base::RunLoop::Run+0x2e [C:\b\c\b\win_clang\src\base\run_loop.cc @ 136] 
0c 0515f894 549e65f3 07070728 e770174d 00000005 chrome!ChromeBrowserMainParts::MainMessageLoopRun+0x8b [C:\b\c\b\win_clang\src\chrome\browser\chrome_browser_main.cc @ 1977] 
0d 0515f8c4 549e65ae 0515f8e8 0515f914 546b3e42 chrome!content::BrowserMainLoop::RunMainMessageLoopParts+0x3b [C:\b\c\b\win_clang\src\content\browser\browser_main_loop.cc @ 1238] 
0e 0515f8d0 546b3e42 00000018 0515f8f8 0515f8f8 chrome!content::BrowserMainRunnerImpl::Run+0xe [C:\b\c\b\win_clang\src\content\browser\browser_main_runner.cc @ 145] 
0f 0515f914 546b3d32 0515f9fc 07070b38 0515f94c chrome!content::BrowserMain+0x9d [C:\b\c\b\win_clang\src\content\browser\browser_main.cc @ 46] 
10 0515f9e4 546b3c0c 0515fa18 0515f9fc 0515fbc8 chrome!content::RunNamedProcessTypeMain+0xee [C:\b\c\b\win_clang\src\content\app\content_main_runner.cc @ 426] 
11 0515fa40 546a50b6 fffffffe 00000003 0515fb5c chrome!content::ContentMainRunnerImpl::Run+0x9a [C:\b\c\b\win_clang\src\content\app\content_main_runner.cc @ 717] 
12 0515fb4c 546a4d87 0515fb58 0515fb5c 56976f00 chrome!service_manager::Main+0x26e [C:\b\c\b\win_clang\src\services\service_manager\embedder\main.cc @ 456] 
13 0515fb90 546a1b05 0515fbac 0515fb9c 0515fb98 chrome!content::ContentMain+0x33 [C:\b\c\b\win_clang\src\content\app\content_main.cc @ 19] 
14 0515fbf8 00192f32 00190000 0515fc40 dfde7890 chrome!ChromeMain+0x113 [C:\b\c\b\win_clang\src\chrome\app\chrome_main.cc @ 132] 
15 0515fc84 00191467 00190000 dfde7890 00000005 chrome_exe!MainDllLoader::Launch+0x230 [C:\b\c\b\win_clang\src\chrome\app\main_dll_loader_win.cc @ 199] 
16 0515fdf0 0024f2f8 00190000 00000000 05262926 chrome_exe!wWinMain+0x467 [C:\b\c\b\win_clang\src\chrome\app\chrome_exe_main_win.cc @ 231] 
17 (Inline) -------- -------- -------- -------- chrome_exe!invoke_main+0x1a [f:\dd\vctools\crt\vcstartup\src\startup\exe_common.inl @ 118] 
18 0515fe3c 75ee62c4 04e15000 75ee62a0 973d263c chrome_exe!__scrt_common_main_seh+0xf6 [f:\dd\vctools\crt\vcstartup\src\startup\exe_common.inl @ 283] 
19 0515fe50 773f0f79 04e15000 0f602595 00000000 KERNEL32!BaseThreadInitThunk+0x24
1a 0515fe98 773f0f44 ffffffff 77412eda 00000000 ntdll_77390000!__RtlUserThreadStart+0x2f
1b 0515fea8 00000000 0024f370 04e15000 00000000 ntdll_77390000!_RtlUserThreadStart+0x1b



```