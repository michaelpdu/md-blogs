# 使用content_shell加载本地文件

## 命令行参数

从下面这行命令开始说起：

```
content_shell.exe --enable-logging --v=1 --disable-gpu --no-sandbox --run-layout-test c:\Users\michael_du\Desktop\test.html
```

其中：
```
--enable-logging   开启logging的功能
--v=1              设置log level为全开模式
--disable-gpu      关闭GPU加速功能，GPU process也就不存在了
--no-sandbox       关闭sandbox功能，在关闭的时候，log可以显示在独立的console中，开启时，render process的log与browser process的log会显示在同一个console中
--run-layout-test  只会做底层的渲染工作，不会显示UI
```

另外，还有--blink-platform-log-channels，暂时还没有弄明白

## 代码主要逻辑

### 第一步

Browser Process准备navigate MSG，并发给Render Process

```
 # Call Site
00 content!content::RenderFrameHostImpl::SendNavigateMessage [d:\source_code\chromium\depot_tools\src\content\browser\frame_host\render_frame_host_impl.cc @ 2902]
01 content!content::RenderFrameHostImpl::Navigate+0x25f [d:\source_code\chromium\depot_tools\src\content\browser\frame_host\render_frame_host_impl.cc @ 2300]
02 content!content::NavigatorImpl::NavigateToEntry+0xdb1 [d:\source_code\chromium\depot_tools\src\content\browser\frame_host\navigator_impl.cc @ 391]
03 content!content::NavigatorImpl::NavigateToPendingEntry+0x81 [d:\source_code\chromium\depot_tools\src\content\browser\frame_host\navigator_impl.cc @ 440]
04 content!content::NavigationControllerImpl::NavigateToPendingEntryInternal+0x539 [d:\source_code\chromium\depot_tools\src\content\browser\frame_host\navigation_controller_impl.cc @ 1879]
05 content!content::NavigationControllerImpl::NavigateToPendingEntry+0x3c1 [d:\source_code\chromium\depot_tools\src\content\browser\frame_host\navigation_controller_impl.cc @ 1821]
06 content!content::NavigationControllerImpl::LoadEntry+0x48 [d:\source_code\chromium\depot_tools\src\content\browser\frame_host\navigation_controller_impl.cc @ 434]
07 content!content::NavigationControllerImpl::LoadURLWithParams+0xcc3 [d:\source_code\chromium\depot_tools\src\content\browser\frame_host\navigation_controller_impl.cc @ 764]
08 content_shell!content::Shell::LoadURLForFrame+0xa8 [d:\source_code\chromium\depot_tools\src\content\shell\browser\shell.cc @ 200]
09 content_shell!content::Shell::LoadURL+0x39 [d:\source_code\chromium\depot_tools\src\content\shell\browser\shell.cc @ 191]
0a content_shell!content::Shell::CreateNewWindow+0xca [d:\source_code\chromium\depot_tools\src\content\shell\browser\shell.cc @ 187]
0b content_shell!content::ShellBrowserMainParts::InitializeMessageLoopContext+0x6f [d:\source_code\chromium\depot_tools\src\content\shell\browser\shell_browser_main_parts.cc @ 160]
0c content_shell!content::ShellBrowserMainParts::PreMainMessageLoopRun+0x14b [d:\source_code\chromium\depot_tools\src\content\shell\browser\shell_browser_main_parts.cc @ 190]
0d content!content::BrowserMainLoop::PreMainMessageLoopRun+0x109 [d:\source_code\chromium\depot_tools\src\content\browser\browser_main_loop.cc @ 941]
0e content!base::internal::FunctorTraits<int (__cdecl content::BrowserMainLoop::*)(void) __ptr64,void>::Invoke<content::BrowserMainLoop * __ptr64>+0x24 [d:\source_code\chromium\depot_tools\src\base\bind_internal.h @ 215]
0f content!base::internal::InvokeHelper<0,int>::MakeItSo<int (__cdecl content::BrowserMainLoop::*const & __ptr64)(void) __ptr64,content::BrowserMainLoop * __ptr64>+0x37 [d:\source_code\chromium\depot_tools\src\base\bind_internal.h @ 287]
10 content!base::internal::Invoker<base::internal::BindState<int (__cdecl content::BrowserMainLoop::*)(void) __ptr64,base::internal::UnretainedWrapper<content::BrowserMainLoop> >,int __cdecl(void)>::RunImpl<int (__cdecl content::BrowserMainLoop::*const & __ptr64)(void) __ptr64,std::tuple<base::internal::UnretainedWrapper<content::BrowserMainLoop> > const & __ptr64,0>+0x49 [d:\source_code\chromium\depot_tools\src\base\bind_internal.h @ 365]
11 content!base::internal::Invoker<base::internal::BindState<int (__cdecl content::BrowserMainLoop::*)(void) __ptr64,base::internal::UnretainedWrapper<content::BrowserMainLoop> >,int __cdecl(void)>::Run+0x33 [d:\source_code\chromium\depot_tools\src\base\bind_internal.h @ 343]
12 content!base::internal::RunMixin<base::Callback<int __cdecl(void),1,1> >::Run+0x55 [d:\source_code\chromium\depot_tools\src\base\callback.h @ 65]
13 content!content::StartupTaskRunner::RunAllTasksNow+0x93 [d:\source_code\chromium\depot_tools\src\content\browser\startup_task_runner.cc @ 45]
14 content!content::BrowserMainLoop::CreateStartupTasks+0x30e [d:\source_code\chromium\depot_tools\src\content\browser\browser_main_loop.cc @ 832]
15 content!content::BrowserMainRunnerImpl::Initialize+0x53c [d:\source_code\chromium\depot_tools\src\content\browser\browser_main_runner.cc @ 141]
16 content_shell!ShellBrowserMain+0x3b [d:\source_code\chromium\depot_tools\src\content\shell\browser\shell_browser_main.cc @ 23]
17 content_shell!content::ShellMainDelegate::RunProcess+0xc5 [d:\source_code\chromium\depot_tools\src\content\shell\app\shell_main_delegate.cc @ 295]
18 content!content::RunNamedProcessTypeMain+0x9d [d:\source_code\chromium\depot_tools\src\content\app\content_main_runner.cc @ 405]
19 content!content::ContentMainRunnerImpl::Run+0x278 [d:\source_code\chromium\depot_tools\src\content\app\content_main_runner.cc @ 786]
1a content!content::ContentMain+0x81 [d:\source_code\chromium\depot_tools\src\content\app\content_main.cc @ 20]
1b content_shell!wWinMain+0x7b [d:\source_code\chromium\depot_tools\src\content\shell\app\shell_main.cc @ 33]
1c content_shell!invoke_main+0x2d [f:\dd\vctools\crt\vcstartup\src\startup\exe_common.inl @ 118]
1d content_shell!__scrt_common_main_seh+0x127 [f:\dd\vctools\crt\vcstartup\src\startup\exe_common.inl @ 253]
1e content_shell!__scrt_common_main+0xe [f:\dd\vctools\crt\vcstartup\src\startup\exe_common.inl @ 296]
1f content_shell!wWinMainCRTStartup+0x9 [f:\dd\vctools\crt\vcstartup\src\startup\exe_wwinmain.cpp @ 17]
20 KERNEL32!BaseThreadInitThunk+0x14
21 ntdll!RtlUserThreadStart+0x21
```

### 第二步

Render Process对Navigate MSG做处理，在StartAsync函数中发送ResourceHostMsg_RequestResource，请求资源

