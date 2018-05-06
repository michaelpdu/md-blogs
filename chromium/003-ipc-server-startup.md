# IPC通信的连接建立过程

## Table of Contents

1. [Windows环境下的整体描述](#summary)
2. [各种场景下的详细分析](#scenarios)
3. [Linux环境下的差异](#linux)

### Summary

1. browser process会对每一个render/gpu process创建对应的Mojo IPC，并将channel handler转成Int作为mojo-platform-channel-handle参数，传给子进程。子进程会在各自对应的Main函数（GpuMain/RenderMain）中初始化相应的线程（GpuThreadImpl|RenderThreadImpl）时，通过Mojo EDK的accept函数连接上对应的Mojo IPC server。

在分析Mojo IPC channel的建立过程之前，可以先读一下Mojo IPC EDK document，这样，可以了解Mojo IPC的基本使用。

[https://chromium.googlesource.com/chromium/src/+/master/mojo/edk/embedder](https://chromium.googlesource.com/chromium/src/+/master/mojo/edk/embedder)

2. crashpad process会创建一个named pipe(CreateNamedPipe)，后续的process会通过传统的IPC channel（CreateFile）连上这个named pipe。

```sh
# breakpoints for IPC

bu kernel32!CreateNamedPipeWStub "du poi(esp+4)"
bu kernel32!CreateFileW "du poi(esp+4)"
bu kernel32!CallNamedPipeW "du poi(esp+4)"

# breakpoints for mojo IPC

bu chrome!content::internal::ChildProcessLauncherHelper::LaunchProcessOnLauncherThread
bu chrome!mojo::edk::OutgoingBrokerClientInvitation::Send

bu chrome_child!mojo::edk::IncomingBrokerClientInvitation::Accept
bu chrome_child!mojo::edk::PlatformChannelPair::PassClientHandleFromParentProcess
bu chrome_child!mojo::edk::NamedPlatformChannelPair::PassClientHandleFromParentProcess
```

### Scenarios

**browser process与crashpad handler的通信管道是如何建立的？**

browser process与crashpad handler之间没有IPC通信，之所以在browser process中创建，应该是为了提前将验证工作做了。`是不是还有其他什么原因，可以进一步分析。`

browser process创建named pipe的过程，在StartHandler中，会先创建一个named pipe。

```cpp
//"\\.\pipe\crashpad_2540_BNANIIMRFKUXEEYY"
00 KERNEL32!CreateNamedPipeWStub
01 chrome_elf!crashpad::CreateNamedPipeInstance+0x6d [C:\b\c\b\win_clang\src\third_party\crashpad\crashpad\util\win\registration_protocol_win.cc @ 116] 
02 chrome_elf!crashpad::`anonymous namespace'::CreatePipe+0x127
03 chrome_elf!crashpad::CrashpadClient::StartHandler+0x148 [C:\b\c\b\win_clang\src\third_party\crashpad\crashpad\client\crashpad_client_win.cc @ 587] 
04 chrome_elf!crash_reporter::internal::PlatformCrashpadInitialization+0x5d2 [C:\b\c\b\win_clang\src\components\crash\content\app\crashpad_win.cc @ 149] 
05 chrome_elf!crash_reporter::`anonymous namespace'::InitializeCrashpadImpl+0x40 [C:\b\c\b\win_clang\src\components\crash\content\app\crashpad.cc @ 128] 
06 chrome_elf!crash_reporter::InitializeCrashpadWithEmbeddedHandler+0x16 [C:\b\c\b\win_clang\src\components\crash\content\app\crashpad.cc @ 203] 
07 chrome_elf!ChromeCrashReporterClient::InitializeCrashReportingForProcess+0x133 [C:\b\c\b\win_clang\src\chrome\app\chrome_crash_reporter_client_win.cc @ 55] 
08 chrome_elf!elf_crash::InitializeCrashReporting+0x3e [C:\b\c\b\win_clang\src\chrome_elf\crash\crash_helper.cc @ 77] 
09 chrome!wWinMain+0x27 [C:\b\c\b\win_clang\src\chrome\app\chrome_exe_main_win.cc @ 179] 
```

然后，会StartHandlerProcess，并在最后做一个测试(SendToCrashHandlerServer)，会判断named pipe server是否可以正常工作。

```cpp
00 KERNEL32!CreateFileW
// SendToCrashHandlerServer
01 chrome_elf!crashpad::`anonymous namespace'::StartHandlerProcess+0x7d6 [C:\b\c\b\win_clang\src\third_party\crashpad\crashpad\client\crashpad_client_win.cc @ 512] 
02 chrome_elf!crashpad::CrashpadClient::StartHandler+0x330 [C:\b\c\b\win_clang\src\third_party\crashpad\crashpad\client\crashpad_client_win.cc @ 635] 
03 chrome_elf!crash_reporter::internal::PlatformCrashpadInitialization+0x5d2 [C:\b\c\b\win_clang\src\components\crash\content\app\crashpad_win.cc @ 149] 
04 chrome_elf!crash_reporter::`anonymous namespace'::InitializeCrashpadImpl+0x40 [C:\b\c\b\win_clang\src\components\crash\content\app\crashpad.cc @ 128] 
05 chrome_elf!crash_reporter::InitializeCrashpadWithEmbeddedHandler+0x16 [C:\b\c\b\win_clang\src\components\crash\content\app\crashpad.cc @ 203] 
06 chrome_elf!ChromeCrashReporterClient::InitializeCrashReportingForProcess+0x133 [C:\b\c\b\win_clang\src\chrome\app\chrome_crash_reporter_client_win.cc @ 55] 
07 chrome_elf!elf_crash::InitializeCrashReporting+0x3e [C:\b\c\b\win_clang\src\chrome_elf\crash\crash_helper.cc @ 77] 
08 chrome!wWinMain+0x27 [C:\b\c\b\win_clang\src\chrome\app\chrome_exe_main_win.cc @ 179] 
```

在crashpad handler process中，crashpad handler process会使用相同的named pipe重新创建一个named pipe instance。后面的Gpu/Render Process就会连接到这个named pipe上。

```cpp
// "\\.\pipe\crashpad_2540_BNANIIMRFKUXEEYY"
00 KERNEL32!CreateNamedPipeWStub
01 chrome!crashpad::CreateNamedPipeInstance+0x6d [C:\b\c\b\win_clang\src\third_party\crashpad\crashpad\util\win\registration_protocol_win.cc @ 116] 
02 chrome!crashpad::ExceptionHandlerServer::Run+0xbb [C:\b\c\b\win_clang\src\third_party\crashpad\crashpad\util\win\exception_handler_server.cc @ 315] 
03 chrome!crashpad::HandlerMain+0xb0d [C:\b\c\b\win_clang\src\third_party\crashpad\crashpad\handler\handler_main.cc @ 799] 
04 chrome!crash_reporter::RunAsCrashpadHandler+0x639 [C:\b\c\b\win_clang\src\components\crash\content\app\run_as_crashpad_handler_win.cc @ 88] 
05 chrome!wWinMain+0xb4 [C:\b\c\b\win_clang\src\chrome\app\chrome_exe_main_win.cc @ 205] 
```

**browser process与watcher process的通信管道是如何建立的？**

browser process与watcher之间没有IPC

**browser process与render process的通信管道是如何建立的？**

***browser process启动Mojo Pipe server的过程***

browser process为render process创建的mojo ipc: "\\.\pipe\mojo.2540.3796.7531368083646228087"，ipc name是随机生成的。

```
00 KERNEL32!CreateNamedPipeWStub
01 chrome!mojo::edk::PlatformChannelPair::PlatformChannelPair+0xa4 [C:\b\c\b\win_clang\src\mojo\edk\embedder\platform_channel_pair_win.cc @ 37] 
02 chrome!content::internal::ChildProcessLauncherHelper::StartLaunchOnClientThread+0x81 [C:\b\c\b\win_clang\src\content\browser\child_process_launcher_helper.cc @ 83] 
03 chrome!content::ChildProcessLauncher::ChildProcessLauncher+0xf4 [C:\b\c\b\win_clang\src\content\browser\child_process_launcher.cc @ 50] 
04 chrome!content::RenderProcessHostImpl::Init+0x327 [C:\b\c\b\win_clang\src\content\browser\renderer_host\render_process_host_impl.cc @ 1560] 
05 chrome!content::RenderFrameHostManager::CreateSpeculativeRenderFrameHost+0x39 [C:\b\c\b\win_clang\src\content\browser\frame_host\render_frame_host_manager.cc @ 1608] 
06 chrome!content::RenderFrameHostManager::GetFrameHostForNavigation+0x94 [C:\b\c\b\win_clang\src\content\browser\frame_host\render_frame_host_manager.cc @ 570] 
07 chrome!content::RenderFrameHostManager::DidCreateNavigationRequest+0x10 [C:\b\c\b\win_clang\src\content\browser\frame_host\render_frame_host_manager.cc @ 471] 
08 chrome!content::FrameTreeNode::CreatedNavigationRequest+0x7e [C:\b\c\b\win_clang\src\content\browser\frame_host\frame_tree_node.cc @ 494] 
09 chrome!content::NavigatorImpl::RequestNavigation+0x1b1 [C:\b\c\b\win_clang\src\content\browser\frame_host\navigator_impl.cc @ 1071] 
0a chrome!content::NavigatorImpl::NavigateToEntry+0x413 [C:\b\c\b\win_clang\src\content\browser\frame_host\navigator_impl.cc @ 338] 
0b chrome!content::NavigatorImpl::NavigateToPendingEntry+0x6b [C:\b\c\b\win_clang\src\content\browser\frame_host\navigator_impl.cc @ 387] 
0c chrome!content::NavigationControllerImpl::NavigateToPendingEntryInternal+0x128 [C:\b\c\b\win_clang\src\content\browser\frame_host\navigation_controller_impl.cc @ 2157] 
0d chrome!content::NavigationControllerImpl::NavigateToPendingEntry+0x126 [C:\b\c\b\win_clang\src\content\browser\frame_host\navigation_controller_impl.cc @ 2111] 
0e chrome!content::NavigationControllerImpl::LoadEntry+0x4e [C:\b\c\b\win_clang\src\content\browser\frame_host\navigation_controller_impl.cc @ 512] 
0f chrome!content::NavigationControllerImpl::LoadURLWithParams+0x40e [C:\b\c\b\win_clang\src\content\browser\frame_host\navigation_controller_impl.cc @ 870] 
10 chrome!`anonymous namespace'::LoadURLInContents+0x1fc [C:\b\c\b\win_clang\src\chrome\browser\ui\browser_navigator.cc @ 321] 
11 chrome!Navigate+0x7dc [C:\b\c\b\win_clang\src\chrome\browser\ui\browser_navigator.cc @ 612] 
12 chrome!StartupBrowserCreatorImpl::OpenTabsInBrowser+0x1ae [C:\b\c\b\win_clang\src\chrome\browser\ui\startup\startup_browser_creator_impl.cc @ 539] 
13 chrome!StartupBrowserCreatorImpl::RestoreOrCreateBrowser+0x138 [C:\b\c\b\win_clang\src\chrome\browser\ui\startup\startup_browser_creator_impl.cc @ 861] 
14 chrome!StartupBrowserCreatorImpl::DetermineURLsAndLaunch+0x1be [C:\b\c\b\win_clang\src\chrome\browser\ui\startup\startup_browser_creator_impl.cc @ 741] 
15 chrome!StartupBrowserCreatorImpl::Launch+0x25c [C:\b\c\b\win_clang\src\chrome\browser\ui\startup\startup_browser_creator_impl.cc @ 436] 
16 chrome!StartupBrowserCreator::LaunchBrowser+0x124 [C:\b\c\b\win_clang\src\chrome\browser\ui\startup\startup_browser_creator.cc @ 358] 
17 chrome!StartupBrowserCreator::LaunchBrowserForLastProfiles+0x6f [C:\b\c\b\win_clang\src\chrome\browser\ui\startup\startup_browser_creator.cc @ 760] 
18 chrome!StartupBrowserCreator::ProcessCmdLineImpl+0x3e3 [C:\b\c\b\win_clang\src\chrome\browser\ui\startup\startup_browser_creator.cc @ 729] 
19 chrome!StartupBrowserCreator::Start+0x53 [C:\b\c\b\win_clang\src\chrome\browser\ui\startup\startup_browser_creator.cc @ 313] 
1a chrome!ChromeBrowserMainParts::PreMainMessageLoopRunImpl+0xbfc [C:\b\c\b\win_clang\src\chrome\browser\chrome_browser_main.cc @ 2074] 
1b chrome!ChromeBrowserMainParts::PreMainMessageLoopRun+0x91 [C:\b\c\b\win_clang\src\chrome\browser\chrome_browser_main.cc @ 1456] 
1c chrome!content::BrowserMainLoop::PreMainMessageLoopRun+0x44 [C:\b\c\b\win_clang\src\content\browser\browser_main_loop.cc @ 1086] 
1d chrome!base::RepeatingCallback<int ()>::Run+0x7 [C:\b\c\b\win_clang\src\base\callback.h @ 124] 
1e chrome!content::StartupTaskRunner::RunAllTasksNow+0x17 [C:\b\c\b\win_clang\src\content\browser\startup_task_runner.cc @ 45] 
1f chrome!content::BrowserMainLoop::CreateStartupTasks+0x229 [C:\b\c\b\win_clang\src\content\browser\browser_main_loop.cc @ 971] 
20 chrome!content::BrowserMainRunnerImpl::Initialize+0x64 [C:\b\c\b\win_clang\src\content\browser\browser_main_runner.cc @ 140] 
21 chrome!content::BrowserMain+0x8a [C:\b\c\b\win_clang\src\content\browser\browser_main.cc @ 42] 
22 chrome!content::RunNamedProcessTypeMain+0xee [C:\b\c\b\win_clang\src\content\app\content_main_runner.cc @ 423] 
23 chrome!content::ContentMainRunnerImpl::Run+0x8e [C:\b\c\b\win_clang\src\content\app\content_main_runner.cc @ 703] 
24 chrome!service_manager::Main+0x26e [C:\b\c\b\win_clang\src\services\service_manager\embedder\main.cc @ 453] 
25 chrome!content::ContentMain+0x33 [C:\b\c\b\win_clang\src\content\app\content_main.cc @ 19] 
26 chrome!ChromeMain+0x108 [C:\b\c\b\win_clang\src\chrome\app\chrome_main.cc @ 104] 
27 chrome_exe!MainDllLoader::Launch+0x230 [C:\b\c\b\win_clang\src\chrome\app\main_dll_loader_win.cc @ 199] 
28 chrome_exe!wWinMain+0x45d [C:\b\c\b\win_clang\src\chrome\app\chrome_exe_main_win.cc @ 231] 
```

render process启动，然后，browser process的IO thread会再次创建新的named pipe: "\\.\pipe\mojo.2540.4984.16495538753415545208"

```
00 KERNEL32!CreateNamedPipeWStub
01 chrome!mojo::edk::PlatformChannelPair::PlatformChannelPair+0xa4 [C:\b\c\b\win_clang\src\mojo\edk\embedder\platform_channel_pair_win.cc @ 37] 
02 chrome!mojo::edk::NodeController::SendBrokerClientInvitationOnIOThread+0x30 [C:\b\c\b\win_clang\src\mojo\edk\system\node_controller.cc @ 343] 

00 chrome!mojo::edk::NodeController::SendBrokerClientInvitation [C:\b\c\b\win_clang\src\mojo\edk\system\node_controller.cc @ 174] 
01 chrome!mojo::edk::Core::SendBrokerClientInvitation+0x68 [C:\b\c\b\win_clang\src\mojo\edk\system\core.cc @ 212] 
02 chrome!mojo::edk::OutgoingBrokerClientInvitation::Send+0x51 [C:\b\c\b\win_clang\src\mojo\edk\embedder\outgoing_broker_client_invitation.cc @ 62] 
03 chrome!content::internal::ChildProcessLauncherHelper::PostLaunchOnLauncherThread+0xfc [C:\b\c\b\win_clang\src\content\browser\child_process_launcher_helper.cc @ 139] 
04 chrome!content::internal::ChildProcessLauncherHelper::LaunchOnLauncherThread+0xf0 [C:\b\c\b\win_clang\src\content\browser\child_process_launcher_helper.cc @ 115] 
```

为什么会再次创建named pipe？可以看出，两次CreateNamedPipe都是实例化PlatformChannelPair而调用的。

第一次实例化PlatformChannelPair，是创建channel。第二次实例化PlatformChannelPair则是在PostLaunchOnLauncherThread中send invitation触发的。

`关于Mojo IPC的实现原理，后面单独分析。` 下面是官方网站给出的demo code：

```cpp
base::ProcessHandle LaunchCoolChildProcess(
    mojo::edk::ScopedPlatformHandle channel);

int main(int argc, char** argv) {
  mojo::edk::Init();

  base::Thread ipc_thread("ipc!");
  ipc_thread.StartWithOptions(
      base::Thread::Options(base::MessageLoop::TYPE_IO, 0));

  mojo::edk::ScopedIPCSupport ipc_support(
      ipc_thread.task_runner(),
      mojo::edk::ScopedIPCSupport::ShutdownPolicy::CLEAN);

  // This is essentially always an OS pipe (domain socket pair, Windows named
  // pipe, etc.)
  mojo::edk::PlatformChannelPair channel;                         // first time

  // This is a scoper which encapsulates the intent to connect to another
  // process. It exists because process connection is inherently asynchronous,
  // things may go wrong, and the lifetime of any associated resources is bound
  // by the lifetime of this object regardless of success or failure.
  mojo::edk::OutgoingBrokerClientInvitation invitation;

  base::ProcessHandle child_handle =
      LaunchCoolChildProcess(channel.PassClientHandle());

  // At this point it's safe for |invitation| to go out of scope and nothing
  // will break.
  invitation.Send(child_handle, channel.PassServerHandle());      // second time

  return 0;
}
```

`这里有一个疑问？`

browser process的逻辑应该是下面这样：

LaunchProcessOnLauncherThread --> child process启动 --> OutgoingBrokerClientInvitation::Send

但是，实际调试过程中，发现send invitation，并不会等待child process的启动，需要进一步分析原因。

***render process连接Mojo Pipe server的过程***

ChildThreadImp在初始化的时候，会去初始化MojoIPCChannel，在此过程中，会根据mojo-platform-channel-handle的值连上对应的mojo ipc server。

```cpp
00 chrome_child!mojo::edk::IncomingBrokerClientInvitation::Accept [C:\b\c\b\win_clang\src\mojo\edk\embedder\incoming_broker_client_invitation.cc @ 22] 
01 chrome_child!content::`anonymous namespace'::InitializeMojoIPCChannel+0x15e [C:\b\c\b\win_clang\src\content\child\child_thread_impl.cc @ 263] 
02 chrome_child!content::ChildThreadImpl::Init+0x1d3 [C:\b\c\b\win_clang\src\content\child\child_thread_impl.cc @ 458] 
03 chrome_child!content::ChildThreadImpl::ChildThreadImpl+0x177 [C:\b\c\b\win_clang\src\content\child\child_thread_impl.cc @ 392] 
04 chrome_child!content::RenderThreadImpl::RenderThreadImpl+0xa0 [C:\b\c\b\win_clang\src\content\renderer\render_thread_impl.cc @ 760] 
05 chrome_child!content::RenderThreadImpl::Create+0x72 [C:\b\c\b\win_clang\src\content\renderer\render_thread_impl.cc @ 677] 
06 chrome_child!content::RendererMain+0x323 [C:\b\c\b\win_clang\src\content\renderer\renderer_main.cc @ 215] 
07 chrome_child!content::RunNamedProcessTypeMain+0x10c [C:\b\c\b\win_clang\src\content\app\content_main_runner.cc @ 423] 
08 chrome_child!content::ContentMainRunnerImpl::Run+0x8e [C:\b\c\b\win_clang\src\content\app\content_main_runner.cc @ 703] 
09 chrome_child!service_manager::Main+0x26e [C:\b\c\b\win_clang\src\services\service_manager\embedder\main.cc @ 453] 
0a chrome_child!content::ContentMain+0x33 [C:\b\c\b\win_clang\src\content\app\content_main.cc @ 19] 
0b chrome_child!ChromeMain+0x108 [C:\b\c\b\win_clang\src\chrome\app\chrome_main.cc @ 104] 
0c chrome!MainDllLoader::Launch+0x230 [C:\b\c\b\win_clang\src\chrome\app\main_dll_loader_win.cc @ 199] 
0d chrome!wWinMain+0x45d [C:\b\c\b\win_clang\src\chrome\app\chrome_exe_main_win.cc @ 231] 
```

在InitializeMojoIPCChannel中，会先获取平台独立的platform_channel，然后，再接受来自这个channel的invitation。

```cpp
std::unique_ptr<mojo::edk::IncomingBrokerClientInvitation>
InitializeMojoIPCChannel() {
  TRACE_EVENT0("startup", "InitializeMojoIPCChannel");
  mojo::edk::ScopedPlatformHandle platform_channel;
#if defined(OS_WIN)
  if (base::CommandLine::ForCurrentProcess()->HasSwitch(
      mojo::edk::PlatformChannelPair::kMojoPlatformChannelHandleSwitch)) {
    platform_channel =
        mojo::edk::PlatformChannelPair::PassClientHandleFromParentProcess(
            *base::CommandLine::ForCurrentProcess());
  } else {
    // If this process is elevated, it will have a pipe path passed on the
    // command line.
    platform_channel =
        mojo::edk::NamedPlatformChannelPair::PassClientHandleFromParentProcess(
            *base::CommandLine::ForCurrentProcess());
  }
#elif defined(OS_FUCHSIA)
  platform_channel =
      mojo::edk::PlatformChannelPair::PassClientHandleFromParentProcess(
          *base::CommandLine::ForCurrentProcess());
#elif defined(OS_POSIX)
  platform_channel.reset(mojo::edk::PlatformHandle(
      base::GlobalDescriptors::GetInstance()->Get(kMojoIPCChannel)));
#endif
  // Mojo isn't supported on all child process types.
  // TODO(crbug.com/604282): Support Mojo in the remaining processes.
  if (!platform_channel.is_valid())
    return nullptr;

  return mojo::edk::IncomingBrokerClientInvitation::Accept(
      mojo::edk::ConnectionParams(mojo::edk::TransportProtocol::kLegacy,
                                  std::move(platform_channel)));
}

```

RenderThread是一种ChildThread，在构造RenderThread的时候，首先需要创建ChildThread。

在ChildThreadImpl::Init函数中，
  - IPC::SyncChannel::Create，创建sync channel (`做什么用的？看起来是用于profile的`)
  - InitializeMojoIPCChannel， 初始化Mojo IPC channel
  - extract service pipe from command line and ServiceManagerConnection::Create, and then, AddConnectionFilter to this connection
  - InitTracing, memory_instrumentation::ClientProcessImpl::CreateInstance, PowerMonitor
  - Add filters to sync channel and ConnectChannel (`这里sync channel 与 Mojo IPC之间的互动是做什么用的？`)
  - StartServiceManagerConnection if auto_start_service_manager_connection is true，否则不会立即调用，而是在render thread的init函数中调用
  - message loop, 定时查询，确保channel处于连接状态，一旦不再连接，立即终止当前进程

```
00 chrome_child!content::ChildThreadImpl::StartServiceManagerConnection [C:\b\c\b\win_clang\src\content\child\child_thread_impl.cc @ 746] 
01 chrome_child!content::RenderThreadImpl::Init+0xd87 [C:\b\c\b\win_clang\src\content\renderer\render_thread_impl.cc @ 823] 
02 chrome_child!content::RenderThreadImpl::RenderThreadImpl+0x4ac [C:\b\c\b\win_clang\src\content\renderer\render_thread_impl.cc @ 662] 
03 chrome_child!content::RenderThreadImpl::Create+0x72 [C:\b\c\b\win_clang\src\content\renderer\render_thread_impl.cc @ 574] 
04 chrome_child!content::RendererMain+0x1a1 [C:\b\c\b\win_clang\src\content\renderer\renderer_main.cc @ 198] 
05 chrome_child!content::RunNamedProcessTypeMain+0x10c [C:\b\c\b\win_clang\src\content\app\content_main_runner.cc @ 426] 
06 chrome_child!content::ContentMainRunnerImpl::Run+0x9a [C:\b\c\b\win_clang\src\content\app\content_main_runner.cc @ 717] 
07 chrome_child!service_manager::Main+0x26e [C:\b\c\b\win_clang\src\services\service_manager\embedder\main.cc @ 456] 
08 chrome_child!content::ContentMain+0x33 [C:\b\c\b\win_clang\src\content\app\content_main.cc @ 19] 
09 chrome_child!ChromeMain+0x113 [C:\b\c\b\win_clang\src\chrome\app\chrome_main.cc @ 132] 
0a chrome!MainDllLoader::Launch+0x230 [C:\b\c\b\win_clang\src\chrome\app\main_dll_loader_win.cc @ 199] 
0b chrome!wWinMain+0x467 [C:\b\c\b\win_clang\src\chrome\app\chrome_exe_main_win.cc @ 231] 
```

***render process连接到crashpad handler的named pipe上的过程***

在Main函数中会初始化crash reporting，内部会根据传入的参数，设置IPC Pipe，进而连接上pipe server。

```
"\\.\pipe\crashpad_2540_BNANIIMRFKUXEEYY"

00 KERNEL32!CreateFileW
01 chrome_elf!crashpad::CrashpadClient::SetHandlerIPCPipe+0x89 [C:\b\c\b\win_clang\src\third_party\crashpad\crashpad\client\crashpad_client_win.cc @ 670] 
02 chrome_elf!crash_reporter::internal::PlatformCrashpadInitialization+0x3a3 [C:\b\c\b\win_clang\src\components\crash\content\app\crashpad_win.cc @ 154] 
03 chrome_elf!crash_reporter::`anonymous namespace'::InitializeCrashpadImpl+0x40 [C:\b\c\b\win_clang\src\components\crash\content\app\crashpad.cc @ 128] 
04 chrome_elf!crash_reporter::InitializeCrashpadWithEmbeddedHandler+0x16 [C:\b\c\b\win_clang\src\components\crash\content\app\crashpad.cc @ 203] 
05 chrome_elf!ChromeCrashReporterClient::InitializeCrashReportingForProcess+0x133 [C:\b\c\b\win_clang\src\chrome\app\chrome_crash_reporter_client_win.cc @ 55] 
06 chrome_elf!elf_crash::InitializeCrashReporting+0x3e [C:\b\c\b\win_clang\src\chrome_elf\crash\crash_helper.cc @ 77] 
07 chrome!wWinMain+0x27 [C:\b\c\b\win_clang\src\chrome\app\chrome_exe_main_win.cc @ 179]
```

**gpu process与browser process的通信管道是如何建立的？**

GPU process的逻辑基本上与render process一致，差异是Main函数入口不同，以及对应的ChildThread，一个是GpuChildThread，一个是RenderThreadImpl。

```
00 chrome_child!mojo::edk::IncomingBrokerClientInvitation::Accept [C:\b\c\b\win_clang\src\mojo\edk\embedder\incoming_broker_client_invitation.cc @ 22] 
01 chrome_child!content::`anonymous namespace'::InitializeMojoIPCChannel+0x15e [C:\b\c\b\win_clang\src\content\child\child_thread_impl.cc @ 263] 
02 chrome_child!content::ChildThreadImpl::Init+0x1e4 [C:\b\c\b\win_clang\src\content\child\child_thread_impl.cc @ 457] 
03 chrome_child!content::ChildThreadImpl::ChildThreadImpl+0x177 [C:\b\c\b\win_clang\src\content\child\child_thread_impl.cc @ 391] 
04 chrome_child!content::GpuChildThread::GpuChildThread+0x3b [C:\b\c\b\win_clang\src\content\gpu\gpu_child_thread.cc @ 171] 
05 chrome_child!content::GpuChildThread::GpuChildThread+0x7d [C:\b\c\b\win_clang\src\content\gpu\gpu_child_thread.cc @ 152] 
06 chrome_child!content::GpuMain+0x28a [C:\b\c\b\win_clang\src\content\gpu\gpu_main.cc @ 310] 
07 chrome_child!content::RunNamedProcessTypeMain+0x10c [C:\b\c\b\win_clang\src\content\app\content_main_runner.cc @ 426] 
08 chrome_child!content::ContentMainRunnerImpl::Run+0x9a [C:\b\c\b\win_clang\src\content\app\content_main_runner.cc @ 717] 
09 chrome_child!service_manager::Main+0x26e [C:\b\c\b\win_clang\src\services\service_manager\embedder\main.cc @ 456] 
0a chrome_child!content::ContentMain+0x33 [C:\b\c\b\win_clang\src\content\app\content_main.cc @ 19] 
0b chrome_child!ChromeMain+0x113 [C:\b\c\b\win_clang\src\chrome\app\chrome_main.cc @ 132] 
0c chrome!MainDllLoader::Launch+0x230 [C:\b\c\b\win_clang\src\chrome\app\main_dll_loader_win.cc @ 199] 
0d chrome!wWinMain+0x467 [C:\b\c\b\win_clang\src\chrome\app\chrome_exe_main_win.cc @ 231] 
```

在browser process的Chrome_IOThread中，创建与GPU process的通信管道，会执行下面两次CreateNamedPipe，原因如同render process。

```cpp
content::BrowserGpuChannelHostFactory::EstablishRequest::EstablishOnIO
  --> content::GpuProcessHost::Get
    --> content::GpuProcessHost::Init
      --> content::GpuProcessHost::LaunchGpuProcess
        --> content::BrowserChildProcessHostImpl::Launch
          --> content::ChildProcessLauncher::ChildProcessLauncher
            --> content::internal::ChildProcessLauncherHelper::StartLaunchOnClientThread
              --> mojo::edk::PlatformChannelPair::PlatformChannelPair
                --> CreateNamedPipeWStub    // "\\.\pipe\mojo.2540.4984.14806064607191076756"

mojo::edk::NodeController::SendBrokerClientInvitationOnIOThread
--> mojo::edk::PlatformChannelPair::PlatformChannelPair
  --> CreateNamedPipeWStub    // "\\.\pipe\mojo.2540.4984.10513371942978119032"
```

***crashpad handler process与gpu process的通信管道***

GPU Process连接crashpad handler的过程，与render process一致：

```cpp
wWinMain
--> elf_crash::InitializeCrashReporting
  --> ChromeCrashReporterClient::InitializeCrashReportingForProcess
    --> crash_reporter::InitializeCrashpadWithEmbeddedHandler
      --> crash_reporter::`anonymous namespace'::InitializeCrashpadImpl
        --> crash_reporter::internal::PlatformCrashpadInitialization
          --> crashpad::CrashpadClient::SetHandlerIPCPipe
            --> CreateFileW     // "\\.\pipe\crashpad_2540_BNANIIMRFKUXEEYY"
```

**browser process与utility process的通信管道是如何建立的？**

在browser process的IO thread上，启动Utility Process的时候，会创建Mojo Channel。

"\\.\pipe\mojo.2540.4984.8285217466325777353"

```
00 KERNEL32!CreateNamedPipeWStub
01 chrome!mojo::edk::PlatformChannelPair::PlatformChannelPair+0xa4 [C:\b\c\b\win_clang\src\mojo\edk\embedder\platform_channel_pair_win.cc @ 37] 
02 chrome!content::internal::ChildProcessLauncherHelper::StartLaunchOnClientThread+0x81 [C:\b\c\b\win_clang\src\content\browser\child_process_launcher_helper.cc @ 83] 
03 chrome!content::ChildProcessLauncher::ChildProcessLauncher+0xf4 [C:\b\c\b\win_clang\src\content\browser\child_process_launcher.cc @ 50] 
04 chrome!content::BrowserChildProcessHostImpl::Launch+0x175 [C:\b\c\b\win_clang\src\content\browser\browser_child_process_host_impl.cc @ 243] 
05 chrome!content::UtilityProcessHostImpl::StartProcess+0x33d [C:\b\c\b\win_clang\src\content\browser\utility_process_host_impl.cc @ 344] 
06 chrome!content::`anonymous namespace'::StartServiceInUtilityProcess+0x11f [C:\b\c\b\win_clang\src\content\browser\service_manager\service_manager_context.cc @ 143] 
07 chrome!base::internal::FunctorTraits<void (*)(const std::basic_string<char,std::char_traits<char>,std::allocator<char> > &, const std::basic_string<wchar_t,std::char_traits<wchar_t>,std::allocator<wchar_t> > &, base::Optional<std::basic_string<char,std::char_traits<char>,std::allocator<char> > >, mojo::InterfaceRequest<service_manager::mojom::Service>, mojo::InterfacePtr<service_manager::mojom::PIDReceiver>, service_manager::mojom::ConnectResult, const std::basic_string<char,std::char_traits<char>,std::allocator<char> > &),void>::Invoke<std::basic_string<char,std::char_traits<char>,std::allocator<char> >,std::basic_string<wchar_t,std::char_traits<wchar_t>,std::allocator<wchar_t> >,base::Optional<std::basic_string<char,std::char_traits<char>,std::allocator<char> > >,mojo::InterfaceRequest<service_manager::mojom::Service>,mojo::InterfacePtr<service_manager::mojom::PIDReceiver>,service_manager::mojom::ConnectResult,const std::basic_string<char,std::char_traits<char>,std::allocator<char> > &>+0xc0 [C:\b\c\b\win_clang\src\base\bind_internal.h @ 402] 
08 chrome!base::internal::InvokeHelper<0,void>::MakeItSo+0x19 [C:\b\c\b\win_clang\src\base\bind_internal.h @ 530] 
09 chrome!base::internal::Invoker<base::internal::BindState<void (*)(const std::basic_string<char,std::char_traits<char>,std::allocator<char> > &, const std::basic_string<wchar_t,std::char_traits<wchar_t>,std::allocator<wchar_t> > &, base::Optional<std::basic_string<char,std::char_traits<char>,std::allocator<char> > >, mojo::InterfaceRequest<service_manager::mojom::Service>, mojo::InterfacePtr<service_manager::mojom::PIDReceiver>, service_manager::mojom::ConnectResult, const std::basic_string<char,std::char_traits<char>,std::allocator<char> > &),std::basic_string<char,std::char_traits<char>,std::allocator<char> >,std::basic_string<wchar_t,std::char_traits<wchar_t>,std::allocator<wchar_t> >,base::Optional<std::basic_string<char,std::char_traits<char>,std::allocator<char> > >,mojo::InterfaceRequest<service_manager::mojom::Service>,mojo::InterfacePtr<service_manager::mojom::PIDReceiver> >,void (service_manager::mojom::ConnectResult, const std::basic_string<char,std::char_traits<char>,std::allocator<char> > &)>::RunImpl+0x2b [C:\b\c\b\win_clang\src\base\bind_internal.h @ 604] 
0a chrome!base::internal::Invoker<base::internal::BindState<void (*)(const std::basic_string<char,std::char_traits<char>,std::allocator<char> > &, const std::basic_string<wchar_t,std::char_traits<wchar_t>,std::allocator<wchar_t> > &, base::Optional<std::basic_string<char,std::char_traits<char>,std::allocator<char> > >, mojo::InterfaceRequest<service_manager::mojom::Service>, mojo::InterfacePtr<service_manager::mojom::PIDReceiver>, service_manager::mojom::ConnectResult, const std::basic_string<char,std::char_traits<char>,std::allocator<char> > &),std::basic_string<char,std::char_traits<char>,std::allocator<char> >,std::basic_string<wchar_t,std::char_traits<wchar_t>,std::allocator<wchar_t> >,base::Optional<std::basic_string<char,std::char_traits<char>,std::allocator<char> > >,mojo::InterfaceRequest<service_manager::mojom::Service>,mojo::InterfacePtr<service_manager::mojom::PIDReceiver> >,void (service_manager::mojom::ConnectResult, const std::basic_string<char,std::char_traits<char>,std::allocator<char> > &)>::RunOnce+0x3f [C:\b\c\b\win_clang\src\base\bind_internal.h @ 572] 
0b chrome!base::OnceCallback<void (blink::mojom::AppBannerPromptReply, const std::basic_string<char,std::char_traits<char>,std::allocator<char> > &)>::Run+0x1b [C:\b\c\b\win_clang\src\base\callback.h @ 95] 
0c chrome!blink::mojom::AppBannerController_BannerPromptRequest_ForwardToCallback::Accept+0x9f [C:\b\c\b\win_clang\src\out\Release\gen\third_party\WebKit\public\platform\modules\app_banner\app_banner.mojom.cc @ 195] 
0d chrome!mojo::InterfaceEndpointClient::HandleValidatedMessage+0x201 [C:\b\c\b\win_clang\src\mojo\public\cpp\bindings\lib\interface_endpoint_client.cc @ 414] 
0e chrome!mojo::internal::MultiplexRouter::ProcessIncomingMessage+0x17d [C:\b\c\b\win_clang\src\mojo\public\cpp\bindings\lib\multiplex_router.cc @ 881] 
0f chrome!mojo::internal::MultiplexRouter::Accept+0xae [C:\b\c\b\win_clang\src\mojo\public\cpp\bindings\lib\multiplex_router.cc @ 608] 
10 chrome!mojo::Connector::ReadSingleMessage+0xef [C:\b\c\b\win_clang\src\mojo\public\cpp\bindings\lib\connector.cc @ 444] 
11 chrome!mojo::Connector::ReadAllAvailableMessages+0x47 [C:\b\c\b\win_clang\src\mojo\public\cpp\bindings\lib\connector.cc @ 474] 
12 chrome!mojo::Connector::OnHandleReadyInternal+0x23 [C:\b\c\b\win_clang\src\mojo\public\cpp\bindings\lib\connector.cc @ 377] 
13 chrome!base::internal::FunctorTraits<void (base::internal::AdaptCallbackForRepeatingHelper<const SkBitmap &>::*)(const SkBitmap &) __attribute__((thiscall)),void>::Invoke+0x9 [C:\b\c\b\win_clang\src\base\bind_internal.h @ 447] 
14 chrome!base::internal::InvokeHelper<0,void>::MakeItSo+0x9 [C:\b\c\b\win_clang\src\base\bind_internal.h @ 530] 
15 chrome!base::internal::Invoker<base::internal::BindState<void (base::internal::AdaptCallbackForRepeatingHelper<const SkBitmap &>::*)(const SkBitmap &) __attribute__((thiscall)),std::unique_ptr<base::internal::AdaptCallbackForRepeatingHelper<const SkBitmap &>,std::default_delete<base::internal::AdaptCallbackForRepeatingHelper<const SkBitmap &> > > >,void (const SkBitmap &)>::RunImpl+0x9 [C:\b\c\b\win_clang\src\base\bind_internal.h @ 604] 
16 chrome!base::internal::Invoker<base::internal::BindState<void (base::internal::AdaptCallbackForRepeatingHelper<const SkBitmap &>::*)(const SkBitmap &) __attribute__((thiscall)),std::unique_ptr<base::internal::AdaptCallbackForRepeatingHelper<const SkBitmap &>,std::default_delete<base::internal::AdaptCallbackForRepeatingHelper<const SkBitmap &> > > >,void (const SkBitmap &)>::Run+0xf [C:\b\c\b\win_clang\src\base\bind_internal.h @ 589] 
17 chrome!base::RepeatingCallback<void (unsigned int)>::Run+0x9 [C:\b\c\b\win_clang\src\base\callback.h @ 124] 
18 chrome!mojo::SimpleWatcher::DiscardReadyState+0xf [C:\b\c\b\win_clang\src\mojo\public\cpp\system\simple_watcher.h @ 194] 
19 chrome!base::internal::FunctorTraits<void (*)(const InlineLoginHandlerImpl::FinishCompleteLoginParams &, Profile *, Profile::CreateStatus),void>::Invoke+0xa [C:\b\c\b\win_clang\src\base\bind_internal.h @ 402] 
1a chrome!base::internal::InvokeHelper<0,void>::MakeItSo+0xa [C:\b\c\b\win_clang\src\base\bind_internal.h @ 530] 
1b chrome!base::internal::Invoker<base::internal::BindState<void (*)(const InlineLoginHandlerImpl::FinishCompleteLoginParams &, Profile *, Profile::CreateStatus),InlineLoginHandlerImpl::FinishCompleteLoginParams>,void (Profile *, Profile::CreateStatus)>::RunImpl+0xa [C:\b\c\b\win_clang\src\base\bind_internal.h @ 604] 
1c chrome!base::internal::Invoker<base::internal::BindState<void (*)(const InlineLoginHandlerImpl::FinishCompleteLoginParams &, Profile *, Profile::CreateStatus),InlineLoginHandlerImpl::FinishCompleteLoginParams>,void (Profile *, Profile::CreateStatus)>::Run+0x13 [C:\b\c\b\win_clang\src\base\bind_internal.h @ 586] 
1d chrome!base::RepeatingCallback<void (unsigned int, const mojo::HandleSignalsState &)>::Run+0xb [C:\b\c\b\win_clang\src\base\callback.h @ 124] 
1e chrome!mojo::SimpleWatcher::OnHandleReady+0x91 [C:\b\c\b\win_clang\src\mojo\public\cpp\system\simple_watcher.cc @ 276] 
1f chrome!base::internal::FunctorTraits<void (mojo::SimpleWatcher::*)(int, unsigned int, const mojo::HandleSignalsState &) __attribute__((thiscall)),void>::Invoke+0x1c [C:\b\c\b\win_clang\src\base\bind_internal.h @ 447] 
20 chrome!base::internal::InvokeHelper<1,void>::MakeItSo+0x36 [C:\b\c\b\win_clang\src\base\bind_internal.h @ 550] 
21 chrome!base::internal::Invoker<base::internal::BindState<void (mojo::SimpleWatcher::*)(int, unsigned int, const mojo::HandleSignalsState &) __attribute__((thiscall)),base::WeakPtr<mojo::SimpleWatcher>,int,unsigned int,mojo::HandleSignalsState>,void ()>::RunImpl+0x36 [C:\b\c\b\win_clang\src\base\bind_internal.h @ 604] 
22 chrome!base::internal::Invoker<base::internal::BindState<void (mojo::SimpleWatcher::*)(int, unsigned int, const mojo::HandleSignalsState &) __attribute__((thiscall)),base::WeakPtr<mojo::SimpleWatcher>,int,unsigned int,mojo::HandleSignalsState>,void ()>::Run+0x40 [C:\b\c\b\win_clang\src\base\bind_internal.h @ 589] 
23 chrome!base::OnceCallback<void ()>::Run+0x10 [C:\b\c\b\win_clang\src\base\callback.h @ 95] 
24 chrome!base::debug::TaskAnnotator::RunTask+0x9f [C:\b\c\b\win_clang\src\base\debug\task_annotator.cc @ 61] 
25 chrome!base::internal::IncomingTaskQueue::RunTask+0x13 [C:\b\c\b\win_clang\src\base\message_loop\incoming_task_queue.cc @ 125] 
26 chrome!base::MessageLoop::RunTask+0x1b6 [C:\b\c\b\win_clang\src\base\message_loop\message_loop.cc @ 396] 
27 chrome!base::MessageLoop::DeferOrRunPendingTask+0x53 [C:\b\c\b\win_clang\src\base\message_loop\message_loop.cc @ 407] 
28 chrome!base::MessageLoop::DoWork+0xd3 [C:\b\c\b\win_clang\src\base\message_loop\message_loop.cc @ 451] 
29 chrome!base::MessagePumpForIO::DoRunLoop+0x11d [C:\b\c\b\win_clang\src\base\message_loop\message_pump_win.cc @ 476] 
2a chrome!base::MessagePumpWin::Run+0x6e [C:\b\c\b\win_clang\src\base\message_loop\message_pump_win.cc @ 58] 
2b chrome!base::MessageLoop::Run+0x1f [C:\b\c\b\win_clang\src\base\message_loop\message_loop.cc @ 346] 
2c chrome!base::RunLoop::Run+0x2e [C:\b\c\b\win_clang\src\base\run_loop.cc @ 139] 
2d chrome!base::Thread::Run+0xb [C:\b\c\b\win_clang\src\base\threading\thread.cc @ 256] 
2e chrome!content::BrowserThreadImpl::IOThreadRun+0x21 [C:\b\c\b\win_clang\src\content\browser\browser_thread_impl.cc @ 234] 
2f chrome!content::BrowserThreadImpl::Run+0x52 [C:\b\c\b\win_clang\src\content\browser\browser_thread_impl.cc @ 260] 
30 chrome!base::Thread::ThreadMain+0x155 [C:\b\c\b\win_clang\src\base\threading\thread.cc @ 341] 
31 chrome!base::`anonymous namespace'::ThreadFunc+0xbb [C:\b\c\b\win_clang\src\base\threading\platform_thread_win.cc @ 94] 
32 KERNEL32!BaseThreadInitThunk+0x24

00 KERNEL32!CreateNamedPipeWStub
01 chrome!mojo::edk::PlatformChannelPair::PlatformChannelPair+0xa4 [C:\b\c\b\win_clang\src\mojo\edk\embedder\platform_channel_pair_win.cc @ 37] 
02 chrome!mojo::edk::NodeController::SendBrokerClientInvitationOnIOThread+0x30 [C:\b\c\b\win_clang\src\mojo\edk\system\node_controller.cc @ 343] 
```

Utility Process在Main函数中，初始化UtilityThreadImpl的时候，会去初始化MojoIPCChannel，并通过Mojo EDK的Accept函数连上对应的Mojo server。

```cpp
00 chrome_child!mojo::edk::IncomingBrokerClientInvitation::Accept [C:\b\c\b\win_clang\src\mojo\edk\embedder\incoming_broker_client_invitation.cc @ 22] 
01 chrome_child!content::`anonymous namespace'::InitializeMojoIPCChannel+0x15e [C:\b\c\b\win_clang\src\content\child\child_thread_impl.cc @ 263] 
02 chrome_child!content::ChildThreadImpl::Init+0x1d3 [C:\b\c\b\win_clang\src\content\child\child_thread_impl.cc @ 458] 
03 chrome_child!content::ChildThreadImpl::ChildThreadImpl+0x177 [C:\b\c\b\win_clang\src\content\child\child_thread_impl.cc @ 392] 
04 chrome_child!content::UtilityThreadImpl::UtilityThreadImpl+0x6a [C:\b\c\b\win_clang\src\content\utility\utility_thread_impl.cc @ 60] 
05 chrome_child!content::UtilityMain+0x132 [C:\b\c\b\win_clang\src\content\utility\utility_main.cc @ 69] 
06 chrome_child!content::RunNamedProcessTypeMain+0x10c [C:\b\c\b\win_clang\src\content\app\content_main_runner.cc @ 423] 
07 chrome_child!content::ContentMainRunnerImpl::Run+0x8e [C:\b\c\b\win_clang\src\content\app\content_main_runner.cc @ 703] 
08 chrome_child!service_manager::Main+0x26e [C:\b\c\b\win_clang\src\services\service_manager\embedder\main.cc @ 453] 
09 chrome_child!content::ContentMain+0x33 [C:\b\c\b\win_clang\src\content\app\content_main.cc @ 19] 
0a chrome_child!ChromeMain+0x108 [C:\b\c\b\win_clang\src\chrome\app\chrome_main.cc @ 104] 
0b chrome!MainDllLoader::Launch+0x230 [C:\b\c\b\win_clang\src\chrome\app\main_dll_loader_win.cc @ 199] 
0c chrome!wWinMain+0x45d [C:\b\c\b\win_clang\src\chrome\app\chrome_exe_main_win.cc @ 231] 
```

## Linux

Linux上的Chrome会稍微有些差异，主要是由于zygote process的缘故。

读过Chromium的document以后，知道在Linux环境下面，IPC是通过socketpair实现的。
 
Linux环境下，Chrome在BrowserMainLoop::EarlyInitialization中会准备sandboxed的socket。

```cpp
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
                    --> SandboxHostLinux::Init
```

```cpp
void SetupSandbox(const base::CommandLine& parsed_command_line) {
  TRACE_EVENT0("startup", "SetupSandbox");
  // SandboxHostLinux needs to be initialized even if the sandbox and
  // zygote are both disabled. It initializes the sandboxed process socket.
  SandboxHostLinux::GetInstance()->Init();      // ---> here

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
#0  poll () at ../sysdeps/unix/syscall-template.S:84
#1  (anonymous namespace)::SandboxIPCHandler::Run() (this=0x5848e034200) at ../../content/browser/sandbox_ipc_linux.cc:100
#2  (anonymous namespace)::DelegateSimpleThread::Run() (this=0x5848dfe5700) at ../../base/threading/simple_thread.cc:92
#3  (anonymous namespace)::SimpleThread::ThreadMain() (this=0x5848dfe5700) at ../../base/threading/simple_thread.cc:68
#4  (anonymous namespace)::(anonymous namespace)::ThreadFunc(void*) (params=0x5848df9b700) at ../../base/threading/platform_thread_posix.cc:76
#5  start_thread (arg=0x7fffcd599700) at pthread_create.c:333
#6  clone () at ../sysdeps/unix/sysv/linux/x86_64/clone.S:109

```

```cpp
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

在上一篇，我们介绍过zygote process的启动过程，如下：

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
                            --> socketpair
```

```cpp
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

browser process需要创建render process时，会让zygote process fork一个process出来，同时，会将mojo ipc作为参数传给render process。







