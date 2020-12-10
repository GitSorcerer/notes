# 安装redis

# 下载Redis

```html
 wget http://download.redis.io/releases/redis-5.0.5.tar.gz
```

解压

```shell
tar -zxvf redis-5.0.5.tar.gz
#移动文件
mv redis-5.0.5.tar.gz /usr/redis
//进入redis目录
cd /usr/redis 
#安装gcc环境
 yum -y install gcc-c++
#make编译
make
#进入src目录
cd src/
#编译
make install
#新建文件夹
mkdir ect
mkdir bin

#将下面文件移动到斌目录下 
mv mkreleasehdr.sh redis-benchmark redis-check-aof redis-check-rdb redis-cli redis-server /usr/local/redis-5.0.5/bin/

redis-server ../etc/redis.conf 

#查看redis信息
netstat -tunpl |grep 6379

```

# elasticSearch安装

　1）添加用户：adduser esuser

　　2）设定密码：passwd esuser

　　3）添加权限：chown -R esuser /opt/elasticsearch-6.5.0



yml

```
network.host: 0.0.0.0
cluster.name: zccs
http.cors.enabled: true
http.cors.allow-origin: "*"
```



/etc/security/limits.conf

```
* soft nofile 131072
* hard nofile 131072
* soft nproc 262144
* hard nproc 262144

* - nofile 65536
* - memlock unlimited
```

/etc/sysctl.conf

```
vm.max_map_count=262144
```

