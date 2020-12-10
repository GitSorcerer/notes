# linux常见问题

## 1.不能用root用户启动

chown username /usr/local/elasticsearch/elasticsearch-6.6.1 -R 

```shell
adduser gaoh
passwd gaoh
...
chown gaoh /usr/local/elasticsearch/elasticsearch-6.6.1 -R 
su gaoh
```

## 2.内存不够

```
执行命令：

sysctl -w vm.max_map_count=262144

查看结果：

sysctl -a|grep vm.max_map_count

显示：

vm.max_map_count = 262144

 

上述方法修改之后，如果重启虚拟机将失效，所以：

解决办法：

在   /etc/sysctl.conf文件最后添加一行

vm.max_map_count=262144

即可永久修改
```

如果启动后直接制动kill 

# 下面方法直接使用

修改jvm.options

```
-Xms512m
-Xmx512m
```



如若其他问题

1.修改/etc/security/limits.conf

```

* soft nofile 65536
* hard nofile 131072
* soft nproc 2048
* hard nproc 4096

```

1.修改/etc/sysctl.conf

```
vm.max_map_count=262144
```