```
 # Call Site
00 content!content::ResourceDispatcher::StartAsync [d:\source_code\chromium\depot_tools\src\content\child\resource_dispatcher.cc @ 648]
01 content!content::WebURLLoaderImpl::Context::Start+0x1277 [d:\source_code\chromium\depot_tools\src\content\child\web_url_loader_impl.cc @ 600]
02 content!content::WebURLLoaderImpl::loadAsynchronously+0x21e [d:\source_code\chromium\depot_tools\src\content\child\web_url_loader_impl.cc @ 1192]
03 blink_core!blink::ResourceLoader::start+0x32b [d:\source_code\chromium\depot_tools\src\third_party\webkit\source\core\fetch\resourceloader.cpp @ 92]
04 blink_core!blink::ResourceFetcher::startLoad+0x420 [d:\source_code\chromium\depot_tools\src\third_party\webkit\source\core\fetch\resourcefetcher.cpp @ 1077]
05 blink_core!blink::ResourceFetcher::requestResource+0x1048 [d:\source_code\chromium\depot_tools\src\third_party\webkit\source\core\fetch\resourcefetcher.cpp @ 520]
06 blink_core!blink::RawResource::fetchMainResource+0x25d [d:\source_code\chromium\depot_tools\src\third_party\webkit\source\core\fetch\rawresource.cpp @ 63]
07 blink_core!blink::DocumentLoader::startLoadingMainResource+0x51f [d:\source_code\chromium\depot_tools\src\third_party\webkit\source\core\loader\documentloader.cpp @ 664]
08 blink_core!blink::FrameLoader::startLoad+0x812 [d:\source_code\chromium\depot_tools\src\third_party\webkit\source\core\loader\frameloader.cpp @ 1471]
09 blink_core!blink::FrameLoader::load+0x9d9 [d:\source_code\chromium\depot_tools\src\third_party\webkit\source\core\loader\frameloader.cpp @ 1024]
0a blink_web!blink::WebLocalFrameImpl::load+0x298 [d:\source_code\chromium\depot_tools\src\third_party\webkit\source\web\weblocalframeimpl.cpp @ 1884]
0b content!content::RenderFrameImpl::NavigateInternal+0x1297 [d:\source_code\chromium\depot_tools\src\content\renderer\render_frame_impl.cc @ 5696]
0c content!content::RenderFrameImpl::OnNavigate+0x2c7 [d:\source_code\chromium\depot_tools\src\content\renderer\render_frame_impl.cc @ 1614]
0d content!base::DispatchToMethodImpl<content::RenderFrameImpl * __ptr64,void (__cdecl content::RenderFrameImpl::*)(content::CommonNavigationParams const & __ptr64,content::StartNavigationParams const & __ptr64,content::RequestNavigationParams const & __ptr64) __ptr64,std::tuple<content::CommonNavigationParams,content::StartNavigationParams,content::RequestNavigationParams> const & __ptr64,0,1,2>+0x83 [d:\source_code\chromium\depot_tools\src\base\tuple.h @ 145]
0e content!base::DispatchToMethod<content::RenderFrameImpl * __ptr64,void (__cdecl content::RenderFrameImpl::*)(content::CommonNavigationParams const & __ptr64,content::StartNavigationParams const & __ptr64,content::RequestNavigationParams const & __ptr64) __ptr64,std::tuple<content::CommonNavigationParams,content::StartNavigationParams,content::RequestNavigationParams> const & __ptr64>+0x4b [d:\source_code\chromium\depot_tools\src\base\tuple.h @ 153]
0f content!IPC::DispatchToMethod<content::RenderFrameImpl,void (__cdecl content::RenderFrameImpl::*)(content::CommonNavigationParams const & __ptr64,content::StartNavigationParams const & __ptr64,content::RequestNavigationParams const & __ptr64) __ptr64,void,std::tuple<content::CommonNavigationParams,content::StartNavigationParams,content::RequestNavigationParams> >+0x42 [d:\source_code\chromium\depot_tools\src\ipc\ipc_message_templates.h @ 27]
10 content!IPC::MessageT<FrameMsg_Navigate_Meta,std::tuple<content::CommonNavigationParams,content::StartNavigationParams,content::RequestNavigationParams>,void>::Dispatch<content::RenderFrameImpl,content::RenderFrameImpl,void,void (__cdecl content::RenderFrameImpl::*)(content::CommonNavigationParams const & __ptr64,content::StartNavigationParams const & __ptr64,content::RequestNavigationParams const & __ptr64) __ptr64>+0x142 [d:\source_code\chromium\depot_tools\src\ipc\ipc_message_templates.h @ 122]
11 content!content::RenderFrameImpl::OnMessageReceived+0x3d0 [d:\source_code\chromium\depot_tools\src\content\renderer\render_frame_impl.cc @ 1480]
12 ipc!IPC::MessageRouter::RouteMessage+0x4d [d:\source_code\chromium\depot_tools\src\ipc\message_router.cc @ 57]
13 content!content::ChildThreadImpl::ChildThreadMessageRouter::RouteMessage+0x1e [d:\source_code\chromium\depot_tools\src\content\child\child_thread_impl.cc @ 369]
14 ipc!IPC::MessageRouter::OnMessageReceived+0x4b [d:\source_code\chromium\depot_tools\src\ipc\message_router.cc @ 49]
15 content!content::ChildThreadImpl::OnMessageReceived+0x73b [d:\source_code\chromium\depot_tools\src\content\child\child_thread_impl.cc @ 793]
16 ipc!IPC::ChannelProxy::Context::OnDispatchMessage+0x92 [d:\source_code\chromium\depot_tools\src\ipc\ipc_channel_proxy.cc @ 340]
17 ipc!base::internal::FunctorTraits<void (__cdecl IPC::ChannelProxy::Context::*)(IPC::Message const & __ptr64) __ptr64,void>::Invoke<scoped_refptr<IPC::ChannelProxy::Context> const & __ptr64,IPC::Message const & __ptr64>+0x4a [d:\source_code\chromium\depot_tools\src\base\bind_internal.h @ 215]
18 ipc!base::internal::InvokeHelper<0,void>::MakeItSo<void (__cdecl IPC::ChannelProxy::Context::*const & __ptr64)(IPC::Message const & __ptr64) __ptr64,scoped_refptr<IPC::ChannelProxy::Context> const & __ptr64,IPC::Message const & __ptr64>+0x69 [d:\source_code\chromium\depot_tools\src\base\bind_internal.h @ 287]
19 ipc!base::internal::Invoker<base::internal::BindState<void (__cdecl IPC::ChannelProxy::Context::*)(IPC::Message const & __ptr64) __ptr64,scoped_refptr<IPC::ChannelProxy::Context>,IPC::Message>,void __cdecl(void)>::RunImpl<void (__cdecl IPC::ChannelProxy::Context::*const & __ptr64)(IPC::Message const & __ptr64) __ptr64,std::tuple<scoped_refptr<IPC::ChannelProxy::Context>,IPC::Message> const & __ptr64,0,1>+0x73 [d:\source_code\chromium\depot_tools\src\base\bind_internal.h @ 365]
1a ipc!base::internal::Invoker<base::internal::BindState<void (__cdecl IPC::ChannelProxy::Context::*)(IPC::Message const & __ptr64) __ptr64,scoped_refptr<IPC::ChannelProxy::Context>,IPC::Message>,void __cdecl(void)>::Run+0x33 [d:\source_code\chromium\depot_tools\src\base\bind_internal.h @ 343]
1b base!base::internal::RunMixin<base::Callback<void __cdecl(void),1,1> >::Run+0x54 [d:\source_code\chromium\depot_tools\src\base\callback.h @ 65]
1c base!base::debug::TaskAnnotator::RunTask+0x1f7 [d:\source_code\chromium\depot_tools\src\base\debug\task_annotator.cc @ 56]
1d blink_platform!blink::scheduler::TaskQueueManager::ProcessTaskFromWorkQueue+0x606 [d:\source_code\chromium\depot_tools\src\third_party\webkit\source\platform\scheduler\base\task_queue_manager.cc @ 340]
1e blink_platform!blink::scheduler::TaskQueueManager::DoWork+0x411 [d:\source_code\chromium\depot_tools\src\third_party\webkit\source\platform\scheduler\base\task_queue_manager.cc @ 234]
1f blink_platform!base::internal::FunctorTraits<void (__cdecl blink::scheduler::TaskQueueManager::*)(base::TimeTicks,bool) __ptr64,void>::Invoke<base::WeakPtr<blink::scheduler::TaskQueueManager> const & __ptr64,base::TimeTicks const & __ptr64,bool const & __ptr64>+0x67 [d:\source_code\chromium\depot_tools\src\base\bind_internal.h @ 215]
20 blink_platform!base::internal::InvokeHelper<1,void>::MakeItSo<void (__cdecl blink::scheduler::TaskQueueManager::*const & __ptr64)(base::TimeTicks,bool) __ptr64,base::WeakPtr<blink::scheduler::TaskQueueManager> const & __ptr64,base::TimeTicks const & __ptr64,bool const & __ptr64>+0x9e [d:\source_code\chromium\depot_tools\src\base\bind_internal.h @ 308]
21 blink_platform!base::internal::Invoker<base::internal::BindState<void (__cdecl blink::scheduler::TaskQueueManager::*)(base::TimeTicks,bool) __ptr64,base::WeakPtr<blink::scheduler::TaskQueueManager>,base::TimeTicks,bool>,void __cdecl(void)>::RunImpl<void (__cdecl blink::scheduler::TaskQueueManager::*const & __ptr64)(base::TimeTicks,bool) __ptr64,std::tuple<base::WeakPtr<blink::scheduler::TaskQueueManager>,base::TimeTicks,bool> const & __ptr64,0,1,2>+0x9a [d:\source_code\chromium\depot_tools\src\base\bind_internal.h @ 365]
22 blink_platform!base::internal::Invoker<base::internal::BindState<void (__cdecl blink::scheduler::TaskQueueManager::*)(base::TimeTicks,bool) __ptr64,base::WeakPtr<blink::scheduler::TaskQueueManager>,base::TimeTicks,bool>,void __cdecl(void)>::Run+0x33 [d:\source_code\chromium\depot_tools\src\base\bind_internal.h @ 343]
23 base!base::internal::RunMixin<base::Callback<void __cdecl(void),1,1> >::Run+0x54 [d:\source_code\chromium\depot_tools\src\base\callback.h @ 65]
24 base!base::debug::TaskAnnotator::RunTask+0x1f7 [d:\source_code\chromium\depot_tools\src\base\debug\task_annotator.cc @ 56]
25 base!base::MessageLoop::RunTask+0x3b9 [d:\source_code\chromium\depot_tools\src\base\message_loop\message_loop.cc @ 470]
26 base!base::MessageLoop::DeferOrRunPendingTask+0x3c [d:\source_code\chromium\depot_tools\src\base\message_loop\message_loop.cc @ 481]
27 base!base::MessageLoop::DoWork+0x172 [d:\source_code\chromium\depot_tools\src\base\message_loop\message_loop.cc @ 602]
28 base!base::MessagePumpDefault::Run+0xfa [d:\source_code\chromium\depot_tools\src\base\message_loop\message_pump_default.cc @ 35]
29 base!base::MessageLoop::RunHandler+0xf6 [d:\source_code\chromium\depot_tools\src\base\message_loop\message_loop.cc @ 433]
2a base!base::RunLoop::Run+0x3d [d:\source_code\chromium\depot_tools\src\base\run_loop.cc @ 36]
2b content!content::RendererMain+0x3ca [d:\source_code\chromium\depot_tools\src\content\renderer\renderer_main.cc @ 198]
2c content!content::RunNamedProcessTypeMain+0xd4 [d:\source_code\chromium\depot_tools\src\content\app\content_main_runner.cc @ 418]
2d content!content::ContentMainRunnerImpl::Run+0x278 [d:\source_code\chromium\depot_tools\src\content\app\content_main_runner.cc @ 786]
2e content!content::ContentMain+0x81 [d:\source_code\chromium\depot_tools\src\content\app\content_main.cc @ 20]
2f content_shell!wWinMain+0x7b [d:\source_code\chromium\depot_tools\src\content\shell\app\shell_main.cc @ 33]
30 content_shell!invoke_main+0x2d [f:\dd\vctools\crt\vcstartup\src\startup\exe_common.inl @ 118]
31 content_shell!__scrt_common_main_seh+0x127 [f:\dd\vctools\crt\vcstartup\src\startup\exe_common.inl @ 253]
32 content_shell!__scrt_common_main+0xe [f:\dd\vctools\crt\vcstartup\src\startup\exe_common.inl @ 296]
33 content_shell!wWinMainCRTStartup+0x9 [f:\dd\vctools\crt\vcstartup\src\startup\exe_wwinmain.cpp @ 17]
34 KERNEL32!BaseThreadInitThunk+0x14
35 ntdll!RtlUserThreadStart+0x21
```

```C
int ResourceDispatcher::StartAsync(
    std::unique_ptr<ResourceRequest> request,
    int routing_id,
    scoped_refptr<base::SingleThreadTaskRunner> loading_task_runner,
    const GURL& frame_origin,
    std::unique_ptr<RequestPeer> peer,
    blink::WebURLRequest::LoadingIPCType ipc_type,
    mojom::URLLoaderFactory* url_loader_factory) {
  CheckSchemeForReferrerPolicy(*request);

  // Compute a unique request_id for this renderer process.
  int request_id = MakeRequestID();
  pending_requests_[request_id] = base::MakeUnique<PendingRequestInfo>(
      std::move(peer), request->resource_type, request->origin_pid,
      frame_origin, request->url, request->download_to_file);

  if (resource_scheduling_filter_.get() && loading_task_runner) {
    resource_scheduling_filter_->SetRequestIdTaskRunner(request_id,
                                                        loading_task_runner);
  }

  if (ipc_type == blink::WebURLRequest::LoadingIPCType::Mojo) {
    std::unique_ptr<URLLoaderClientImpl> client(
        new URLLoaderClientImpl(request_id, this, main_thread_task_runner_));
    mojom::URLLoaderPtr url_loader;
    url_loader_factory->CreateLoaderAndStart(
        GetProxy(&url_loader), request_id, *request,
        client->CreateInterfacePtrAndBind());
    pending_requests_[request_id]->url_loader = std::move(url_loader);
    pending_requests_[request_id]->url_loader_client = std::move(client);
  } else {
    message_sender_->Send(
        new ResourceHostMsg_RequestResource(routing_id, request_id, *request));
  }

  return request_id;
}
```

### 第三步

在Browser Process中对RequestResource MSG做处理，在处理函数OnRequestResource中，会创建一个NavigationResourceThrottle对象。

```C
bool ResourceDispatcherHostImpl::OnMessageReceived(
    const IPC::Message& message,
    ResourceMessageFilter* filter) {
  DCHECK_CURRENTLY_ON(BrowserThread::IO);
  filter_ = filter;
  bool handled = true;
  IPC_BEGIN_MESSAGE_MAP(ResourceDispatcherHostImpl, message)
    IPC_MESSAGE_HANDLER(ResourceHostMsg_RequestResource, OnRequestResource)
    IPC_MESSAGE_HANDLER_DELAY_REPLY(ResourceHostMsg_SyncLoad, OnSyncLoad)
    IPC_MESSAGE_HANDLER(ResourceHostMsg_ReleaseDownloadedFile,
                        OnReleaseDownloadedFile)
    IPC_MESSAGE_HANDLER(ResourceHostMsg_DataDownloaded_ACK, OnDataDownloadedACK)
    IPC_MESSAGE_HANDLER(ResourceHostMsg_CancelRequest, OnCancelRequest)
    IPC_MESSAGE_HANDLER(ResourceHostMsg_DidChangePriority, OnDidChangePriority)
    IPC_MESSAGE_UNHANDLED(handled = false)
  IPC_END_MESSAGE_MAP()
  ...
}
```

