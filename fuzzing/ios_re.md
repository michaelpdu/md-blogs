# 常用工具
## 使用USB调试iphone

### ssh连接越狱iPhone

usbmuxd自带工具iproxy，iproxy可以快捷连接iPhone操作。由于Mac上只支持4位的端口号，所以需要把iPhone的默认端口22映射到Mac上，相当于建立一个Mac和iPhone的通道。

终端输入：
```
iproxy 2525 22
```
然后会自动显示如下等待连接字样
```
waiting for connection
```

在另外一个终端输入：
```
ssh -p 2222 root@127.0.0.1
```

参考：[SSH连接越狱iPhone](https://www.jianshu.com/p/d5fbacb1bf5c)

### 默认密码

越狱以后的默认密码：
```
alpine
```


# 其他信息

## iPhone与CPU架构对应关系

```
armv6: iPhone、iPhone 2、iPhone 3G、iPod Touch(第一代)、iPod Touch(第二代)       

armv7: iPhone 3Gs、iPhone 4、iPhone 4s、iPad、iPad 2

armv7s: iPhone 5、iPhone 5c (静态库只要支持了armv7,就可以在armv7s的架构上运行)

arm64(注:无armv64): iPhone 5s、iPhone 6、iPhone 6 Plus、iPhone 6s、iPhone 6s Plus、 iPhone 7 、iPhone 7 Plus、iPad Air、iPad Air2、iPad mini2、iPad mini3、iPad mini4、iPad Pro

```
