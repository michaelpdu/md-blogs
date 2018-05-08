# prepare development environment

先准备好环境，再去拿code，要不会失败！！！有几个重要的环境变量需要提前设置好，如下：

```
DEPOT_TOOLS_WIN_TOOLCHAIN=0
GYP_MSVS_VERSION=2017 # if VS is 2015, this value need to be 2015
```

如果重新指定了VS2017的安装目录，需要设置下面的环境变量，因为在toolchain的默认路径中是找不到的。

```
vs2017_install="D:\Program Files\Visual Studio 2017"
```

## 之前的记录（for VS2015）

As of March 11, 2016 Chromium requires Visual Studio 2015 to build.

Install Visual Studio 2015 Update 2 or later - Community Edition should work if its license is appropriate for you. Use the Custom Install option and select:

  * Visual C++, which will select three sub-categories including MFC
  * Universal Windows Apps Development Tools > Tools
  * Universal Windows Apps Development Tools > Windows 10 SDK (10.0.10586)

You must have the 10586 SDK installed or else you will hit compile errors such as redefined macros.

Install Windows Driver Kit (WDK) 10, or use some other method to get the Debugging Tools for Windows.

Run set DEPOT_TOOLS_WIN_TOOLCHAIN=0, or set that variable in your global environment.

Compilation is done through ninja, not Visual Studio.

上面强调的部分一定要格外注意，之前没有设置这个环境变量，导致gclient sync的时候，总是在get toolchain的时候失败，最后分析才知道是需要设置这个变量。

### 

官方source code中，有不少文件包含特殊的字符，从而导致编译失败，需要先将system locale改为US。否则会出现如下错误：

```
d:\source_code\chromium\depot_tools\src\native_client\src\trusted\validator_ragel\validator.h: warning C4819: The file contains a character that cannot be represented in the current code page (936). Save the file in Unicode format to prevent data loss
```

# get source code

Get source code:

```
fetch chromium
```

fetch以后，会有一个.gclient file和src folder

```
cd src
```

Update code:

```
git rebase-update
gclient sync
```

# compile code

**使用GN生成.ninja**

```
gn gen out/Default
```

这里，也可以使用args指定相应的参数。

```sh
# 下面这个命令会直接使用args的参数生成.ninja，相应的args，也会存放在目标文件夹下的args.gn中
gn gen out\chrome_x86_rel --args="is_debug=false is_component_build=false enable_nacl=false remove_webcore_debug_symbols=true is_clang=false symbol_level=2 use_jumbo_build=true"

# 也可以使用下面的命令，此时，会启动文本编辑器用于编辑args.gn，编辑完成以后保存，gn会自动生成.ninja
gn args out\chrome_x86_rel

# 如果想查看具体使用的args，可以使用下面的命令
gn args --list out\chrome_x86_rel

# 使用--ide指定对应的开发环境
# 使用--sln指定VS的.sln文件名
# 使用--filters指定只包含特定的文件目录，其中//是指src的根目录
gn gen --ide=vs --sln=mini --filters="//chrome;//base;//content" out\Default
```

GN的语法可以参考：

- [https://chromium.googlesource.com/chromium/src/+/master/tools/gn/docs/language.md](https://chromium.googlesource.com/chromium/src/+/master/tools/gn/docs/language.md)
- [https://blog.csdn.net/yujiawang/article/details/72627138](https://blog.csdn.net/yujiawang/article/details/72627138)

**使用Ninja编译目标文件**

```
ninja -C out/Default chrome
```

# build in Linux

refer to [Linux Build Instructions](https://chromium.googlesource.com/chromium/src/+/master/docs/linux_build_instructions.md#Install)

# references

More information about --filter could refer to following link:
[https://bugs.chromium.org/p/chromium/issues/detail?id=589099](https://bugs.chromium.org/p/chromium/issues/detail?id=589099)

More information about build instructions, please refer to following link:
[https://chromium.googlesource.com/chromium/src/+/master/docs/windows_build_instructions.md](https://chromium.googlesource.com/chromium/src/+/master/docs/windows_build_instructions.md)
