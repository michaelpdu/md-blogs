---
title: 利用flash player脱壳的思路
date: 2016-08-31 21:51:47
tags: flash
---

# 缘起

随着混淆工具在恶意flash中的使用越来越流行，一对一的脱壳手法已经不能满足需求，想找一种简便的方案，最后想在flash player中做点文章，既然flash player可以正确地执行flash，那么，脱壳对flash player而言，就不是问题。


# 思路

2015年的时候，做过这些事，当时没有记录如何找到hook point的过程，从而导致过了许久以后，出了问题，想重新找到其他版本flash player上的hook point的过程就显得较为耗时。不过，耗时归耗时，还是可以找到，这里将这次找hook point的思路整理出来，方便以后遇到类似问题所使用。

最开始想做这个的时候，有两种思路：

- flash脱壳会用到loadBytes，那么，可以hook这个JIT函数，因为是JIT函数，所以，在hook之前需要做些其他的事，稍微有些麻烦。下面有两种实现，可以参考：
  + F-Secure实现的[sulo](https://github.com/F-Secure/Sulo)是基于PIN技术做的，本质上也是Hook了loadBytes函数
  + 作者实现的WinDBG plugin： [flashext](https://github.com/michaelpdu/flashext)，其中，就包括了Hook loadBytes是思路和实现
- flash中的JIT函数最后应该还是会调用到flash player的inline函数，我们能不能找到这个点，然后，直接在inline function上做Hook

因为第一种思路，作者已经在WinDBG Plugin中有了类似的实现，所以想去找找第二种是否有可能做到，如果能找到，应该会更简单一些。

既然flash player可以将embedded flash解开并加载，那么，肯定会有加载embedded flash的指令，并且这些指令一定会检测magic number，所以，这些magic number就是一个很合适的点。

## 通过调试器找到准确的加载时机

通过vmware+debugger的帮助，可以准备一个snapshot，使得在分配embedded flash的地址时是相同的基址，这样，就可以很方便地使用硬件断点来找到读写这个基址的指令。

需要一个断点来辅助我们在加载embedded flash之前停下来，之前，有flashext的帮助，可以在loadbytes JIT function上断下来，而对于Windows 8以后内置的flash而言，之前的flashext也遇到同样的困境，hook point失效了。
那么我们需要找到其他的点，也辅助我们完整这个工作，通过搜索strings，可以看到，像FWS/CWS之类的string都没有找到合适的。

![](images/thinking-of-embedded-flash-dump-in-flash-player/1.jpg)

那么，我们再试试找常量，因为我们知道FWS = \x46\x57\x53，CWS=\x43\x57\x53， ZWS=\x5A\x57\x53， 所以，\x57 \x53如果能够同时出现在一段指令中，说明可能是我们期望的目标。

![](images/thinking-of-embedded-flash-dump-in-flash-player/2.jpg)

![](images/thinking-of-embedded-flash-dump-in-flash-player/3.jpg)

![](images/thinking-of-embedded-flash-dump-in-flash-player/4.jpg)

在这个上下文的指令中可以找到\x43,\x53,\x57之类的常量，我们可以分析一下这些指令的意思，的确也是在比较内存是否是CWS或者FWS之类的东西，太好了，那么，上调试器看看是否就是我们想要找的hook point?

通过下面命令来设置断点：

```
bp Flash+22870A
```

在调试环境中可以看到，这个点其实并非我们想要的点，不过，却可以用于辅助我们完成定位读写embedded flash的指令。（如之前使用flashext中给loadBytes设置breakpoint一样）

![](images/thinking-of-embedded-flash-dump-in-flash-player/5.jpg)

在我的调试环境中，执行前六次go命令，还没有触发解开embedded flash的动作，所以在memory中也是找不到的。第七次执行go命令，在memory中就可以找到相应的embedded flash。经过几次尝试以后，我们可以将embedded flash的分配地址固定下来，在我这里是0x0af3f000。

![](images/thinking-of-embedded-flash-dump-in-flash-player/6.jpg)

通过硬件断点，可以方便知道哪条指令在这个地址上写了内容。

![](images/thinking-of-embedded-flash-dump-in-flash-player/7.jpg)

也可以很方便知道，哪条指令会从这里读取数据，这里，很有可能就是我们想要的点，计算偏移，去IDA里看看。

![](images/thinking-of-embedded-flash-dump-in-flash-player/8.jpg)

分析这段汇编指令以后可以知道这个函数接收两个参数，用于比较这两个string是否相等。这里，是比较ESI所指向的memory是否等于"CWS","FWS"或“ZWS”。配合调试器可以找到caller，然后，再到IDA中看看究竟。

![](images/thinking-of-embedded-flash-dump-in-flash-player/9.jpg)

这段汇编指令很简单，就是检查ESI所指向的memory是否是SWF的magic number，看来距离我们的目标越来越近了。继续往下分析，看看谁调用了这个函数。

![](images/thinking-of-embedded-flash-dump-in-flash-player/10.jpg)

分析相应的caller以后，可以弄明白这个函数是load flash的地方，其中，传入的参数，有一个是embedded flash的首地址，存放在ESI中，有一个是flash length，存放在EDI中。这样，我们的目标就确定了。

## 在IDA中搜索text查找FWS,CWS,ZWS

在上面的分析过程中，我发现magic number的字符串其实是有地方存放的，只是在string window中找不到。

![](images/thinking-of-embedded-flash-dump-in-flash-player/11.jpg)

那么，有没有其他途径？可以试试在文本内容中搜索，只是速度较慢。

![](images/thinking-of-embedded-flash-dump-in-flash-player/12.jpg)

![](images/thinking-of-embedded-flash-dump-in-flash-player/13.jpg)

等待一段时间以后，可以看到，全文搜索以后，能符合我们需求的地方，也只有一个，而且，恰恰就在我们之前看到的check_flash_magic_number这个函数里。这种方式要比上面介绍的方式更快捷，如果不在意前面的思路，可以直接用这种方式找到相应的点。

当然，还有其他的hook point，如果有人找到更合适的点，可以分享出来。