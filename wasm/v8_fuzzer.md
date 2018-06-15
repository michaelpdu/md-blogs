下载和编译V8，可以参考官方文档：

[Build V8 from Source Code](https://github.com/v8/v8/wiki/Building-from-Source)

如果需要使用AFL插桩，可以参考下面这篇文章的介绍：

[Ubuntu下编译pdfium](https://www.jianshu.com/p/8bb348ba8d61) 中方法1中介绍，Chromium中默认是支持afl的，所以，可以使用相同的方法，将AFL下载到third_party目录下。

在生成命令中增加对于的gn选项，如下：
```
python tools/dev/v8gen.py gen -b x64.release x64.release.afl -- use_afl=true optimize_for_fuzzing=true
```

只编译v8_fuzzer的话，可以使用下面的命令：
```
ninja -C out.gn/x64.release.afl2/ v8_fuzzer 
```
