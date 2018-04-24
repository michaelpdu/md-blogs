# IPC Fuzzer介绍

在Chromium的source code中，有专门的fuzzer tool（`src/tools/ipc_fuzzer`）用于对IPC进行模糊测试。

## 工具的编译

想要使用这个fuzzer，需要配合相应的编译选项进行编译。Chromium的Markdown里有简单的介绍，可以参考这个文档里的描述：

[https://chromium.googlesource.com/chromium/src/+/lkcr/docs/ipc_fuzzer.md](https://chromium.googlesource.com/chromium/src/+/lkcr/docs/ipc_fuzzer.md)

在Build instructions中写到，当前的ipc_fuzzer，不支持调试版本，不支持component build，所以，必须将这两个选项关闭。另外，build target需要设置为ipc_fuzzer_all，这样，可以将相关的tools都build出来。

个人设置的编译选项以及编译命令：

```sh
gn gen out/ipc_fuzzer --args="enable_ipc_fuzzer=true is_debug=false is_component_build=false enable_nacl=false remove_webcore_debug_symbols=true use_jumbo_build=true"
ninja -C out/ipc_fuzzer ipc_fuzzer_all
ninja -C out/ipc_fuzzer chrome
```

## 工具的使用

在 `src/tools/ipc_fuzzer/scripts` 目录下面，有一些Python tools，用于辅助测试。

```sh
$ ll ./
total 48
drwxrwxr-x  8 michael michael 4096 4月   5 01:16 ./
drwxrwxr-x 91 michael michael 4096 4月   5 01:23 ../
-rw-rw-r--  1 michael michael  928 4月   5 01:16 BUILD.gn
-rw-rw-r--  1 michael michael   77 4月   5 01:16 DEPS
drwxrwxr-x  2 michael michael 4096 4月   5 01:16 fuzzer/            # 生成ipc_fuzzer
-rw-rw-r--  1 michael michael  700 4月   5 01:16 ipc_fuzzer.gni
drwxrwxr-x  2 michael michael 4096 4月   5 01:16 message_dump/      
drwxrwxr-x  2 michael michael 4096 4月   5 01:16 message_lib/
drwxrwxr-x  2 michael michael 4096 4月   5 01:16 message_replay/
drwxrwxr-x  2 michael michael 4096 4月   5 01:16 message_tools/
-rw-rw-r--  1 michael michael  104 4月   5 01:16 OWNERS
drwxrwxr-x  2 michael michael 4096 4月  23 17:38 scripts/

./scripts/
├── cf_package_builder.py       # 
├── ipc_fuzzer_gen.py           # Generational ClusterFuzz fuzzer
├── ipc_fuzzer_mut.py           # Mutational ClusterFuzz fuzzer
├── play_testcase.py            # 用于回放testcase的脚本
├── remove_close_messages.py    # 
├── utils.py                    # ipc_fuzzer_gen.py和ipc_fuzzer_mut.py公用库文件


```

### 生成ipcdump

生成ipcdump的工具是`ipc_fuzzer`，每调用一次下面的command，可以生成一个ipcdump file。

```sh
$ ./ipc_fuzzer
Mutate messages from an exiting message file.
Usage:
  ipc_fuzzer [--count=c] [--frequency=q] [--fuzzer-name=f] [--help] [--type-list=x,y,z...] [--permute] [infile (mutation only)] outfile
 --count        - Number of messages to generate (generator).
 --frequency    - Probability of mutation; tweak every 1/|q| times (mutator).
 --fuzzer-name  - Select from generate, mutate, or no-op. Default: generate
 --help         - Show this message.
 --type-list    - Explicit list of the only message-ids to mutate (mutator).
 --permute      - Randomly shuffle the order of all messages (mutator).

```

另外也可以使用`ipc_fuzzer_gen.py|ipc_fuzzer_mut.py`批量化生成ipcdump files。

这里以`ipc_fuzzer_gen.py`为例，介绍具体使用方法。

```sh
# 设置chrome的路径
export BUILD_DIR=/sa/src_code/chromium/src/out/ipc_fuzzer
# 生成指定数量的testcase，这里是500
# 对于ipc_fuzzer_gen.py而言， --input_dir这个选项是没有实际作用的
# --output_dir用于指定对于的testcase文件存放在哪里
# --no_of_files用于指定生成多少个testcase files
python ipc_fuzzer_gen.py --input_dir=/sa/src_code/chromium/src/out/ipc_fuzzer --output_dir=/sa/src_code/chromium/src/out/ipc_fuzzer/ipcdump/gen_dir --no_of_files=500
```

使用`ipc_fuzzer_gen.py`调用`ipc_fuzzer`，相当于使用了N次下面的命令：

```sh
ipc_fuzzer --fuzzer-name=generate --count=num_ipc_messages fuzz-xxx.ipcdump
# MAX_IPC_MESSAGES_PER_TESTCASE = 1500
# 这里的num_ipc_messages是一个[0, MAX_IPC_MESSAGES_PER_TESTCASE]的随机数
```

### 回放生成的IPC messages

`play_testcase.py`是用于testcase回放的工具


### IPC Message相关的工具

#### Dump Messages

可以使用`ipc_message_util`将.ipcdump文件中的messages打印出来。

```sh
$ ./ipc_message_util --help
ipc_message_util: Concatenate all |infile| message files and copy a subset of the result to |outfile|.
Usage:
  ipc_message_util [--start=n] [--end=m] [--in=i[,j,...]] [--regexp=x] [--invert] [--dump] [--help] infile,infile,... [outfile]
    --start  - output messages after |n|th message in file (inclusive).
    --end    - output messages before |m|th message in file (exclusive).
    --in     - output only the messages at the specified positions in the file.
    --regexp - output messages matching regular expression |x|.
    --invert - output messages NOT meeting above criteria.
    --dump   - dump human-readable form to stdout instead of copying.
    --help   - display this message.
```

具体使用方法，举例说明：

```sh
# dump出指定*.ipcdump文件中的messages
$ ./ipc_message_util --dump ipcdump/gen_dir/fuzz-youvuucarodvqgne.ipcdump | less

0. FrameMsg_GetSerializedHtmlWithLocalLinks
1. PpapiHostMsg_Gamepad_Create
2. BlobHostMsg_RevokePublicURL
3. PpapiHostMsg_Graphics2D_Create
4. FrameMsg_Collapse
5. BrowserPluginHostMsg_ExtendSelectionAndDelete
6. PpapiPluginMsg_MediaStreamTrack_EnqueueBuffers
7. PpapiPluginMsg_FileRef_MakeDirectoryReply
8. PpapiHostMsg_AudioEncoder_RequestBitrateChange
9. ServiceWorkerMsg_ServiceWorkerStateChanged
10. ChromotingNetworkToAnyMsg_StopProcessStatsReport
11. PrintMsg_PrintingDone
12. FrameMsg_IntrinsicSizingInfoOfChildChanged
13. FrameHostMsg_RunFileChooser
14. FrameHostMsg_DownloadUrl
15. PpapiPluginMsg_AudioEncoder_NotifyError
16. ExtensionHostMsg_RequestWorker
17. FrameHostMsg_PepperPluginHung
18. PpapiPluginMsg_VideoEncoder_GetSupportedProfilesReply
19. PpapiPluginMsg_AudioOutput_OpenReply
20. BrowserPluginHostMsg_SetEditCommandsForNextKeyEvent
...

# dump符合正则表达式`FrameHost.*`的messages
$ ./ipc_message_util --regexp='FrameHost.*' --dump ipcdump/gen_dir/fuzz-youvuucarodvqgne.ipcdump | less

13. FrameHostMsg_RunFileChooser
14. FrameHostMsg_DownloadUrl
17. FrameHostMsg_PepperPluginHung
22. FrameHostMsg_PluginCrashed
23. FrameHostMsg_ShowCreatedWindow
29. FrameHostMsg_JavaScriptExecuteResponse
...

```

#### List IPC Messages

