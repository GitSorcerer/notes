# minio 上传下载文件失败 （The difference between the request time and the server's time is too large.）



# minio上传下载文件失败：

错误消息：

```
The difference between the request time and the server's time is too large.
```

原因：linux服务器时区的问题。

# 解决方案：

一、查看系统时间、硬件时间

```
1.# date // 查看系统时间
2.#hwclock // 查看硬件时间
```

二、时间服务器上的时间同步的方法
安装ntpdate工具

```
1.# yum -y install ntp ntpdate
```

设置系统时间与网络时间同步

```
2.# ntpdate cn.pool.ntp.org
```

将系统时间写入硬件时间

```
3.# hwclock --systohc
```