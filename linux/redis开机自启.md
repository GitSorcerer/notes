cd /opt/redis-4.0.2
vim redis.conf

```
daemonize yes # no改成yes
```

vim /etc/init.d/redis 

```properties
# chkconfig: 2345 20 80
# description: Start and Stop redis

PATH=/usr/local/bin:/sbin:/usr/bin:/bin
REDISPORT=6379
# 自己的redis-server路径(需要自己更改)
EXEC=/usr/local/redis-5.0.5/bin/redis-server
# 自己的redis-cli路径(需要自己更改)
REDIS_CLI=/usr/local/redis-5.0.5/bin/redis-cli                                                                                   

PIDFILE=/var/run/redis.pid
# 自己的redis.conf 路径(需要自己更改)
CONF="/usr/local/redis-5.0.5/etc/redis.conf"                                                                                         
AUTH="1234"                                                                                                                  
                                                                                                                           
case "$1" in                                                                                                               
        start)                                                                                                             
                if [ -f $PIDFILE ]                                                                                         
                then                                                                                                       
                        echo "$PIDFILE exists, process is already running or crashed."                                     
                else                                                                                                       
                        echo "Starting Redis server..."                                                                    
                        $EXEC $CONF                                                                                        
                fi                                                                                                         
                if [ "$?"="0" ]                                                                                            
                then                                                                                                       
                        echo "Redis is running..."                                                                         
                fi                                                                                                         
                ;;                                                                                                         
        stop)                                                                                                              
                if [ ! -f $PIDFILE ]                                                                                       
                then                                                                                                       
                        echo "$PIDFILE exists, process is not running."                                                    
                else                                                                                                       
                        PID=$(cat $PIDFILE)                                                                                
                        echo "Stopping..."                                                                                 
                       $REDIS_CLI -p $REDISPORT  SHUTDOWN                                                                  
                        sleep 2                                                                                            
                       while [ -x $PIDFILE ]                                                                               
                       do                                                                                                  
                                echo "Waiting for Redis to shutdown..."                                                    
                               sleep 1                                                                                     
                        done                                                                                               
                        echo "Redis stopped"                                                                               
                fi                                                                                                         
                ;;                                                                                                         
        restart|force-reload)                                                                                              
                ${0} stop                                                                                                  
                ${0} start                                                                                                 
                ;;                                                                                                         
        *)                                                                                                                 
               echo "Usage: /etc/init.d/redis {start|stop|restart|force-reload}" >&2                                       
                exit 1                                                                                                     
esac
```

> 执行权限

```
chmod +x /etc/init.d/redis
1
```

> 尝试启动或停止redis

```
service redis start
service redis stop
12
```

> 开启服务自启动

```
chkconfig redis on
```