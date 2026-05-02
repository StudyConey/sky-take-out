# 外卖系统



## B端





## C端









## 重疑难点

### 1. Nginx

好处：

- 提高访问速度
- 进行负载均衡：大量请求按照指定的方式均衡的分配给集群中的每台服务器
- 保护后端服务安全

反向代理：

前端发送的**动态请求**由 *Nginx* **转发**到后端服务器中

前端网址：

```url
http://localhost/api/employee/login
```

后端网址：

```
http://localhos:8080/admin/employee/login
```

配置反向代理：

```nginx
server{
    listen 80;
    server_name localhost;
    
    location /api/{
        proxy_pass http://localhost:8080/admin/; #反向代理
    }
}
```

配置负载均衡：

```nginx
upstream webservers{
    server 192.168.100.128:8080;
    server 192.168.100.129:8080;
}
server{
    listen 80;
    server_name localhost;
    
    location /api/{
        proxy_pass http://webservers/admin;#负载均衡
    }
}
```

