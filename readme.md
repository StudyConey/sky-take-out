# 外卖系统



## B端

### 员工类

```java
//对象属性拷贝
BeanUtils.copyProperties(employeeDTO, employee);
```



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

**nginx 负载均衡策略：**

| **名称**   | **说明**                                               |
| ---------- | ------------------------------------------------------ |
| 轮询       | 默认方式                                               |
| weight     | 权重方式，默认为1，权重越高，被分配的客户端请求就越多  |
| ip_hash    | 依据ip分配方式，这样每个访客可以固定访问一个后端服务   |
| least_conn | 依据最少连接方式，把请求优先分配给连接数少的后端服务   |
| url_hash   | 依据url分配方式，这样相同的url会被分配到同一个后端服务 |
| fair       | 依据响应时间方式，响应时间短的服务将会被优先分配       |
