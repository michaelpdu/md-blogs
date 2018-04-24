# Chrome IPC and Mojo IPC

在Chrome中，并存了两种IPC通信机制，一种是传统的IPC，另外一种是Mojo IPC。



## Mojo的介绍

[Google Source Mojo](https://chromium.googlesource.com/chromium/src/+/master/mojo/README.md)
[](https://docs.google.com/drawings/d/1RwhzKblXUZw-zhy_KDVobAYprYSqxZzopXTUsbwzDPw/pub?w=570&h=324)

## Mojo EDK的使用

[Mojo EDK](https://chromium.googlesource.com/chromium/src/+/master/mojo/edk/embedder)

## 如何从传统的IPC转到Mojo IPC

[Converting Legacy Chrome IPC To Mojo](https://chromium.googlesource.com/chromium/src/+/master/ipc)

## Chrome IPC与Mojo IPC的对应关系

[Cheat Sheet](https://www.chromium.org/developers/design-documents/mojo/chrome-ipc-to-mojo-ipc-cheat-sheet)

## Chrome中的一些配置

在chrome的tab页，输入：chrome://flags，然后，搜索mojo，可以找到下面这个选项。
```
Navigation response using Mojo
Browser side navigation (aka PlzNavigate) is using blob URLs to deliver the body of the main resource to the renderer process. This flag replaces this mechanism by using a Mojo DataPipe. – Mac, Windows, Linux, Chrome OS, Android
```

PlzNavigate的作用是使用browser process代替render processs做navigation，好处如下：
```
We are starting a new project to move the navigation logic from the renderer process to the browser process. This will be beneficial in terms of:
- speed: this allows to parallelize renderer startup with network requests (renderer startup takes a median 170ms on Android).
- simplicity: the content module will be greatly simplified, as pending render frame hosts and the transfer mechanism will be removed
- security: the browser process will be in a better position to decide which navigations are allowed, and which renderer process to use for them.

```

具体详情，可以参考：

[Plz Navigate: Browser-side navigation in Chromew](https://docs.google.com/document/d/1cSW8fpJIUnibQKU8TMwLE5VxYZPh4u4LNu_wtkok8UE/edit#heading=h.xt7xrw8lahpf)

[Discussion Topic](https://groups.google.com/a/chromium.org/forum/#!topic/chromium-dev/sOs05JfmPFw)