```
[24072:2920:0513/113626:401376921:VERBOSE1:resourcefetcher.cpp(592)] Loading Resource for "file:///C:/Users/michael_du/Desktop/2.html"

 # Call Site
00 content!content::NavigationResourceThrottle::NavigationResourceThrottle [d:\source_code\chromium\depot_tools\src\content\browser\loader\navigation_resource_throttle.cc @ 150]
01 content!content::ResourceDispatcherHostImpl::AddStandardHandlers+0x302 [d:\source_code\chromium\depot_tools\src\content\browser\loader\resource_dispatcher_host_impl.cc @ 1644]
02 content!content::ResourceDispatcherHostImpl::CreateResourceHandler+0x791 [d:\source_code\chromium\depot_tools\src\content\browser\loader\resource_dispatcher_host_impl.cc @ 1604]
03 content!content::ResourceDispatcherHostImpl::ContinuePendingBeginRequest+0x1481 [d:\source_code\chromium\depot_tools\src\content\browser\loader\resource_dispatcher_host_impl.cc @ 1520]
04 content!content::ResourceDispatcherHostImpl::BeginRequest+0xa6b [d:\source_code\chromium\depot_tools\src\content\browser\loader\resource_dispatcher_host_impl.cc @ 1292]
05 content!content::ResourceDispatcherHostImpl::OnRequestResourceInternal+0x1f1 [d:\source_code\chromium\depot_tools\src\content\browser\loader\resource_dispatcher_host_impl.cc @ 1062]
06 content!content::ResourceDispatcherHostImpl::OnRequestResource+0x7e [d:\source_code\chromium\depot_tools\src\content\browser\loader\resource_dispatcher_host_impl.cc @ 1032]
07 content!base::DispatchToMethodImpl<content::ResourceDispatcherHostImpl * __ptr64,void (__cdecl content::ResourceDispatcherHostImpl::*)(int,int,content::ResourceRequest const & __ptr64) __ptr64,std::tuple<int,int,content::ResourceRequest> const & __ptr64,0,1,2>+0x82 [d:\source_code\chromium\depot_tools\src\base\tuple.h @ 145]
08 content!base::DispatchToMethod<content::ResourceDispatcherHostImpl * __ptr64,void (__cdecl content::ResourceDispatcherHostImpl::*)(int,int,content::ResourceRequest const & __ptr64) __ptr64,std::tuple<int,int,content::ResourceRequest> const & __ptr64>+0x4b [d:\source_code\chromium\depot_tools\src\base\tuple.h @ 153]
09 content!IPC::DispatchToMethod<content::ResourceDispatcherHostImpl,void (__cdecl content::ResourceDispatcherHostImpl::*)(int,int,content::ResourceRequest const & __ptr64) __ptr64,void,std::tuple<int,int,content::ResourceRequest> >+0x42 [d:\source_code\chromium\depot_tools\src\ipc\ipc_message_templates.h @ 27]
0a content!IPC::MessageT<ResourceHostMsg_RequestResource_Meta,std::tuple<int,int,content::ResourceRequest>,void>::Dispatch<content::ResourceDispatcherHostImpl,content::ResourceDispatcherHostImpl,void,void (__cdecl content::ResourceDispatcherHostImpl::*)(int,int,content::ResourceRequest const & __ptr64) __ptr64>+0x142 [d:\source_code\chromium\depot_tools\src\ipc\ipc_message_templates.h @ 122]
0b content!content::ResourceDispatcherHostImpl::OnMessageReceived+0x263 [d:\source_code\chromium\depot_tools\src\content\browser\loader\resource_dispatcher_host_impl.cc @ 992]
0c content!content::ResourceMessageFilter::OnMessageReceived+0x25 [d:\source_code\chromium\depot_tools\src\content\browser\loader\resource_message_filter.cc @ 50]
0d content!content::BrowserMessageFilter::Internal::DispatchMessageW+0x4a [d:\source_code\chromium\depot_tools\src\content\public\browser\browser_message_filter.cc @ 90]
0e content!content::BrowserMessageFilter::Internal::OnMessageReceived+0x1b2 [d:\source_code\chromium\depot_tools\src\content\public\browser\browser_message_filter.cc @ 70]
0f ipc!IPC::`anonymous namespace'::TryFiltersImpl+0x63 [d:\source_code\chromium\depot_tools\src\ipc\message_filter_router.cc @ 22]
10 ipc!IPC::MessageFilterRouter::TryFilters+0x70 [d:\source_code\chromium\depot_tools\src\ipc\message_filter_router.cc @ 88]
11 ipc!IPC::ChannelProxy::Context::TryFilters+0x13a [d:\source_code\chromium\depot_tools\src\ipc\ipc_channel_proxy.cc @ 101]
12 ipc!IPC::ChannelProxy::Context::OnMessageReceived+0x1d [d:\source_code\chromium\depot_tools\src\ipc\ipc_channel_proxy.cc @ 136]
13 ipc!IPC::ChannelMojo::OnMessageReceived+0x18d [d:\source_code\chromium\depot_tools\src\ipc\ipc_channel_mojo.cc @ 407]
14 ipc!IPC::internal::MessagePipeReader::Receive+0x3e5 [d:\source_code\chromium\depot_tools\src\ipc\ipc_message_pipe_reader.cc @ 114]
15 ipc!IPC::mojom::ChannelStub::Accept+0x523 [d:\source_code\chromium\depot_tools\src\out\win_console_app\gen\ipc\ipc.mojom.cc @ 245]
16 ipc!mojo::InterfaceEndpointClient::HandleValidatedMessage+0x6a1 [d:\source_code\chromium\depot_tools\src\mojo\public\cpp\bindings\lib\interface_endpoint_client.cc @ 341]
17 ipc!mojo::InterfaceEndpointClient::HandleIncomingMessageThunk::Accept+0x21 [d:\source_code\chromium\depot_tools\src\mojo\public\cpp\bindings\lib\interface_endpoint_client.cc @ 129]
18 ipc!mojo::FilterChain::Accept+0x1c1 [d:\source_code\chromium\depot_tools\src\mojo\public\cpp\bindings\lib\filter_chain.cc @ 41]
19 ipc!mojo::InterfaceEndpointClient::HandleIncomingMessage+0x110 [d:\source_code\chromium\depot_tools\src\mojo\public\cpp\bindings\lib\interface_endpoint_client.cc @ 274]
1a ipc!IPC::`anonymous namespace'::ChannelAssociatedGroupController::Accept+0x718 [d:\source_code\chromium\depot_tools\src\ipc\ipc_mojo_bootstrap.cc @ 644]
1b ipc!mojo::FilterChain::Accept+0x1c1 [d:\source_code\chromium\depot_tools\src\mojo\public\cpp\bindings\lib\filter_chain.cc @ 41]
1c ipc!mojo::Connector::ReadSingleMessage+0x127 [d:\source_code\chromium\depot_tools\src\mojo\public\cpp\bindings\lib\connector.cc @ 246]
1d ipc!mojo::Connector::ReadAllAvailableMessages+0x25 [d:\source_code\chromium\depot_tools\src\mojo\public\cpp\bindings\lib\connector.cc @ 272]
1e ipc!mojo::Connector::OnHandleReadyInternal+0x121 [d:\source_code\chromium\depot_tools\src\mojo\public\cpp\bindings\lib\connector.cc @ 207]
1f ipc!mojo::Connector::OnWatcherHandleReady+0x1b [d:\source_code\chromium\depot_tools\src\mojo\public\cpp\bindings\lib\connector.cc @ 184]
20 ipc!base::internal::FunctorTraits<void (__cdecl mojo::Connector::*)(unsigned int) __ptr64,void>::Invoke<mojo::Connector * __ptr64,unsigned int>+0x35 [d:\source_code\chromium\depot_tools\src\base\bind_internal.h @ 215]
21 ipc!base::internal::InvokeHelper<0,void>::MakeItSo<void (__cdecl mojo::Connector::*const & __ptr64)(unsigned int) __ptr64,mojo::Connector * __ptr64,unsigned int>+0x53 [d:\source_code\chromium\depot_tools\src\base\bind_internal.h @ 287]
22 ipc!base::internal::Invoker<base::internal::BindState<void (__cdecl mojo::Connector::*)(unsigned int) __ptr64,base::internal::UnretainedWrapper<mojo::Connector> >,void __cdecl(unsigned int)>::RunImpl<void (__cdecl mojo::Connector::*const & __ptr64)(unsigned int) __ptr64,std::tuple<base::internal::UnretainedWrapper<mojo::Connector> > const & __ptr64,0>+0x65 [d:\source_code\chromium\depot_tools\src\base\bind_internal.h @ 365]
23 ipc!base::internal::Invoker<base::internal::BindState<void (__cdecl mojo::Connector::*)(unsigned int) __ptr64,base::internal::UnretainedWrapper<mojo::Connector> >,void __cdecl(unsigned int)>::Run+0x52 [d:\source_code\chromium\depot_tools\src\base\bind_internal.h @ 343]
24 ipc!base::internal::RunMixin<base::Callback<void __cdecl(unsigned int),1,1> >::Run+0x70 [d:\source_code\chromium\depot_tools\src\base\callback.h @ 65]
25 ipc!mojo::Watcher::OnHandleReady+0x163 [d:\source_code\chromium\depot_tools\src\mojo\public\cpp\system\watcher.cc @ 123]
26 ipc!base::internal::FunctorTraits<void (__cdecl mojo::Watcher::*)(unsigned int) __ptr64,void>::Invoke<base::WeakPtr<mojo::Watcher> const & __ptr64,unsigned int const & __ptr64>+0x37 [d:\source_code\chromium\depot_tools\src\base\bind_internal.h @ 215]
27 ipc!base::internal::InvokeHelper<1,void>::MakeItSo<void (__cdecl mojo::Watcher::*const & __ptr64)(unsigned int) __ptr64,base::WeakPtr<mojo::Watcher> const & __ptr64,unsigned int const & __ptr64>+0x66 [d:\source_code\chromium\depot_tools\src\base\bind_internal.h @ 308]
28 ipc!base::internal::Invoker<base::internal::BindState<void (__cdecl mojo::Watcher::*)(unsigned int) __ptr64,base::WeakPtr<mojo::Watcher>,unsigned int>,void __cdecl(void)>::RunImpl<void (__cdecl mojo::Watcher::*const & __ptr64)(unsigned int) __ptr64,std::tuple<base::WeakPtr<mojo::Watcher>,unsigned int> const & __ptr64,0,1>+0x73 [d:\source_code\chromium\depot_tools\src\base\bind_internal.h @ 365]
29 ipc!base::internal::Invoker<base::internal::BindState<void (__cdecl mojo::Watcher::*)(unsigned int) __ptr64,base::WeakPtr<mojo::Watcher>,unsigned int>,void __cdecl(void)>::Run+0x33 [d:\source_code\chromium\depot_tools\src\base\bind_internal.h @ 343]
2a base!base::internal::RunMixin<base::Callback<void __cdecl(void),1,1> >::Run+0x54 [d:\source_code\chromium\depot_tools\src\base\callback.h @ 65]
2b base!base::debug::TaskAnnotator::RunTask+0x1f7 [d:\source_code\chromium\depot_tools\src\base\debug\task_annotator.cc @ 56]
2c base!base::MessageLoop::RunTask+0x3b9 [d:\source_code\chromium\depot_tools\src\base\message_loop\message_loop.cc @ 470]
2d base!base::MessageLoop::DeferOrRunPendingTask+0x3c [d:\source_code\chromium\depot_tools\src\base\message_loop\message_loop.cc @ 481]
2e base!base::MessageLoop::DoWork+0x172 [d:\source_code\chromium\depot_tools\src\base\message_loop\message_loop.cc @ 602]
2f base!base::MessagePumpForIO::DoRunLoop+0x27 [d:\source_code\chromium\depot_tools\src\base\message_loop\message_pump_win.cc @ 732]
30 base!base::MessagePumpWin::Run+0x9d [d:\source_code\chromium\depot_tools\src\base\message_loop\message_pump_win.cc @ 143]
31 base!base::MessageLoop::RunHandler+0xf6 [d:\source_code\chromium\depot_tools\src\base\message_loop\message_loop.cc @ 433]
32 base!base::RunLoop::Run+0x3d [d:\source_code\chromium\depot_tools\src\base\run_loop.cc @ 36]
33 base!base::Thread::Run+0x195 [d:\source_code\chromium\depot_tools\src\base\threading\thread.cc @ 242]
34 content!content::BrowserThreadImpl::IOThreadRun+0x2f [d:\source_code\chromium\depot_tools\src\content\browser\browser_thread_impl.cc @ 244]
35 content!content::BrowserThreadImpl::Run+0x20a [d:\source_code\chromium\depot_tools\src\content\browser\browser_thread_impl.cc @ 278]
36 base!base::Thread::ThreadMain+0x556 [d:\source_code\chromium\depot_tools\src\base\threading\thread.cc @ 322]
37 base!base::`anonymous namespace'::ThreadFunc+0x131 [d:\source_code\chromium\depot_tools\src\base\threading\platform_thread_win.cc @ 86]
38 KERNEL32!BaseThreadInitThunk+0x14
39 ntdll!RtlUserThreadStart+0x21

 # Call Site
