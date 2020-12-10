# 1.nginx负载均衡配置

```properties
   upstream hongLove{
      ip_hash;   #避免session冲突
      server 127.0.0.1:8080 weight=1;   #设置权重
      server 127.0.0.1:8081 weight=2;   
    }
    server {                                     
        listen       80;         #映射的端口          
	server_name  39.100.11.163;    #外面访问的地址           
             
                                                 
           
	
	location / {
	   proxy_pass http://hongLove;
	   proxy_redirect default;
        }
    }   
```

