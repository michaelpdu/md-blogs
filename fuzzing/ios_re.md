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
