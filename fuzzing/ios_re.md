# 常用工具

## 使用USB调试iphone

用wifi调试的时候，会觉得有些卡壳，所以，改用USB，这样速度会明显提升。使用USB连接，需要安装usbmuxd，安装以后，做端口转发即可。

### ssh连接越狱iPhone

usbmuxd自带工具iproxy，iproxy可以快捷连接iPhone操作。由于Mac上只支持4位的端口号，所以需要把iPhone的默认端口22映射到Mac上，相当于建立一个Mac和iPhone的通道。

终端输入：
```
iproxy 25025 22
```
然后会自动显示如下等待连接字样
```
waiting for connection
```

在另外一个终端输入：
```
ssh -p 25025 root@127.0.0.1
```

参考：[SSH连接越狱iPhone](https://www.jianshu.com/p/d5fbacb1bf5c)

下面的nohup命令是让iproxy在后台运行，这样，可以避免起多个terminal：

```
nohup iproxy 25025 22 > /dev/null 2>&1 &
ssh root@localhost -p 25025
```

### 默认密码

越狱以后的默认密码：
```
alpine
```

## 使用IDA分析可执行程序

这个毋庸多说，直接找对应的可执行程序，拷贝到mac上，然后，用IDA打开就可以了。

需要说的是对于加壳的程序，需要先砸壳。下面就介绍砸壳的过程：

1. 下载和编译下面的dumpdecrypted，编译完成以后，会生成dumpdecrypted.dylib

[https://github.com/stefanesser/dumpdecrypted](https://github.com/stefanesser/dumpdecrypted)

2. 将dumpdecrypted.dylib拷贝到当前权限可以写的目录下（看到不少文章写要放在对应的document目录，应该是对于没有越狱的iphone，有权限限制的问题）

```
DYLD_INSERT_LIBRARIES=dumpdecrypted.dylib /var/mobile/Applications/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx/AAA.app/AAA
```

3. 执行完成以后，就会发现有一个AAA.decrypted的程序，这样，就解决加壳的问题了。

## 使用LLDB调试

### 配置debugserver

默认连接xcode给ios添加的debugserver是支持多个CPU架构的，需要根据自己的机型选择合适的CPU架构，这些信息可以参考下面的`iPhone与CPU架构对应关系`

目前我使用的是iphone5s，所以，保留arm64的平台。

从ios上将debugserver拷贝到mac上，然后，用下面的命令给debugserver瘦身。

```
lipo -thin arm64 ./debugserver -output ./debugserver
```

给debugserver添加task_for_pid权限，需要先下载[ent.xml](http://7xibfi.com1.z0.glb.clouddn.com/uploads/default/668/c134605bb19a433f.xml)， 然后，用下面的命令:

```
ldid -Sent.xml debugserver
```

然后，将瘦身的debugserver再拷贝到iphone的/user/bin目录下，可以直接执行

### 在iOS上用debugserver来attach进程

调试器常用的两种方式，一种是attach process，另外一种是start process，下面分别介绍：

#### attach process

启动debugserver，并attach SpringBoard，开放端口25026给LLDB客户端连接

```
debugserver *:25026 -a "SpringBoard"
```

在MacOS的终端上，先做端口转发，将本地25026端口转发到iphone的25026端口上

```
nohup iproxy 25026 25026 > /dev/null 2>&1 &
```

在LLDB中用下面的命令连接：

```
process connect connect://localhost:25026
```

参考：[iOS应用逆向工程](http://iosre.com/t/debugserver-lldb-gdb/65)

#### start process

```
debugserver -x backboard *:25026 path-to-application

网上有些说法是用posix代替backboard，个人实验以后，发现下面的情况：
1) backboard可以看到用户界面
2）posix看不到UI
具体使用哪种，视情况而定
```

### 对抗反调试问题

对于有反调试保护的APP，需要绕开反调试，目前反调试的方法主要还是以后ptrace，所以，只要跳过ptrace即可。

实现方法多样，可以是调试器的command，也可以是script，也可以自己编写tweak。

用command的话，下面的命令可供参考：

```
# set breakpoint at ptrace
b ptrace

# continue and trigger breakpoint
c

# modify $pc to return address
register write pc `$lr`

# continue to run
c
```

# 其他信息

## iPhone与CPU架构对应关系

```
armv6: iPhone、iPhone 2、iPhone 3G、iPod Touch(第一代)、iPod Touch(第二代)       

armv7: iPhone 3Gs、iPhone 4、iPhone 4s、iPad、iPad 2

armv7s: iPhone 5、iPhone 5c (静态库只要支持了armv7,就可以在armv7s的架构上运行)

arm64(注:无armv64): iPhone 5s、iPhone 6、iPhone 6 Plus、iPhone 6s、iPhone 6s Plus、 iPhone 7 、iPhone 7 Plus、iPad Air、iPad Air2、iPad mini2、iPad mini3、iPad mini4、iPad Pro

```

## LLDB Cheatsheet

[LLDB Cheatsheet](https://www.nesono.com/sites/default/files/lldb%20cheat%20sheet.pdf)
