# nginx相关配置

### 反向代理

```properties
 upstream hongLove{
	server 94.191.23.136:8080;
    }
    server {                                     
        listen       80;                    
	server_name  www.gaohwang.cn;                  
             
                                                 
        location / {   
	    proxy_pass http://hongLove;         
            index  love.html;         
        }                                        
    }     
```

### 静态文件访问

**在server**

```properties
server {
		listen       81;
        server_name  localhost;

        location / {
            root   /student/staticPage/ccps_7_H-ui.admin.page_3.0/;
            index  index.html index.htm;
        }
  }
        
        
        /student/staticPage/ccps_7_H-ui.admin.page_3.0
```

```properties
 upstream assets_management{
	server 127.0.0.1:8080;
    }
    server {                                     
        listen       80;                    
	server_name  www.zccs.com;                  
             
                                                 
        location / {   
	    proxy_pass http://assets_management;         
            index  /jsp/login/index.jsp;         
        }                                        
    }    
    
    
    
     upstream assets_management{
	server http://localhost:8080;
    }
    server {                                     
        listen       80;                    
	server_name  www.zccs.com;                  
             
                                                 
        location / {   
	    proxy_pass http://assets_management;         
            index  /jsp/login/index.jsp;         
        }                                        
    }    
```