00 content!content::NavigationResourceThrottle::WillStartRequest [d:\source_code\chromium\depot_tools\src\content\browser\loader\navigation_resource_throttle.cc @ 154]
01 content!content::ThrottlingResourceHandler::OnWillStart+0x159 [d:\source_code\chromium\depot_tools\src\content\browser\loader\throttling_resource_handler.cc @ 70]
02 content!content::MimeSniffingResourceHandler::OnWillStart+0x1e2 [d:\source_code\chromium\depot_tools\src\content\browser\loader\mime_sniffing_resource_handler.cc @ 139]
03 content!content::ThrottlingResourceHandler::OnWillStart+0x241 [d:\source_code\chromium\depot_tools\src\content\browser\loader\throttling_resource_handler.cc @ 84]
04 content!content::ResourceLoader::StartRequest+0xfa [d:\source_code\chromium\depot_tools\src\content\browser\loader\resource_loader.cc @ 170]
05 content!content::ResourceDispatcherHostImpl::StartLoading+0xb5 [d:\source_code\chromium\depot_tools\src\content\browser\loader\resource_dispatcher_host_impl.cc @ 2401]
06 content!content::ResourceDispatcherHostImpl::BeginRequestInternal+0x5c3 [d:\source_code\chromium\depot_tools\src\content\browser\loader\resource_dispatcher_host_impl.cc @ 2328]
07 content!content::ResourceDispatcherHostImpl::ContinuePendingBeginRequest+0x1512 [d:\source_code\chromium\depot_tools\src\content\browser\loader\resource_dispatcher_host_impl.cc @ 1522]
08 content!content::ResourceDispatcherHostImpl::BeginRequest+0xa6b [d:\source_code\chromium\depot_tools\src\content\browser\loader\resource_dispatcher_host_impl.cc @ 1292]
09 content!content::ResourceDispatcherHostImpl::OnRequestResourceInternal+0x1f1 [d:\source_code\chromium\depot_tools\src\content\browser\loader\resource_dispatcher_host_impl.cc @ 1062]
0a content!content::ResourceDispatcherHostImpl::OnRequestResource+0x7e [d:\source_code\chromium\depot_tools\src\content\browser\loader\resource_dispatcher_host_impl.cc @ 1032]
0b content!base::DispatchToMethodImpl<content::ResourceDispatcherHostImpl * __ptr64,void (__cdecl content::ResourceDispatcherHostImpl::*)(int,int,content::ResourceRequest const & __ptr64) __ptr64,std::tuple<int,int,content::ResourceRequest> const & __ptr64,0,1,2>+0x82 [d:\source_code\chromium\depot_tools\src\base\tuple.h @ 145]
0c content!base::DispatchToMethod<content::ResourceDispatcherHostImpl * __ptr64,void (__cdecl content::ResourceDispatcherHostImpl::*)(int,int,content::ResourceRequest const & __ptr64) __ptr64,std::tuple<int,int,content::ResourceRequest> const & __ptr64>+0x4b [d:\source_code\chromium\depot_tools\src\base\tuple.h @ 153]
0d content!IPC::DispatchToMethod<content::ResourceDispatcherHostImpl,void (__cdecl content::ResourceDispatcherHostImpl::*)(int,int,content::ResourceRequest const & __ptr64) __ptr64,void,std::tuple<int,int,content::ResourceRequest> >+0x42 [d:\source_code\chromium\depot_tools\src\ipc\ipc_message_templates.h @ 27]
0e content!IPC::MessageT<ResourceHostMsg_RequestResource_Meta,std::tuple<int,int,content::ResourceRequest>,void>::Dispatch<content::ResourceDispatcherHostImpl,content::ResourceDispatcherHostImpl,void,void (__cdecl content::ResourceDispatcherHostImpl::*)(int,int,content::ResourceRequest const & __ptr64) __ptr64>+0x142 [d:\source_code\chromium\depot_tools\src\ipc\ipc_message_templates.h @ 122]
0f content!content::ResourceDispatcherHostImpl::OnMessageReceived+0x263 [d:\source_code\chromium\depot_tools\src\content\browser\loader\resource_dispatcher_host_impl.cc @ 992]
10 content!content::ResourceMessageFilter::OnMessageReceived+0x25 [d:\source_code\chromium\depot_tools\src\content\browser\loader\resource_message_filter.cc @ 50]
11 content!content::BrowserMessageFilter::Internal::DispatchMessageW+0x4a [d:\source_code\chromium\depot_tools\src\content\public\browser\browser_message_filter.cc @ 90]
12 content!content::BrowserMessageFilter::Internal::OnMessageReceived+0x1b2 [d:\source_code\chromium\depot_tools\src\content\public\browser\browser_message_filter.cc @ 70]
13 ipc!IPC::`anonymous namespace'::TryFiltersImpl+0x63 [d:\source_code\chromium\depot_tools\src\ipc\message_filter_router.cc @ 22]
14 ipc!IPC::MessageFilterRouter::TryFilters+0x70 [d:\source_code\chromium\depot_tools\src\ipc\message_filter_router.cc @ 88]
15 ipc!IPC::ChannelProxy::Context::TryFilters+0x13a [d:\source_code\chromium\depot_tools\src\ipc\ipc_channel_proxy.cc @ 101]
16 ipc!IPC::ChannelProxy::Context::OnMessageReceived+0x1d [d:\source_code\chromium\depot_tools\src\ipc\ipc_channel_proxy.cc @ 136]
17 ipc!IPC::ChannelMojo::OnMessageReceived+0x18d [d:\source_code\chromium\depot_tools\src\ipc\ipc_channel_mojo.cc @ 407]
18 ipc!IPC::internal::MessagePipeReader::Receive+0x3e5 [d:\source_code\chromium\depot_tools\src\ipc\ipc_message_pipe_reader.cc @ 114]
19 ipc!IPC::mojom::ChannelStub::Accept+0x523 [d:\source_code\chromium\depot_tools\src\out\win_console_app\gen\ipc\ipc.mojom.cc @ 245]
1a ipc!mojo::InterfaceEndpointClient::HandleValidatedMessage+0x6a1 [d:\source_code\chromium\depot_tools\src\mojo\public\cpp\bindings\lib\interface_endpoint_client.cc @ 341]
1b ipc!mojo::InterfaceEndpointClient::HandleIncomingMessageThunk::Accept+0x21 [d:\source_code\chromium\depot_tools\src\mojo\public\cpp\bindings\lib\interface_endpoint_client.cc @ 129]
1c ipc!mojo::FilterChain::Accept+0x1c1 [d:\source_code\chromium\depot_tools\src\mojo\public\cpp\bindings\lib\filter_chain.cc @ 41]
1d ipc!mojo::InterfaceEndpointClient::HandleIncomingMessage+0x110 [d:\source_code\chromium\depot_tools\src\mojo\public\cpp\bindings\lib\interface_endpoint_client.cc @ 274]
1e ipc!IPC::`anonymous namespace'::ChannelAssociatedGroupController::Accept+0x718 [d:\source_code\chromium\depot_tools\src\ipc\ipc_mojo_bootstrap.cc @ 644]
1f ipc!mojo::FilterChain::Accept+0x1c1 [d:\source_code\chromium\depot_tools\src\mojo\public\cpp\bindings\lib\filter_chain.cc @ 41]
20 ipc!mojo::Connector::ReadSingleMessage+0x127 [d:\source_code\chromium\depot_tools\src\mojo\public\cpp\bindings\lib\connector.cc @ 246]
21 ipc!mojo::Connector::ReadAllAvailableMessages+0x25 [d:\source_code\chromium\depot_tools\src\mojo\public\cpp\bindings\lib\connector.cc @ 272]
22 ipc!mojo::Connector::OnHandleReadyInternal+0x121 [d:\source_code\chromium\depot_tools\src\mojo\public\cpp\bindings\lib\connector.cc @ 207]
23 ipc!mojo::Connector::OnWatcherHandleReady+0x1b [d:\source_code\chromium\depot_tools\src\mojo\public\cpp\bindings\lib\connector.cc @ 184]
24 ipc!base::internal::FunctorTraits<void (__cdecl mojo::Connector::*)(unsigned int) __ptr64,void>::Invoke<mojo::Connector * __ptr64,unsigned int>+0x35 [d:\source_code\chromium\depot_tools\src\base\bind_internal.h @ 215]
25 ipc!base::internal::InvokeHelper<0,void>::MakeItSo<void (__cdecl mojo::Connector::*const & __ptr64)(unsigned int) __ptr64,mojo::Connector * __ptr64,unsigned int>+0x53 [d:\source_code\chromium\depot_tools\src\base\bind_internal.h @ 287]
26 ipc!base::internal::Invoker<base::internal::BindState<void (__cdecl mojo::Connector::*)(unsigned int) __ptr64,base::internal::UnretainedWrapper<mojo::Connector> >,void __cdecl(unsigned int)>::RunImpl<void (__cdecl mojo::Connector::*const & __ptr64)(unsigned int) __ptr64,std::tuple<base::internal::UnretainedWrapper<mojo::Connector> > const & __ptr64,0>+0x65 [d:\source_code\chromium\depot_tools\src\base\bind_internal.h @ 365]
27 ipc!base::internal::Invoker<base::internal::BindState<void (__cdecl mojo::Connector::*)(unsigned int) __ptr64,base::internal::UnretainedWrapper<mojo::Connector> >,void __cdecl(unsigned int)>::Run+0x52 [d:\source_code\chromium\depot_tools\src\base\bind_internal.h @ 343]
28 ipc!base::internal::RunMixin<base::Callback<void __cdecl(unsigned int),1,1> >::Run+0x70 [d:\source_code\chromium\depot_tools\src\base\callback.h @ 65]
29 ipc!mojo::Watcher::OnHandleReady+0x163 [d:\source_code\chromium\depot_tools\src\mojo\public\cpp\system\watcher.cc @ 123]
2a ipc!base::internal::FunctorTraits<void (__cdecl mojo::Watcher::*)(unsigned int) __ptr64,void>::Invoke<base::WeakPtr<mojo::Watcher> const & __ptr64,unsigned int const & __ptr64>+0x37 [d:\source_code\chromium\depot_tools\src\base\bind_internal.h @ 215]
2b ipc!base::internal::InvokeHelper<1,void>::MakeItSo<void (__cdecl mojo::Watcher::*const & __ptr64)(unsigned int) __ptr64,base::WeakPtr<mojo::Watcher> const & __ptr64,unsigned int const & __ptr64>+0x66 [d:\source_code\chromium\depot_tools\src\base\bind_internal.h @ 308]
2c ipc!base::internal::Invoker<base::internal::BindState<void (__cdecl mojo::Watcher::*)(unsigned int) __ptr64,base::WeakPtr<mojo::Watcher>,unsigned int>,void __cdecl(void)>::RunImpl<void (__cdecl mojo::Watcher::*const & __ptr64)(unsigned int) __ptr64,std::tuple<base::WeakPtr<mojo::Watcher>,unsigned int> const & __ptr64,0,1>+0x73 [d:\source_code\chromium\depot_tools\src\base\bind_internal.h @ 365]
2d ipc!base::internal::Invoker<base::internal::BindState<void (__cdecl mojo::Watcher::*)(unsigned int) __ptr64,base::WeakPtr<mojo::Watcher>,unsigned int>,void __cdecl(void)>::Run+0x33 [d:\source_code\chromium\depot_tools\src\base\bind_internal.h @ 343]
2e base!base::internal::RunMixin<base::Callback<void __cdecl(void),1,1> >::Run+0x54 [d:\source_code\chromium\depot_tools\src\base\callback.h @ 65]
2f base!base::debug::TaskAnnotator::RunTask+0x1f7 [d:\source_code\chromium\depot_tools\src\base\debug\task_annotator.cc @ 56]
30 base!base::MessageLoop::RunTask+0x3b9 [d:\source_code\chromium\depot_tools\src\base\message_loop\message_loop.cc @ 470]
31 base!base::MessageLoop::DeferOrRunPendingTask+0x3c [d:\source_code\chromium\depot_tools\src\base\message_loop\message_loop.cc @ 481]
32 base!base::MessageLoop::DoWork+0x172 [d:\source_code\chromium\depot_tools\src\base\message_loop\message_loop.cc @ 602]
33 base!base::MessagePumpForIO::DoRunLoop+0x27 [d:\source_code\chromium\depot_tools\src\base\message_loop\message_pump_win.cc @ 732]
34 base!base::MessagePumpWin::Run+0x9d [d:\source_code\chromium\depot_tools\src\base\message_loop\message_pump_win.cc @ 143]
35 base!base::MessageLoop::RunHandler+0xf6 [d:\source_code\chromium\depot_tools\src\base\message_loop\message_loop.cc @ 433]
36 base!base::RunLoop::Run+0x3d [d:\source_code\chromium\depot_tools\src\base\run_loop.cc @ 36]
37 base!base::Thread::Run+0x195 [d:\source_code\chromium\depot_tools\src\base\threading\thread.cc @ 242]
38 content!content::BrowserThreadImpl::IOThreadRun+0x2f [d:\source_code\chromium\depot_tools\src\content\browser\browser_thread_impl.cc @ 244]
39 content!content::BrowserThreadImpl::Run+0x20a [d:\source_code\chromium\depot_tools\src\content\browser\browser_thread_impl.cc @ 278]
3a base!base::Thread::ThreadMain+0x556 [d:\source_code\chromium\depot_tools\src\base\threading\thread.cc @ 322]
3b base!base::`anonymous namespace'::ThreadFunc+0x131 [d:\source_code\chromium\depot_tools\src\base\threading\platform_thread_win.cc @ 86]
3c KERNEL32!BaseThreadInitThunk+0x14
3d ntdll!RtlUserThreadStart+0x21

[16436:23504:0513/113646:401396718:VERBOSE1:resource_loader.cc(338)] OnResponseStarted: file:///C:/Users/michael_du/Desktop/2.html

 # Call Site
00 content!content::NavigationResourceThrottle::WillProcessResponse [d:\source_code\chromium\depot_tools\src\content\browser\loader\navigation_resource_throttle.cc @ 226]
01 content!content::ThrottlingResourceHandler::OnResponseStarted+0x148 [d:\source_code\chromium\depot_tools\src\content\browser\loader\throttling_resource_handler.cc @ 93]
02 content!content::MimeSniffingResourceHandler::ReplayResponseReceived+0x11c [d:\source_code\chromium\depot_tools\src\content\browser\loader\mime_sniffing_resource_handler.cc @ 310]
03 content!content::MimeSniffingResourceHandler::ProcessState+0xa9 [d:\source_code\chromium\depot_tools\src\content\browser\loader\mime_sniffing_resource_handler.cc @ 280]
04 content!content::MimeSniffingResourceHandler::OnResponseStarted+0x245 [d:\source_code\chromium\depot_tools\src\content\browser\loader\mime_sniffing_resource_handler.cc @ 170]
05 content!content::ThrottlingResourceHandler::OnResponseStarted+0x21c [d:\source_code\chromium\depot_tools\src\content\browser\loader\throttling_resource_handler.cc @ 107]
06 content!content::ResourceLoader::CompleteResponseStarted+0x150 [d:\source_code\chromium\depot_tools\src\content\browser\loader\resource_loader.cc @ 546]
07 content!content::ResourceLoader::OnResponseStarted+0x2d0 [d:\source_code\chromium\depot_tools\src\content\browser\loader\resource_loader.cc @ 347]
08 net!net::URLRequest::Delegate::OnResponseStarted+0x28 [d:\source_code\chromium\depot_tools\src\net\url_request\url_request.cc @ 167]
09 net!net::URLRequest::NotifyResponseStarted+0x172 [d:\source_code\chromium\depot_tools\src\net\url_request\url_request.cc @ 854]
0a net!net::URLRequestJob::NotifyHeadersComplete+0x557 [d:\source_code\chromium\depot_tools\src\net\url_request\url_request_job.cc @ 502]
0b net!net::URLRequestFileJob::DidSeek+0x86 [d:\source_code\chromium\depot_tools\src\net\url_request\url_request_file_job.cc @ 288]
0c net!net::URLRequestFileJob::DidOpen+0x30f [d:\source_code\chromium\depot_tools\src\net\url_request\url_request_file_job.cc @ 276]
0d net!base::internal::FunctorTraits<void (__cdecl net::URLRequestFileJob::*)(int) __ptr64,void>::Invoke<base::WeakPtr<net::URLRequestFileJob> const & __ptr64,int>+0x37 [d:\source_code\chromium\depot_tools\src\base\bind_internal.h @ 215]
0e net!base::internal::InvokeHelper<1,void>::MakeItSo<void (__cdecl net::URLRequestFileJob::*const & __ptr64)(int) __ptr64,base::WeakPtr<net::URLRequestFileJob> const & __ptr64,int>+0x66 [d:\source_code\chromium\depot_tools\src\base\bind_internal.h @ 308]
0f net!base::internal::Invoker<base::internal::BindState<void (__cdecl net::URLRequestFileJob::*)(int) __ptr64,base::WeakPtr<net::URLRequestFileJob> >,void __cdecl(int)>::RunImpl<void (__cdecl net::URLRequestFileJob::*const & __ptr64)(int) __ptr64,std::tuple<base::WeakPtr<net::URLRequestFileJob> > const & __ptr64,0>+0x68 [d:\source_code\chromium\depot_tools\src\base\bind_internal.h @ 365]
10 net!base::internal::Invoker<base::internal::BindState<void (__cdecl net::URLRequestFileJob::*)(int) __ptr64,base::WeakPtr<net::URLRequestFileJob> >,void __cdecl(int)>::Run+0x52 [d:\source_code\chromium\depot_tools\src\base\bind_internal.h @ 343]
11 net!base::internal::RunMixin<base::Callback<void __cdecl(int),1,1> >::Run+0x70 [d:\source_code\chromium\depot_tools\src\base\callback.h @ 65]
12 net!net::`anonymous namespace'::CallInt64ToInt+0x23 [d:\source_code\chromium\depot_tools\src\net\base\file_stream_context.cc @ 28]
13 net!base::internal::FunctorTraits<void (__cdecl*)(base::Callback<void __cdecl(int),1,1> const & __ptr64,__int64),void>::Invoke<base::Callback<void __cdecl(int),1,1> const & __ptr64,__int64>+0x3b [d:\source_code\chromium\depot_tools\src\base\bind_internal.h @ 165]
14 net!base::internal::InvokeHelper<0,void>::MakeItSo<void (__cdecl*const & __ptr64)(base::Callback<void __cdecl(int),1,1> const & __ptr64,__int64),base::Callback<void __cdecl(int),1,1> const & __ptr64,__int64>+0x53 [d:\source_code\chromium\depot_tools\src\base\bind_internal.h @ 287]
15 net!base::internal::Invoker<base::internal::BindState<void (__cdecl*)(base::Callback<void __cdecl(int),1,1> const & __ptr64,__int64),base::Callback<void __cdecl(int),1,1> >,void __cdecl(__int64)>::RunImpl<void (__cdecl*const & __ptr64)(base::Callback<void __cdecl(int),1,1> const & __ptr64,__int64),std::tuple<base::Callback<void __cdecl(int),1,1> > const & __ptr64,0>+0x68 [d:\source_code\chromium\depot_tools\src\base\bind_internal.h @ 365]
16 net!base::internal::Invoker<base::internal::BindState<void (__cdecl*)(base::Callback<void __cdecl(int),1,1> const & __ptr64,__int64),base::Callback<void __cdecl(int),1,1> >,void __cdecl(__int64)>::Run+0x52 [d:\source_code\chromium\depot_tools\src\base\bind_internal.h @ 343]
17 net!base::internal::RunMixin<base::Callback<void __cdecl(__int64),1,1> >::Run+0x71 [d:\source_code\chromium\depot_tools\src\base\callback.h @ 65]
18 net!net::FileStream::Context::OnAsyncCompleted+0x98 [d:\source_code\chromium\depot_tools\src\net\base\file_stream_context.cc @ 244]
19 net!net::FileStream::Context::OnOpenCompleted+0x99 [d:\source_code\chromium\depot_tools\src\net\base\file_stream_context.cc @ 204]
1a net!base::internal::FunctorTraits<void (__cdecl net::FileStream::Context::*)(base::Callback<void __cdecl(int),1,1> const & __ptr64,net::FileStream::Context::OpenResult) __ptr64,void>::Invoke<net::FileStream::Context * __ptr64,base::Callback<void __cdecl(int),1,1> const & __ptr64,net::FileStream::Context::OpenResult>+0x78 [d:\source_code\chromium\depot_tools\src\base\bind_internal.h @ 215]
1b net!base::internal::InvokeHelper<0,void>::MakeItSo<void (__cdecl net::FileStream::Context::*const & __ptr64)(base::Callback<void __cdecl(int),1,1> const & __ptr64,net::FileStream::Context::OpenResult) __ptr64,net::FileStream::Context * __ptr64,base::Callback<void __cdecl(int),1,1> const & __ptr64,net::FileStream::Context::OpenResult>+0x6f [d:\source_code\chromium\depot_tools\src\base\bind_internal.h @ 287]
1c net!base::internal::Invoker<base::internal::BindState<void (__cdecl net::FileStream::Context::*)(base::Callback<void __cdecl(int),1,1> const & __ptr64,net::FileStream::Context::OpenResult) __ptr64,base::internal::UnretainedWrapper<net::FileStream::Context>,base::Callback<void __cdecl(int),1,1> >,void __cdecl(net::FileStream::Context::OpenResult)>::RunImpl<void (__cdecl net::FileStream::Context::*const & __ptr64)(base::Callback<void __cdecl(int),1,1> const & __ptr64,net::FileStream::Context::OpenResult) __ptr64,std::tuple<base::internal::UnretainedWrapper<net::FileStream::Context>,base::Callback<void __cdecl(int),1,1> > const & __ptr64,0,1>+0x8c [d:\source_code\chromium\depot_tools\src\base\bind_internal.h @ 365]
1d net!base::internal::Invoker<base::internal::BindState<void (__cdecl net::FileStream::Context::*)(base::Callback<void __cdecl(int),1,1> const & __ptr64,net::FileStream::Context::OpenResult) __ptr64,base::internal::UnretainedWrapper<net::FileStream::Context>,base::Callback<void __cdecl(int),1,1> >,void __cdecl(net::FileStream::Context::OpenResult)>::Run+0x52 [d:\source_code\chromium\depot_tools\src\base\bind_internal.h @ 343]
1e net!base::internal::RunMixin<base::Callback<void __cdecl(net::FileStream::Context::OpenResult),1,1> >::Run+0x71 [d:\source_code\chromium\depot_tools\src\base\callback.h @ 64]
1f net!base::internal::ReplyAdapter<net::FileStream::Context::OpenResult,net::FileStream::Context::OpenResult>+0x5e [d:\source_code\chromium\depot_tools\src\base\task_runner_util.h @ 35]
20 net!base::internal::FunctorTraits<void (__cdecl*)(base::Callback<void __cdecl(net::FileStream::Context::OpenResult),1,1> const & __ptr64,net::FileStream::Context::OpenResult * __ptr64),void>::Invoke<base::Callback<void __cdecl(net::FileStream::Context::OpenResult),1,1> const & __ptr64,net::FileStream::Context::OpenResult * __ptr64>+0x3b [d:\source_code\chromium\depot_tools\src\base\bind_internal.h @ 165]
21 net!base::internal::InvokeHelper<0,void>::MakeItSo<void (__cdecl*const & __ptr64)(base::Callback<void __cdecl(net::FileStream::Context::OpenResult),1,1> const & __ptr64,net::FileStream::Context::OpenResult * __ptr64),base::Callback<void __cdecl(net::FileStream::Context::OpenResult),1,1> const & __ptr64,net::FileStream::Context::OpenResult * __ptr64>+0x53 [d:\source_code\chromium\depot_tools\src\base\bind_internal.h @ 287]
22 net!base::internal::Invoker<base::internal::BindState<void (__cdecl*)(base::Callback<void __cdecl(net::FileStream::Context::OpenResult),1,1> const & __ptr64,net::FileStream::Context::OpenResult * __ptr64),base::Callback<void __cdecl(net::FileStream::Context::OpenResult),1,1>,base::internal::OwnedWrapper<net::FileStream::Context::OpenResult> >,void __cdecl(void)>::RunImpl<void (__cdecl*const & __ptr64)(base::Callback<void __cdecl(net::FileStream::Context::OpenResult),1,1> const & __ptr64,net::FileStream::Context::OpenResult * __ptr64),std::tuple<base::Callback<void __cdecl(net::FileStream::Context::OpenResult),1,1>,base::internal::OwnedWrapper<net::FileStream::Context::OpenResult> > const & __ptr64,0,1>+0x70 [d:\source_code\chromium\depot_tools\src\base\bind_internal.h @ 365]
23 net!base::internal::Invoker<base::internal::BindState<void (__cdecl*)(base::Callback<void __cdecl(net::FileStream::Context::OpenResult),1,1> const & __ptr64,net::FileStream::Context::OpenResult * __ptr64),base::Callback<void __cdecl(net::FileStream::Context::OpenResult),1,1>,base::internal::OwnedWrapper<net::FileStream::Context::OpenResult> >,void __cdecl(void)>::Run+0x33 [d:\source_code\chromium\depot_tools\src\base\bind_internal.h @ 343]
24 base!base::internal::RunMixin<base::Callback<void __cdecl(void),1,1> >::Run+0x54 [d:\source_code\chromium\depot_tools\src\base\callback.h @ 65]
25 base!base::`anonymous namespace'::PostTaskAndReplyRelay::RunReplyAndSelfDestruct+0xf9 [d:\source_code\chromium\depot_tools\src\base\threading\post_task_and_reply_impl.cc @ 64]
26 base!base::internal::FunctorTraits<void (__cdecl base::`anonymous namespace'::PostTaskAndReplyRelay::*)(void) __ptr64,void>::Invoke<base::`anonymous namespace'::PostTaskAndReplyRelay * __ptr64>+0x24 [d:\source_code\chromium\depot_tools\src\base\bind_internal.h @ 215]
27 base!base::internal::InvokeHelper<0,void>::MakeItSo<void (__cdecl base::`anonymous namespace'::PostTaskAndReplyRelay::*const & __ptr64)(void) __ptr64,base::A0x039d942e::PostTaskAndReplyRelay * __ptr64>+0x37 [d:\source_code\chromium\depot_tools\src\base\bind_internal.h @ 287]
28 base!base::internal::Invoker<base::internal::BindState<void (__cdecl base::`anonymous namespace'::PostTaskAndReplyRelay::*)(void) __ptr64,base::internal::UnretainedWrapper<base::`anonymous namespace'::PostTaskAndReplyRelay> >,void __cdecl(void)>::RunImpl<void (__cdecl base::`anonymous namespace'::PostTaskAndReplyRelay::*const & __ptr64)(void) __ptr64,std::tuple<base::internal::UnretainedWrapper<base::`anonymous namespace'::PostTaskAndReplyRelay> > const & __ptr64,0>+0x49 [d:\source_code\chromium\depot_tools\src\base\bind_internal.h @ 365]
29 base!base::internal::Invoker<base::internal::BindState<void (__cdecl base::`anonymous namespace'::PostTaskAndReplyRelay::*)(void) __ptr64,base::internal::UnretainedWrapper<base::`anonymous namespace'::PostTaskAndReplyRelay> >,void __cdecl(void)>::Run+0x33 [d:\source_code\chromium\depot_tools\src\base\bind_internal.h @ 343]
2a base!base::internal::RunMixin<base::Callback<void __cdecl(void),1,1> >::Run+0x54 [d:\source_code\chromium\depot_tools\src\base\callback.h @ 65]
2b base!base::debug::TaskAnnotator::RunTask+0x1f7 [d:\source_code\chromium\depot_tools\src\base\debug\task_annotator.cc @ 56]
2c base!base::MessageLoop::RunTask+0x3b9 [d:\source_code\chromium\depot_tools\src\base\message_loop\message_loop.cc @ 470]
2d base!base::MessageLoop::DeferOrRunPendingTask+0x3c [d:\source_code\chromium\depot_tools\src\base\message_loop\message_loop.cc @ 481]
2e base!base::MessageLoop::DoWork+0x172 [d:\source_code\chromium\depot_tools\src\base\message_loop\message_loop.cc @ 602]
2f base!base::MessagePumpForIO::DoRunLoop+0x27 [d:\source_code\chromium\depot_tools\src\base\message_loop\message_pump_win.cc @ 732]
30 base!base::MessagePumpWin::Run+0x9d [d:\source_code\chromium\depot_tools\src\base\message_loop\message_pump_win.cc @ 143]
31 base!base::MessageLoop::RunHandler+0xf6 [d:\source_code\chromium\depot_tools\src\base\message_loop\message_loop.cc @ 433]
32 base!base::RunLoop::Run+0x3d [d:\source_code\chromium\depot_tools\src\base\run_loop.cc @ 36]
33 base!base::Thread::Run+0x195 [d:\source_code\chromium\depot_tools\src\base\threading\thread.cc @ 242]
34 content!content::BrowserThreadImpl::IOThreadRun+0x2f [d:\source_code\chromium\depot_tools\src\content\browser\browser_thread_impl.cc @ 244]
35 content!content::BrowserThreadImpl::Run+0x20a [d:\source_code\chromium\depot_tools\src\content\browser\browser_thread_impl.cc @ 278]
36 base!base::Thread::ThreadMain+0x556 [d:\source_code\chromium\depot_tools\src\base\threading\thread.cc @ 322]
37 base!base::`anonymous namespace'::ThreadFunc+0x131 [d:\source_code\chromium\depot_tools\src\base\threading\platform_thread_win.cc @ 86]
38 KERNEL32!BaseThreadInitThunk+0x14
39 ntdll!RtlUserThreadStart+0x21

[16436:23504:0513/113652:401403296:VERBOSE1:url_request_job.cc(555)] ReadRawDataComplete() "file:///C:/Users/michael_du/Desktop/2.html" pre bytes read = 77 pre total = 77 post total = 77
[16436:23504:0513/113652:401403296:VERBOSE1:resource_loader.cc(360)] OnReadCompleted: "file:///C:/Users/michael_du/Desktop/2.html" bytes_read = 77
[16436:23504:0513/113652:401403312:VERBOSE1:resource_loader.cc(360)] OnReadCompleted: "file:///C:/Users/michael_du/Desktop/2.html" bytes_read = 0
[16436:23504:0513/113652:401403312:VERBOSE1:resource_loader.cc(648)] ResponseCompleted: file:///C:/Users/michael_du/Desktop/2.html
[16436:14576:0513/113652:401403375:VERBOSE1:navigation_controller_impl.cc(852)] Navigation finished at (smoothed) timestamp 13139120212690660
[24072:2920:0513/113652:401403375:VERBOSE1:histogram.cc(377)] Histogram: PreloadScanner.DocumentWrite.ScriptLength has bad minimum: 0
[24072:2920:0513/113652:401403375:VERBOSE1:htmltreebuilder.cpp(2507)] Not implemented.
[24072:2920:0513/113652:401403390:VERBOSE1:histogram.cc(377)] Histogram: V8.CompileNoncacheableMicroSeconds has bad minimum: 0
[24072:2920:0513/113652:401403390:VERBOSE1:htmltreebuilder.cpp(1449)] Not implmeneted.
[16436:14576:0513/113652:401403437:VERBOSE1:navigation_controller_impl.cc(852)] Navigation finished at (smoothed) timestamp 13139120212759688
[24072:2920:0513/113652:401403437:VERBOSE1:htmltreebuilder.cpp(2507)] Not implemented.
[24072:2920:0513/113652:401403437:VERBOSE1:htmltreebuilder.cpp(2448)] Not implemented.
[16436:23504:0513/113652:401403453:VERBOSE1:media_stream_dispatcher_host.cc(144)] MediaStreamDispatcherHost::OnChannelClosing

```

从code中可以看出，ContinuePendingBeginRequest会创建一个ResourceHandler，然后，用这个handler开始一个新的request。

```C
void ResourceDispatcherHostImpl::ContinuePendingBeginRequest(
    int request_id,
    const ResourceRequest& request_data,
    IPC::Message* sync_result,  // only valid for sync
    int route_id,
    const net::HttpRequestHeaders& headers,
    mojo::InterfaceRequest<mojom::URLLoader> mojo_request,
    mojom::URLLoaderClientPtr url_loader_client,
    bool continue_request,
    int error_code) {
  ....
  std::unique_ptr<ResourceHandler> handler(CreateResourceHandler(
      new_request.get(), request_data, sync_result, route_id, process_type,
      child_id, resource_context, std::move(mojo_request),
      std::move(url_loader_client)));

  if (handler)
    BeginRequestInternal(std::move(new_request), std::move(handler));
}
```

在WillStartRequest中，会通过task来执行NavigationResourceThrottle::OnUIChecksPerformed，其原因是保证UI的相应，如果用户取消了这次访问，便可以直接取消。否则，就继续执行文件加载的动作。

```C
NavigationResourceThrottle::NavigationResourceThrottle(
    net::URLRequest* request,
    ResourceDispatcherHostDelegate* resource_dispatcher_host_delegate,
    RequestContextType request_context_type)
    : request_(request),
      resource_dispatcher_host_delegate_(resource_dispatcher_host_delegate),
      request_context_type_(request_context_type),
      weak_ptr_factory_(this) {}

NavigationResourceThrottle::~NavigationResourceThrottle() {}

void NavigationResourceThrottle::WillStartRequest(bool* defer) {
  DCHECK_CURRENTLY_ON(BrowserThread::IO);
  const ResourceRequestInfoImpl* info =
      ResourceRequestInfoImpl::ForRequest(request_);
  if (!info)
    return;

  int render_process_id, render_frame_id;
  if (!info->GetAssociatedRenderFrame(&render_process_id, &render_frame_id))
    return;

  bool is_external_protocol =
      !info->GetContext()->GetRequestContext()->job_factory()->IsHandledURL(
          request_->url());
  UIChecksPerformedCallback callback =
      base::Bind(&NavigationResourceThrottle::OnUIChecksPerformed,
                 weak_ptr_factory_.GetWeakPtr());
  DCHECK(request_->method() == "POST" || request_->method() == "GET");
  BrowserThread::PostTask(
      BrowserThread::UI, FROM_HERE,
      base::Bind(&CheckWillStartRequestOnUIThread, callback, render_process_id,
                 render_frame_id, request_->method(), info->body(),
                 Referrer::SanitizeForRequest(
                     request_->url(), Referrer(GURL(request_->referrer()),
                                               info->GetReferrerPolicy())),
                 info->HasUserGesture(), info->GetPageTransition(),
                 is_external_protocol, request_context_type_));
  *defer = true;
}

```

这里的本地文件访问是通过URLRequestFileJob来完成，把本地文件访问通过“file://”协议处理，在内部实现，通过工厂模式生产出相应的URLRequestJob。

```
 # Call Site
00 net!net::URLRequestFileJob::Start [d:\source_code\chromium\depot_tools\src\net\url_request\url_request_file_job.cc @ 68]
01 net!net::URLRequest::StartJob+0x791 [d:\source_code\chromium\depot_tools\src\net\url_request\url_request.cc @ 679]
02 net!net::URLRequest::BeforeRequestComplete+0x4a7 [d:\source_code\chromium\depot_tools\src\net\url_request\url_request.cc @ 624]
03 net!net::URLRequest::Start+0x410 [d:\source_code\chromium\depot_tools\src\net\url_request\url_request.cc @ 547]
04 content!content::ResourceLoader::StartRequestInternal+0x139 [d:\source_code\chromium\depot_tools\src\content\browser\loader\resource_loader.cc @ 485]
05 content!content::ResourceLoader::Resume+0x1ba [d:\source_code\chromium\depot_tools\src\content\browser\loader\resource_loader.cc @ 448]
06 content!content::MimeSniffingResourceHandler::Resume+0xe6 [d:\source_code\chromium\depot_tools\src\content\browser\loader\mime_sniffing_resource_handler.cc @ 242]
07 content!content::ThrottlingResourceHandler::ResumeStart+0x1ac [d:\source_code\chromium\depot_tools\src\content\browser\loader\throttling_resource_handler.cc @ 159]
08 content!content::ThrottlingResourceHandler::Resume+0x1d9 [d:\source_code\chromium\depot_tools\src\content\browser\loader\throttling_resource_handler.cc @ 137]
09 content!content::NavigationResourceThrottle::OnUIChecksPerformed+0x1e3 [d:\source_code\chromium\depot_tools\src\content\browser\loader\navigation_resource_throttle.cc @ 289]
0a content!base::internal::FunctorTraits<void (__cdecl content::NavigationResourceThrottle::*)(enum content::NavigationThrottle::ThrottleCheckResult) __ptr64,void>::Invoke<base::WeakPtr<content::NavigationResourceThrottle> const & __ptr64,enum content::NavigationThrottle::ThrottleCheckResult>+0x37 [d:\source_code\chromium\depot_tools\src\base\bind_internal.h @ 215]
0b content!base::internal::InvokeHelper<1,void>::MakeItSo<void (__cdecl content::NavigationResourceThrottle::*const & __ptr64)(enum content::NavigationThrottle::ThrottleCheckResult) __ptr64,base::WeakPtr<content::NavigationResourceThrottle> const & __ptr64,enum content::NavigationThrottle::ThrottleCheckResult>+0x66 [d:\source_code\chromium\depot_tools\src\base\bind_internal.h @ 308]
0c content!base::internal::Invoker<base::internal::BindState<void (__cdecl content::NavigationResourceThrottle::*)(enum content::NavigationThrottle::ThrottleCheckResult) __ptr64,base::WeakPtr<content::NavigationResourceThrottle> >,void __cdecl(enum content::NavigationThrottle::ThrottleCheckResult)>::RunImpl<void (__cdecl content::NavigationResourceThrottle::*const & __ptr64)(enum content::NavigationThrottle::ThrottleCheckResult) __ptr64,std::tuple<base::WeakPtr<content::NavigationResourceThrottle> > const & __ptr64,0>+0x68 [d:\source_code\chromium\depot_tools\src\base\bind_internal.h @ 365]
0d content!base::internal::Invoker<base::internal::BindState<void (__cdecl content::NavigationResourceThrottle::*)(enum content::NavigationThrottle::ThrottleCheckResult) __ptr64,base::WeakPtr<content::NavigationResourceThrottle> >,void __cdecl(enum content::NavigationThrottle::ThrottleCheckResult)>::Run+0x52 [d:\source_code\chromium\depot_tools\src\base\bind_internal.h @ 343]
0e content!base::internal::RunMixin<base::Callback<void __cdecl(enum content::NavigationThrottle::ThrottleCheckResult),1,1> >::Run+0x70 [d:\source_code\chromium\depot_tools\src\base\callback.h @ 65]
0f content!base::internal::FunctorTraits<base::Callback<void __cdecl(enum content::NavigationThrottle::ThrottleCheckResult),1,1>,void>::Invoke<base::Callback<void __cdecl(enum content::NavigationThrottle::ThrottleCheckResult),1,1> const & __ptr64,enum content::NavigationThrottle::ThrottleCheckResult const & __ptr64>+0x109 [d:\source_code\chromium\depot_tools\src\base\bind_internal.h @ 266]
10 content!base::internal::InvokeHelper<0,void>::MakeItSo<base::Callback<void __cdecl(enum content::NavigationThrottle::ThrottleCheckResult),1,1> const & __ptr64,enum content::NavigationThrottle::ThrottleCheckResult const & __ptr64>+0x37 [d:\source_code\chromium\depot_tools\src\base\bind_internal.h @ 287]
11 content!base::internal::Invoker<base::internal::BindState<base::Callback<void __cdecl(enum content::NavigationThrottle::ThrottleCheckResult),1,1>,enum content::NavigationThrottle::ThrottleCheckResult>,void __cdecl(void)>::RunImpl<base::Callback<void __cdecl(enum content::NavigationThrottle::ThrottleCheckResult),1,1> const & __ptr64,std::tuple<enum content::NavigationThrottle::ThrottleCheckResult> const & __ptr64,0>+0x4c [d:\source_code\chromium\depot_tools\src\base\bind_internal.h @ 365]
12 content!base::internal::Invoker<base::internal::BindState<base::Callback<void __cdecl(enum content::NavigationThrottle::ThrottleCheckResult),1,1>,enum content::NavigationThrottle::ThrottleCheckResult>,void __cdecl(void)>::Run+0x33 [d:\source_code\chromium\depot_tools\src\base\bind_internal.h @ 343]
13 base!base::internal::RunMixin<base::Callback<void __cdecl(void),1,1> >::Run+0x54 [d:\source_code\chromium\depot_tools\src\base\callback.h @ 65]
14 base!base::debug::TaskAnnotator::RunTask+0x1f7 [d:\source_code\chromium\depot_tools\src\base\debug\task_annotator.cc @ 56]
15 base!base::MessageLoop::RunTask+0x3b9 [d:\source_code\chromium\depot_tools\src\base\message_loop\message_loop.cc @ 470]
16 base!base::MessageLoop::DeferOrRunPendingTask+0x3c [d:\source_code\chromium\depot_tools\src\base\message_loop\message_loop.cc @ 481]
17 base!base::MessageLoop::DoWork+0x172 [d:\source_code\chromium\depot_tools\src\base\message_loop\message_loop.cc @ 602]
18 base!base::MessagePumpForIO::DoRunLoop+0x27 [d:\source_code\chromium\depot_tools\src\base\message_loop\message_pump_win.cc @ 732]
19 base!base::MessagePumpWin::Run+0x9d [d:\source_code\chromium\depot_tools\src\base\message_loop\message_pump_win.cc @ 143]
1a base!base::MessageLoop::RunHandler+0xf6 [d:\source_code\chromium\depot_tools\src\base\message_loop\message_loop.cc @ 433]
1b base!base::RunLoop::Run+0x3d [d:\source_code\chromium\depot_tools\src\base\run_loop.cc @ 36]
1c base!base::Thread::Run+0x195 [d:\source_code\chromium\depot_tools\src\base\threading\thread.cc @ 242]
1d content!content::BrowserThreadImpl::IOThreadRun+0x2f [d:\source_code\chromium\depot_tools\src\content\browser\browser_thread_impl.cc @ 244]
1e content!content::BrowserThreadImpl::Run+0x20a [d:\source_code\chromium\depot_tools\src\content\browser\browser_thread_impl.cc @ 278]
1f base!base::Thread::ThreadMain+0x556 [d:\source_code\chromium\depot_tools\src\base\threading\thread.cc @ 322]
20 base!base::`anonymous namespace'::ThreadFunc+0x131 [d:\source_code\chromium\depot_tools\src\base\threading\platform_thread_win.cc @ 86]
21 KERNEL32!BaseThreadInitThunk+0x14
22 ntdll!RtlUserThreadStart+0x21
```

### 第五步

URLRequestFileJob读取本地文件以后，如何将内容传给Render Process？
在读取完成以后，会触发URLRequestFileJob::DidRead函数，最后，会调用AsyncResourceHandler::OnReadCompleted，给render process发送ResourceMsg_DataReceived MSG

```
 # Call Site
00 content!content::AsyncResourceHandler::OnReadCompleted [d:\source_code\chromium\depot_tools\src\content\browser\loader\async_resource_handler.cc @ 435]
01 content!content::CrossSiteResourceHandler::OnReadCompleted+0xce [d:\source_code\chromium\depot_tools\src\content\browser\loader\cross_site_resource_handler.cc @ 238]
02 content!content::InterceptingResourceHandler::OnReadCompleted+0x122 [d:\source_code\chromium\depot_tools\src\content\browser\loader\intercepting_resource_handler.cc @ 92]
03 content!content::LayeredResourceHandler::OnReadCompleted+0x117 [d:\source_code\chromium\depot_tools\src\content\browser\loader\layered_resource_handler.cc @ 62]
04 content!content::MimeSniffingResourceHandler::OnReadCompleted+0x6d [d:\source_code\chromium\depot_tools\src\content\browser\loader\mime_sniffing_resource_handler.cc @ 197]
05 content!content::LayeredResourceHandler::OnReadCompleted+0x117 [d:\source_code\chromium\depot_tools\src\content\browser\loader\layered_resource_handler.cc @ 62]
06 content!content::ResourceLoader::CompleteRead+0x326 [d:\source_code\chromium\depot_tools\src\content\browser\loader\resource_loader.cc @ 631]
07 content!content::ResourceLoader::OnReadCompleted+0x320 [d:\source_code\chromium\depot_tools\src\content\browser\loader\resource_loader.cc @ 378]
08 net!net::URLRequest::NotifyReadCompleted+0xa8 [d:\source_code\chromium\depot_tools\src\net\url_request\url_request.cc @ 1174]
09 net!net::URLRequestJob::ReadRawDataComplete+0x667 [d:\source_code\chromium\depot_tools\src\net\url_request\url_request_job.cc @ 571]
0a net!net::URLRequestFileJob::DidRead+0x15a [d:\source_code\chromium\depot_tools\src\net\url_request\url_request_file_job.cc @ 300]
0b net!base::internal::FunctorTraits<void (__cdecl net::URLRequestFileJob::*)(scoped_refptr<net::IOBuffer>,int) __ptr64,void>::Invoke<base::WeakPtr<net::URLRequestFileJob> const & __ptr64,scoped_refptr<net::IOBuffer> const & __ptr64,int>+0x6b [d:\source_code\chromium\depot_tools\src\base\bind_internal.h @ 215]
0c net!base::internal::InvokeHelper<1,void>::MakeItSo<void (__cdecl net::URLRequestFileJob::*const & __ptr64)(scoped_refptr<net::IOBuffer>,int) __ptr64,base::WeakPtr<net::URLRequestFileJob> const & __ptr64,scoped_refptr<net::IOBuffer> const & __ptr64,int>+0x82 [d:\source_code\chromium\depot_tools\src\base\bind_internal.h @ 308]
0d net!base::internal::Invoker<base::internal::BindState<void (__cdecl net::URLRequestFileJob::*)(scoped_refptr<net::IOBuffer>,int) __ptr64,base::WeakPtr<net::URLRequestFileJob>,scoped_refptr<net::IOBuffer> >,void __cdecl(int)>::RunImpl<void (__cdecl net::URLRequestFileJob::*const & __ptr64)(scoped_refptr<net::IOBuffer>,int) __ptr64,std::tuple<base::WeakPtr<net::URLRequestFileJob>,scoped_refptr<net::IOBuffer> > const & __ptr64,0,1>+0x8f [d:\source_code\chromium\depot_tools\src\base\bind_internal.h @ 365]
0e net!base::internal::Invoker<base::internal::BindState<void (__cdecl net::URLRequestFileJob::*)(scoped_refptr<net::IOBuffer>,int) __ptr64,base::WeakPtr<net::URLRequestFileJob>,scoped_refptr<net::IOBuffer> >,void __cdecl(int)>::Run+0x52 [d:\source_code\chromium\depot_tools\src\base\bind_internal.h @ 343]
0f net!base::internal::RunMixin<base::Callback<void __cdecl(int),1,1> >::Run+0x70 [d:\source_code\chromium\depot_tools\src\base\callback.h @ 65]
10 net!net::FileStream::Context::InvokeUserCallback+0xd8 [d:\source_code\chromium\depot_tools\src\net\base\file_stream_context_win.cc @ 188]
11 net!net::FileStream::Context::OnIOCompleted+0x456 [d:\source_code\chromium\depot_tools\src\net\base\file_stream_context_win.cc @ 169]
12 base!base::MessagePumpForIO::WaitForIOCompletion+0xc8 [d:\source_code\chromium\depot_tools\src\base\message_loop\message_pump_win.cc @ 791]
13 base!base::MessagePumpForIO::DoRunLoop+0x59 [d:\source_code\chromium\depot_tools\src\base\message_loop\message_pump_win.cc @ 736]
14 base!base::MessagePumpWin::Run+0x9d [d:\source_code\chromium\depot_tools\src\base\message_loop\message_pump_win.cc @ 143]
15 base!base::MessageLoop::RunHandler+0xf6 [d:\source_code\chromium\depot_tools\src\base\message_loop\message_loop.cc @ 433]
16 base!base::RunLoop::Run+0x3d [d:\source_code\chromium\depot_tools\src\base\run_loop.cc @ 36]
17 base!base::Thread::Run+0x195 [d:\source_code\chromium\depot_tools\src\base\threading\thread.cc @ 242]
18 content!content::BrowserThreadImpl::IOThreadRun+0x2f [d:\source_code\chromium\depot_tools\src\content\browser\browser_thread_impl.cc @ 244]
19 content!content::BrowserThreadImpl::Run+0x20a [d:\source_code\chromium\depot_tools\src\content\browser\browser_thread_impl.cc @ 278]
1a base!base::Thread::ThreadMain+0x556 [d:\source_code\chromium\depot_tools\src\base\threading\thread.cc @ 322]
1b base!base::`anonymous namespace'::ThreadFunc+0x131 [d:\source_code\chromium\depot_tools\src\base\threading\platform_thread_win.cc @ 86]
1c KERNEL32!BaseThreadInitThunk+0x14
1d ntdll!RtlUserThreadStart+0x21
```

render process相应的MSG Handler是ResourceDispatcher::OnReceivedData，收到data以后，会
其中，ReceiveData是通过SharedMemoryReceivedDataFactory.Create通过读取共享内存实现获取的。

```
 # Call Site
00 content!content::RenderFrameImpl::SendDidCommitProvisionalLoad [d:\source_code\chromium\depot_tools\src\content\renderer\render_frame_impl.cc @ 4573]
01 content!content::RenderFrameImpl::didCommitProvisionalLoad+0x10a1 [d:\source_code\chromium\depot_tools\src\content\renderer\render_frame_impl.cc @ 3502]
02 blink_web!blink::FrameLoaderClientImpl::dispatchDidCommitLoad+0x23b [d:\source_code\chromium\depot_tools\src\third_party\webkit\source\web\frameloaderclientimpl.cpp @ 478]
03 blink_core!blink::FrameLoader::receivedFirstData+0x18e [d:\source_code\chromium\depot_tools\src\third_party\webkit\source\core\loader\frameloader.cpp @ 453]
04 blink_core!blink::DocumentLoader::createWriterFor+0x297 [d:\source_code\chromium\depot_tools\src\third_party\webkit\source\core\loader\documentloader.cpp @ 706]
05 blink_core!blink::DocumentLoader::ensureWriter+0x2f0 [d:\source_code\chromium\depot_tools\src\third_party\webkit\source\core\loader\documentloader.cpp @ 484]
06 blink_core!blink::DocumentLoader::commitData+0x105 [d:\source_code\chromium\depot_tools\src\third_party\webkit\source\core\loader\documentloader.cpp @ 492]
07 blink_core!blink::DocumentLoader::processData+0xaa [d:\source_code\chromium\depot_tools\src\third_party\webkit\source\core\loader\documentloader.cpp @ 555]
08 blink_core!blink::DocumentLoader::dataReceived+0x485 [d:\source_code\chromium\depot_tools\src\third_party\webkit\source\core\loader\documentloader.cpp @ 532]
09 blink_core!blink::RawResource::appendData+0x74 [d:\source_code\chromium\depot_tools\src\third_party\webkit\source\core\fetch\rawresource.cpp @ 98]
0a blink_core!blink::ResourceLoader::didReceiveData+0x172 [d:\source_code\chromium\depot_tools\src\third_party\webkit\source\core\fetch\resourceloader.cpp @ 172]
0b content!content::WebURLLoaderImpl::Context::OnReceivedData+0x26b [d:\source_code\chromium\depot_tools\src\content\child\web_url_loader_impl.cc @ 776]
0c content!content::WebURLLoaderImpl::RequestPeerImpl::OnReceivedData+0x52 [d:\source_code\chromium\depot_tools\src\content\child\web_url_loader_impl.cc @ 951]
0d content!content::ResourceDispatcher::OnReceivedData+0x54d [d:\source_code\chromium\depot_tools\src\content\child\resource_dispatcher.cc @ 344]
0e content!base::DispatchToMethodImpl<content::ResourceDispatcher * __ptr64,void (__cdecl content::ResourceDispatcher::*)(int,int,int,int,int) __ptr64,std::tuple<int,int,int,int,int> const & __ptr64,0,1,2,3,4>+0xba [d:\source_code\chromium\depot_tools\src\base\tuple.h @ 145]
0f content!base::DispatchToMethod<content::ResourceDispatcher * __ptr64,void (__cdecl content::ResourceDispatcher::*)(int,int,int,int,int) __ptr64,std::tuple<int,int,int,int,int> const & __ptr64>+0x35 [d:\source_code\chromium\depot_tools\src\base\tuple.h @ 153]
10 content!IPC::DispatchToMethod<content::ResourceDispatcher,void (__cdecl content::ResourceDispatcher::*)(int,int,int,int,int) __ptr64,void,std::tuple<int,int,int,int,int> >+0x2c [d:\source_code\chromium\depot_tools\src\ipc\ipc_message_templates.h @ 27]
11 content!IPC::MessageT<ResourceMsg_DataReceived_Meta,std::tuple<int,int,int,int,int>,void>::Dispatch<content::ResourceDispatcher,content::ResourceDispatcher,void,void (__cdecl content::ResourceDispatcher::*)(int,int,int,int,int) __ptr64>+0x111 [d:\source_code\chromium\depot_tools\src\ipc\ipc_message_templates.h @ 122]
12 content!content::ResourceDispatcher::DispatchMessageW+0x44e [d:\source_code\chromium\depot_tools\src\content\child\resource_dispatcher.cc @ 567]
13 content!content::ResourceDispatcher::OnMessageReceived+0x343 [d:\source_code\chromium\depot_tools\src\content\child\resource_dispatcher.cc @ 183]
14 content!content::ResourceSchedulingFilter::DispatchMessageW+0x2a [d:\source_code\chromium\depot_tools\src\content\child\resource_scheduling_filter.cc @ 75]
15 content!base::internal::FunctorTraits<void (__cdecl content::ResourceSchedulingFilter::*)(IPC::Message const & __ptr64) __ptr64,void>::Invoke<base::WeakPtr<content::ResourceSchedulingFilter> const & __ptr64,IPC::Message const & __ptr64>+0x4a [d:\source_code\chromium\depot_tools\src\base\bind_internal.h @ 215]
16 content!base::internal::InvokeHelper<1,void>::MakeItSo<void (__cdecl content::ResourceSchedulingFilter::*const & __ptr64)(IPC::Message const & __ptr64) __ptr64,base::WeakPtr<content::ResourceSchedulingFilter> const & __ptr64,IPC::Message const & __ptr64>+0x7c [d:\source_code\chromium\depot_tools\src\base\bind_internal.h @ 308]
17 content!base::internal::Invoker<base::internal::BindState<void (__cdecl content::ResourceSchedulingFilter::*)(IPC::Message const & __ptr64) __ptr64,base::WeakPtr<content::ResourceSchedulingFilter>,IPC::Message>,void __cdecl(void)>::RunImpl<void (__cdecl content::ResourceSchedulingFilter::*const & __ptr64)(IPC::Message const & __ptr64) __ptr64,std::tuple<base::WeakPtr<content::ResourceSchedulingFilter>,IPC::Message> const & __ptr64,0,1>+0x73 [d:\source_code\chromium\depot_tools\src\base\bind_internal.h @ 365]
18 content!base::internal::Invoker<base::internal::BindState<void (__cdecl content::ResourceSchedulingFilter::*)(IPC::Message const & __ptr64) __ptr64,base::WeakPtr<content::ResourceSchedulingFilter>,IPC::Message>,void __cdecl(void)>::Run+0x33 [d:\source_code\chromium\depot_tools\src\base\bind_internal.h @ 343]
19 base!base::internal::RunMixin<base::Callback<void __cdecl(void),1,1> >::Run+0x54 [d:\source_code\chromium\depot_tools\src\base\callback.h @ 65]
1a base!base::debug::TaskAnnotator::RunTask+0x1f7 [d:\source_code\chromium\depot_tools\src\base\debug\task_annotator.cc @ 56]
1b blink_platform!blink::scheduler::TaskQueueManager::ProcessTaskFromWorkQueue+0x606 [d:\source_code\chromium\depot_tools\src\third_party\webkit\source\platform\scheduler\base\task_queue_manager.cc @ 340]
1c blink_platform!blink::scheduler::TaskQueueManager::DoWork+0x411 [d:\source_code\chromium\depot_tools\src\third_party\webkit\source\platform\scheduler\base\task_queue_manager.cc @ 234]
1d blink_platform!base::internal::FunctorTraits<void (__cdecl blink::scheduler::TaskQueueManager::*)(base::TimeTicks,bool) __ptr64,void>::Invoke<base::WeakPtr<blink::scheduler::TaskQueueManager> const & __ptr64,base::TimeTicks const & __ptr64,bool const & __ptr64>+0x67 [d:\source_code\chromium\depot_tools\src\base\bind_internal.h @ 215]
1e blink_platform!base::internal::InvokeHelper<1,void>::MakeItSo<void (__cdecl blink::scheduler::TaskQueueManager::*const & __ptr64)(base::TimeTicks,bool) __ptr64,base::WeakPtr<blink::scheduler::TaskQueueManager> const & __ptr64,base::TimeTicks const & __ptr64,bool const & __ptr64>+0x9e [d:\source_code\chromium\depot_tools\src\base\bind_internal.h @ 308]
1f blink_platform!base::internal::Invoker<base::internal::BindState<void (__cdecl blink::scheduler::TaskQueueManager::*)(base::TimeTicks,bool) __ptr64,base::WeakPtr<blink::scheduler::TaskQueueManager>,base::TimeTicks,bool>,void __cdecl(void)>::RunImpl<void (__cdecl blink::scheduler::TaskQueueManager::*const & __ptr64)(base::TimeTicks,bool) __ptr64,std::tuple<base::WeakPtr<blink::scheduler::TaskQueueManager>,base::TimeTicks,bool> const & __ptr64,0,1,2>+0x9a [d:\source_code\chromium\depot_tools\src\base\bind_internal.h @ 365]
20 blink_platform!base::internal::Invoker<base::internal::BindState<void (__cdecl blink::scheduler::TaskQueueManager::*)(base::TimeTicks,bool) __ptr64,base::WeakPtr<blink::scheduler::TaskQueueManager>,base::TimeTicks,bool>,void __cdecl(void)>::Run+0x33 [d:\source_code\chromium\depot_tools\src\base\bind_internal.h @ 343]
21 base!base::internal::RunMixin<base::Callback<void __cdecl(void),1,1> >::Run+0x54 [d:\source_code\chromium\depot_tools\src\base\callback.h @ 65]
22 base!base::debug::TaskAnnotator::RunTask+0x1f7 [d:\source_code\chromium\depot_tools\src\base\debug\task_annotator.cc @ 56]
23 base!base::MessageLoop::RunTask+0x3b9 [d:\source_code\chromium\depot_tools\src\base\message_loop\message_loop.cc @ 470]
24 base!base::MessageLoop::DeferOrRunPendingTask+0x3c [d:\source_code\chromium\depot_tools\src\base\message_loop\message_loop.cc @ 481]
25 base!base::MessageLoop::DoWork+0x172 [d:\source_code\chromium\depot_tools\src\base\message_loop\message_loop.cc @ 602]
26 base!base::MessagePumpDefault::Run+0xfa [d:\source_code\chromium\depot_tools\src\base\message_loop\message_pump_default.cc @ 35]
27 base!base::MessageLoop::RunHandler+0xf6 [d:\source_code\chromium\depot_tools\src\base\message_loop\message_loop.cc @ 433]
28 base!base::RunLoop::Run+0x3d [d:\source_code\chromium\depot_tools\src\base\run_loop.cc @ 36]
29 content!content::RendererMain+0x3ca [d:\source_code\chromium\depot_tools\src\content\renderer\renderer_main.cc @ 198]
2a content!content::RunNamedProcessTypeMain+0xd4 [d:\source_code\chromium\depot_tools\src\content\app\content_main_runner.cc @ 418]
2b content!content::ContentMainRunnerImpl::Run+0x278 [d:\source_code\chromium\depot_tools\src\content\app\content_main_runner.cc @ 786]
2c content!content::ContentMain+0x81 [d:\source_code\chromium\depot_tools\src\content\app\content_main.cc @ 20]
2d content_shell!wWinMain+0x7b [d:\source_code\chromium\depot_tools\src\content\shell\app\shell_main.cc @ 33]
2e content_shell!invoke_main+0x2d [f:\dd\vctools\crt\vcstartup\src\startup\exe_common.inl @ 118]
2f content_shell!__scrt_common_main_seh+0x127 [f:\dd\vctools\crt\vcstartup\src\startup\exe_common.inl @ 253]
30 content_shell!__scrt_common_main+0xe [f:\dd\vctools\crt\vcstartup\src\startup\exe_common.inl @ 296]
31 content_shell!wWinMainCRTStartup+0x9 [f:\dd\vctools\crt\vcstartup\src\startup\exe_wwinmain.cpp @ 17]
32 KERNEL32!BaseThreadInitThunk+0x14
33 ntdll!RtlUserThreadStart+0x21
```


### 

在render process中，RenderFrameImpl::RunJavaScriptMessage会给browser process发送FrameHostMsg_RunJavaScriptMessage



### 

在FrameHostMsg_RunJavaScriptMessage的响应函数OnRunJavaScriptMessage中，browser通过调用CreateDialogParamW来显示alert

```
 # Call Site
00 USER32!CreateDialogParamW
01 content_shell!content::ShellJavaScriptDialog::ShellJavaScriptDialog+0xd4 [d:\source_code\chromium\depot_tools\src\content\shell\browser\shell_javascript_dialog_win.cc @ 98]
02 content_shell!content::ShellJavaScriptDialogManager::RunJavaScriptDialog+0x20e [d:\source_code\chromium\depot_tools\src\content\shell\browser\shell_javascript_dialog_manager.cc @ 53]
03 content!content::WebContentsImpl::RunJavaScriptMessage+0x308 [d:\source_code\chromium\depot_tools\src\content\browser\web_contents\web_contents_impl.cc @ 4123]
04 content!content::RenderFrameHostImpl::OnRunJavaScriptMessage+0x24b [d:\source_code\chromium\depot_tools\src\content\browser\frame_host\render_frame_host_impl.cc @ 1617]
05 content!base::DispatchToMethodImpl<content::RenderFrameHostImpl * __ptr64,void (__cdecl content::RenderFrameHostImpl::*)(std::basic_string<wchar_t,std::char_traits<wchar_t>,std::allocator<wchar_t> > const & __ptr64,std::basic_string<wchar_t,std::char_traits<wchar_t>,std::allocator<wchar_t> > const & __ptr64,GURL const & __ptr64,enum content::JavaScriptMessageType,IPC::Message * __ptr64) __ptr64,std::tuple<std::basic_string<wchar_t,std::char_traits<wchar_t>,std::allocator<wchar_t> >,std::basic_string<wchar_t,std::char_traits<wchar_t>,std::allocator<wchar_t> >,GURL,enum content::JavaScriptMessageType> & __ptr64,std::tuple<IPC::Message & __ptr64>,0,1,2,3,0>+0xbe [d:\source_code\chromium\depot_tools\src\base\tuple.h @ 186]
06 content!base::DispatchToMethod<content::RenderFrameHostImpl * __ptr64,void (__cdecl content::RenderFrameHostImpl::*)(std::basic_string<wchar_t,std::char_traits<wchar_t>,std::allocator<wchar_t> > const & __ptr64,std::basic_string<wchar_t,std::char_traits<wchar_t>,std::allocator<wchar_t> > const & __ptr64,GURL const & __ptr64,enum content::JavaScriptMessageType,IPC::Message * __ptr64) __ptr64,std::tuple<std::basic_string<wchar_t,std::char_traits<wchar_t>,std::allocator<wchar_t> >,std::basic_string<wchar_t,std::char_traits<wchar_t>,std::allocator<wchar_t> >,GURL,enum content::JavaScriptMessageType> & __ptr64,std::tuple<IPC::Message & __ptr64> >+0x67 [d:\source_code\chromium\depot_tools\src\base\tuple.h @ 196]
07 content!IPC::MessageT<FrameHostMsg_RunJavaScriptMessage_Meta,std::tuple<std::basic_string<wchar_t,std::char_traits<wchar_t>,std::allocator<wchar_t> >,std::basic_string<wchar_t,std::char_traits<wchar_t>,std::allocator<wchar_t> >,GURL,enum content::JavaScriptMessageType>,std::tuple<bool,std::basic_string<wchar_t,std::char_traits<wchar_t>,std::allocator<wchar_t> > > >::DispatchDelayReply<content::RenderFrameHostImpl,void,void (__cdecl content::RenderFrameHostImpl::*)(std::basic_string<wchar_t,std::char_traits<wchar_t>,std::allocator<wchar_t> > const & __ptr64,std::basic_string<wchar_t,std::char_traits<wchar_t>,std::allocator<wchar_t> > const & __ptr64,GURL const & __ptr64,enum content::JavaScriptMessageType,IPC::Message * __ptr64) __ptr64>+0x18d [d:\source_code\chromium\depot_tools\src\ipc\ipc_message_templates.h @ 197]
08 content!content::RenderFrameHostImpl::OnMessageReceived+0xf86 [d:\source_code\chromium\depot_tools\src\content\browser\frame_host\render_frame_host_impl.cc @ 618]
09 content!content::RenderProcessHostImpl::OnMessageReceived+0x6a7 [d:\source_code\chromium\depot_tools\src\content\browser\renderer_host\render_process_host_impl.cc @ 1988]
0a ipc!IPC::ChannelProxy::Context::OnDispatchMessage+0x92 [d:\source_code\chromium\depot_tools\src\ipc\ipc_channel_proxy.cc @ 340]
0b ipc!base::internal::FunctorTraits<void (__cdecl IPC::ChannelProxy::Context::*)(IPC::Message const & __ptr64) __ptr64,void>::Invoke<scoped_refptr<IPC::ChannelProxy::Context> const & __ptr64,IPC::Message const & __ptr64>+0x4a [d:\source_code\chromium\depot_tools\src\base\bind_internal.h @ 215]
0c ipc!base::internal::InvokeHelper<0,void>::MakeItSo<void (__cdecl IPC::ChannelProxy::Context::*const & __ptr64)(IPC::Message const & __ptr64) __ptr64,scoped_refptr<IPC::ChannelProxy::Context> const & __ptr64,IPC::Message const & __ptr64>+0x69 [d:\source_code\chromium\depot_tools\src\base\bind_internal.h @ 287]
0d ipc!base::internal::Invoker<base::internal::BindState<void (__cdecl IPC::ChannelProxy::Context::*)(IPC::Message const & __ptr64) __ptr64,scoped_refptr<IPC::ChannelProxy::Context>,IPC::Message>,void __cdecl(void)>::RunImpl<void (__cdecl IPC::ChannelProxy::Context::*const & __ptr64)(IPC::Message const & __ptr64) __ptr64,std::tuple<scoped_refptr<IPC::ChannelProxy::Context>,IPC::Message> const & __ptr64,0,1>+0x73 [d:\source_code\chromium\depot_tools\src\base\bind_internal.h @ 365]
0e ipc!base::internal::Invoker<base::internal::BindState<void (__cdecl IPC::ChannelProxy::Context::*)(IPC::Message const & __ptr64) __ptr64,scoped_refptr<IPC::ChannelProxy::Context>,IPC::Message>,void __cdecl(void)>::Run+0x33 [d:\source_code\chromium\depot_tools\src\base\bind_internal.h @ 343]
0f base!base::internal::RunMixin<base::Callback<void __cdecl(void),1,1> >::Run+0x54 [d:\source_code\chromium\depot_tools\src\base\callback.h @ 65]
10 base!base::debug::TaskAnnotator::RunTask+0x1f7 [d:\source_code\chromium\depot_tools\src\base\debug\task_annotator.cc @ 56]
11 base!base::MessageLoop::RunTask+0x3b9 [d:\source_code\chromium\depot_tools\src\base\message_loop\message_loop.cc @ 470]
12 base!base::MessageLoop::DeferOrRunPendingTask+0x3c [d:\source_code\chromium\depot_tools\src\base\message_loop\message_loop.cc @ 481]
13 base!base::MessageLoop::DoWork+0x172 [d:\source_code\chromium\depot_tools\src\base\message_loop\message_loop.cc @ 602]
14 base!base::MessagePumpForUI::DoRunLoop+0x61 [d:\source_code\chromium\depot_tools\src\base\message_loop\message_pump_win.cc @ 263]
15 base!base::MessagePumpWin::Run+0x9d [d:\source_code\chromium\depot_tools\src\base\message_loop\message_pump_win.cc @ 143]
16 base!base::MessageLoop::RunHandler+0xf6 [d:\source_code\chromium\depot_tools\src\base\message_loop\message_loop.cc @ 433]
17 base!base::RunLoop::Run+0x3d [d:\source_code\chromium\depot_tools\src\base\run_loop.cc @ 36]
18 content!content::BrowserMainLoop::MainMessageLoopRun+0x173 [d:\source_code\chromium\depot_tools\src\content\browser\browser_main_loop.cc @ 1431]
19 content!content::BrowserMainLoop::RunMainMessageLoopParts+0x15f [d:\source_code\chromium\depot_tools\src\content\browser\browser_main_loop.cc @ 962]
1a content!content::BrowserMainRunnerImpl::Run+0x1a8 [d:\source_code\chromium\depot_tools\src\content\browser\browser_main_runner.cc @ 156]
1b content_shell!ShellBrowserMain+0x111 [d:\source_code\chromium\depot_tools\src\content\shell\browser\shell_browser_main.cc @ 31]
1c content_shell!content::ShellMainDelegate::RunProcess+0xc5 [d:\source_code\chromium\depot_tools\src\content\shell\app\shell_main_delegate.cc @ 295]
1d content!content::RunNamedProcessTypeMain+0x9d [d:\source_code\chromium\depot_tools\src\content\app\content_main_runner.cc @ 405]
1e content!content::ContentMainRunnerImpl::Run+0x278 [d:\source_code\chromium\depot_tools\src\content\app\content_main_runner.cc @ 786]
1f content!content::ContentMain+0x81 [d:\source_code\chromium\depot_tools\src\content\app\content_main.cc @ 20]
20 content_shell!wWinMain+0x7b [d:\source_code\chromium\depot_tools\src\content\shell\app\shell_main.cc @ 33]
21 content_shell!invoke_main+0x2d [f:\dd\vctools\crt\vcstartup\src\startup\exe_common.inl @ 118]
22 content_shell!__scrt_common_main_seh+0x127 [f:\dd\vctools\crt\vcstartup\src\startup\exe_common.inl @ 253]
23 content_shell!__scrt_common_main+0xe [f:\dd\vctools\crt\vcstartup\src\startup\exe_common.inl @ 296]
24 content_shell!wWinMainCRTStartup+0x9 [f:\dd\vctools\crt\vcstartup\src\startup\exe_wwinmain.cpp @ 17]
25 KERNEL32!BaseThreadInitThunk+0x14
26 ntdll!RtlUserThreadStart+0x21

```

