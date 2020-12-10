# RPM安装

### 下载对应得RPM

**链接:[https://downloads.mysql.com/archives/community/]( https://downloads.mysql.com/archives/community/
)**

选择对应得版本

![](assets/mysqlrpm.png)

### 准备安装

#### 安装服务端

**命令**

```shell
rpm -ivh MySQL-server-5.5.8-1.linux2.6.x86_64.rpm
```

**出现安装异常rpm冲突**

```properties
file /usr/share/mysql/russian/errmsg.sys from install of MySQL-server-5.5.8-1.linux2.6.x86_64 conflicts with file from package mariadb-libs-1:5.5.60-1.el7_5.x86_64
	file /usr/share/mysql/serbian/errmsg.sys from install of MySQL-server-5.5.8-1.linux2.6.x86_64 conflicts with file from package mariadb-libs-1:5.5.60-1.el7_5.x86_64
	file /usr/share/mysql/slovak/errmsg.sys from install of MySQL-server-5.5.8-1.linux2.6.x86_64 conflicts with file from package mariadb-libs-1:5.5.60-1.el7_5.x86_64
```

**移除对应得rpm**

```shell
yum -y remove mariadb-libs-1:5.5.60-1.el7_5*
yum -y remove mariadb-libs-1:5.5.52-1.el7*
```

**重新安装**

```shell
rpm -ivh MySQL-server-5.5.8-1.linux2.6.x86_64.rpm
```

**日志信息**

![](assets/MysqlSeccuss.png)

#### 安装客户端

```shell
 rpm -ivh MySQL-client-5.5.8-1.linux2.6.x86_64.rpm 
```

#### 安装失败

**如果安装失败显示*"GPG keys ..."***

```shell
rpm -ivh MySQL-server-5.5.8-1.linux2.6.x86_64.rpm --force --nodoeps
```

**检验是否安装完成**

```shell
mysqladmin  --version
```

显示对应得版本则安装成功!

### 基本命令

**启动:**

```shell
service mysql start
```

**停止**

```shell
service mysql stop
```

重启

```shell
service mysql restart
```

**设置开机自启/关闭**

```shell
1.每次使用前手动启动服务etc/init.d/mysql start
chkconfig mysql on
chkconfig mysql off #关闭
ntsysv   #查看是否自启
```

### 相关配置

#### **日志信息**

```shell
/usr/bin/mysqladmin -u root password 'new-password'
/usr/bin/mysqladmin -u root -h VM_0_14_centos password 'new-password'
```

#### **设置密码**

```shell
/usr/bin/mysqladmin -u root password gaohangwangh922104
```

#### **查看数据库存放目录**

```shell
ps -ef|grep mysql
```

显示:

```properties
/bin/sh /usr/bin/mysqld_safe --datadir=/var/lib/mysql --pid-file=/var/lib/mysql/VM_0_14_centos.pid TERM=xterm SYSTEMCTL_SKIP_REDIRECT= PATH=/sbin:/usr/sbin:/bin:/usr/bin:/usr/bin PWD=/usr SHLVL=2 SYSTEMCTL_IGNORE_DEPENDENCIES= OLDPWD=/ _=/usr/bin/mysqld_safe
```

#### 文件目录

**数据存放目录:datadir=/var/lib/mysql*** 

***pid文件目录:pid-file=/var/lib/mysql/VM_0_14_centos.pid***

**mysql核心目录**

```properties
/var/lib/mysql :mysq1_ 安装目录
/usr/share/mysql:配置文件
/usr/bin :命令目录(mysqladmin , mysqldump等命令)
etc/init.d/mysql :mysql启停脚本
```

#### MySQL配置文件

|     文件名     |        作用        |
| :------------: | :----------------: |
|  my-huge. cnf  | 高端服务器1-2G内存 |
| my-large. cnf  |      中等规模      |
| my-medium. cnf |        一般        |
| my-small. cnf  |        较小        |

**但是，以上配置文件mysq1默认不能识别，默认只能识别/etc/my. cnf**   

采用my-hulge. cnf

```shell
 cp my-huge.cnf /etc/my.cnf
```

***注意: mysq15. 5默认配置文件/etc/my. cnf; Mysql5. 6默认配置文件/etc/ mysql-default. cnf***

#### 查看mysql编码

```shell
 show variables like '%char%';
```

#### 设置编码

在/etc/my.cnf修改

```properties
[mysql]
default-character-set=utf8
[client]
default-character-set=utf8
[mysqld]
character_set_server=utf8
character_set_client=utf8
collation_server=utf8_general_ci
```

重启MySQL

```shell
service mysql restart

```

