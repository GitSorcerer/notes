# Vue项目部署404问题

```nginx
 server {
    listen       8090; 
    server_name  39.100.11.163;
    root D:/business-super-management/sources/app/;
    index index.html;
  
    location / {
      try_files $uri $uri/ @router;
      index index.html;
    }
    location @router {
      rewrite ^.*$ /index.html last;
    }
	} 
```

--spring.profiles.active=test