# Get Source and Compile

## prepare development environment

先准备好环境，再去拿code，要不会失败！！！务必将下面的环境变量设置好！！！

### 

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

## get source code

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

## compile code

Set up the build:
```
gn gen out/Default
```

Building:
```
ninja -C out\Default chrome
```

A minimal solution that will let you open Chromium source code in the IDE but will not show any source files is:
```
gn gen --ide=vs --sln=mini --filters="//chrome;//base;//content" out\Default
```

## references

More information about --filter could refer to following link:
[https://bugs.chromium.org/p/chromium/issues/detail?id=589099](https://bugs.chromium.org/p/chromium/issues/detail?id=589099)

More information about build instructions, please refer to following link:
[https://chromium.googlesource.com/chromium/src/+/master/docs/windows_build_instructions.md](https://chromium.googlesource.com/chromium/src/+/master/docs/windows_build_instructions.md)