**配置远程连接**

```sql
grant all privileges on *.* to 'root'@'%' identified by 'gaohwangh922104' with grant option;
flush privileges;
#修改密码
update user set password=password('gaohwangh922104') where user='root' and host='localhost';
flush privileges;

grant all privileges on *.* to 'root'@'%' identified by 'gaohangwanghong' with grant option;
flush privileges;


/sbin/iptables -I INPUT -p tcp --dport 3306 -j ACCEPT
/etc/rc.d/init.d/iptables save
/etc/init.d/iptables restart
/sbin/iptables -L -n

/sbin/iptables -I INPUT -p tcp --dport 3306 -j ACCEPT

firewall-cmd --query-port=3306/tcp
```

**完成!**







## 方法/步骤

 

1.  

   执行firewall-cmd --permanent --zone=public --add-port=3306/tcp，提示FirewallD is not running，如下图所示。

   [![centos出现“FirewallD is not running”怎么办](https://imgsa.baidu.com/exp/w=500/sign=3783b5472134349b74066e85f9eb1521/7dd98d1001e939013391d96372ec54e737d196df.jpg)](http://jingyan.baidu.com/album/5552ef47f509bd518ffbc933.html?picindex=1)

2.  

   通过systemctl status firewalld查看firewalld状态，发现当前是dead状态，即防火墙未开启。

   [![centos出现“FirewallD is not running”怎么办](https://imgsa.baidu.com/exp/w=500/sign=0c6057bedc39b6004dce0fb7d9513526/55e736d12f2eb93882fe2eafdc628535e4dd6fdf.jpg)](http://jingyan.baidu.com/album/5552ef47f509bd518ffbc933.html?picindex=2)

3.  

   通过systemctl start firewalld开启防火墙，没有任何提示即开启成功。

   [![centos出现“FirewallD is not running”怎么办](https://imgsa.baidu.com/exp/w=500/sign=32b9322ef2f2b211e42e854efa816511/e61190ef76c6a7efec9afc7bf4faaf51f3de662a.jpg)](http://jingyan.baidu.com/album/5552ef47f509bd518ffbc933.html?picindex=3)

4.  

   再次通过systemctl status firewalld查看firewalld状态，显示running即已开启了。

   [![centos出现“FirewallD is not running”怎么办](https://imgsa.baidu.com/exp/w=500/sign=79a4d13cf11986184147ef847aec2e69/503d269759ee3d6d9f44d3964a166d224e4adee9.jpg)](http://jingyan.baidu.com/album/5552ef47f509bd518ffbc933.html?picindex=4)

5. 5

   如果要关闭防火墙设置，可能通过systemctl stop firewalld这条指令来关闭该功能。

   [![centos出现“FirewallD is not running”怎么办](https://imgsa.baidu.com/exp/w=500/sign=ce97d76d33f33a879e6d001af65d1018/2e2eb9389b504fc2cecce458ecdde71191ef6ddf.jpg)](http://jingyan.baidu.com/album/5552ef47f509bd518ffbc933.html?picindex=5)

6. 6

   再次执行执行firewall-cmd --permanent --zone=public --add-port=3306/tcp，提示success，表示设置成功，这样就可以继续后面的设置了。

   [![centos出现“FirewallD is not running”怎么办](https://imgsa.baidu.com/exp/w=500/sign=9a524f1e82d4b31cf03c94bbb7d7276f/42166d224f4a20a4969c689a99529822730ed0e9.jpg)](http://jingyan.baidu.com/album/5552ef47f509bd518ffbc933.html?picindex=6)

   END

**firewall-cmd --reload**   # 配置立即生效



2、查看防火墙所有开放的端口

**firewall-cmd --zone=public --list-ports**

 

3.、关闭防火墙

如果要开放的端口太多，嫌麻烦，可以关闭防火墙，安全性自行评估

**systemctl stop firewalld.service**

 

4、查看防火墙状态

 **firewall-cmd --state**

 

5、查看监听的端口

**netstat -lnpt**

![img](https://img2018.cnblogs.com/blog/1336432/201903/1336432-20190302110949754-1765820036.png)

*PS:centos7默认没有 netstat 命令，需要安装 net-tools 工具，yum install -y net-tools*

 

 

6、检查端口被哪个进程占用

**netstat -lnpt |grep 5672**

![img](https://img2018.cnblogs.com/blog/1336432/201903/1336432-20190302104128381-1210567174.png)

 

7、查看进程的详细信息

**ps 6832**

![img](https://img2018.cnblogs.com/blog/1336432/201903/1336432-20190302104342651-779103690.png)

 

8、中止进程

**kill -9 6832**