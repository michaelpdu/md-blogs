下载和编译V8，可以参考官方文档：

[Build V8 from Source Code](https://github.com/v8/v8/wiki/Building-from-Source)

如果需要使用AFL插桩，可以参考下面这篇文章的介绍：

[Ubuntu下编译pdfium](https://www.jianshu.com/p/8bb348ba8d61) 中方法1中介绍，Chromium中默认是支持afl的，所以，可以使用相同的方法，将AFL下载到third_party目录下。

在生成命令中增加对于的gn选项，如下：
```
python tools/dev/v8gen.py gen -b ia32.release ia32.release.afl -- use_afl=true optimize_for_fuzzing=true
```

如果需要编译x86 build，也可以将上面的x64.release替换为x86的，可以使用 v8gen.py list命令列出目前支持的平台。
```
michael@michael-OptiPlex-9020:/sa/github/google/v8$ python tools/dev/v8gen.py list
android.arm.debug
android.arm.optdebug
android.arm.release
arm.debug
arm.optdebug
arm.release
arm64.debug
arm64.optdebug
arm64.release
ia32.debug
ia32.optdebug
ia32.release
x64.debug
x64.optdebug
x64.release
...
```

只编译v8_fuzzer的话，可以使用下面的命令：
```
ninja -C out.gn/ia32.release.afl/ v8_fuzzer 
```

切换到不同的branch上，方便验证老版本上的问题，以6.8版本为例：
```
# https://v8project.blogspot.com/2018/06/v8-release-68.html
# 创建并拉去6.8的分支
git checkout -b 6.8 -t branch-heads/6.8
# 拉去最新的code
git pull origin
# 查看分支
git branch
# 切换分支
git checkout xxx
```

编译V8时，可以开启一些编译选项
```
# 开启ASan，在build/config/sanitizers/sanitizers.gni中
# Compile for Address Sanitizer to find memory bugs.
is_asan = true
```

编译standalone_v8，可以将v8-standalone.cc拷贝到samples/下，然后，在BUILD.gn中增加下面的命令：
```
v8_executable("v8_standalone") {
  sources = [
    "samples/v8-standalone.cc",
  ]

  configs = [
    # Note: don't use :internal_config here because this target will get
    # the :external_config applied to it by virtue of depending on :v8, and
    # you can't have both applied to the same target.
    ":internal_config_base",
  ]

  deps = [
    ":v8",
    ":v8_libbase",
    ":v8_libplatform",
    "//build/win:default_exe_manifest",
  ]
}
```

然后，使用下面的命令可以单独生成v8_standalone工具：
```
python tools/dev/v8gen.py gen -b ia32.release ia32.release.6.8
ninja -C out.gn/ia32.release.6.8/ v8_standalone 
```
