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



### 2. ThreadLocaL

ThreadLocaL 为每个线程提供单独一份存储空间，具有线程**隔离**效果

常用方法：

``` java
public void set(T value) // 设置当前线程局部变量的值
public T get() //返回当前线程所对应的线程局部变量的值
public void remove() //移除当前线程的线程局部变量
```
### 3. 时间格式

方式1：

在属性上添加注解

```java
@JsonFormat(pattern = "yyyy-MM-dd HH:mm:ss")
private LocalDateTime createTime;
```

方式2：

在 ***WebMvcConfiguration*** 中拓展Spring MVC的消息转化器，统一对时间格式进行格式化处理

```java
/**
 * 拓展消息转化器
 * 日期类型格式化
 * @param converters
 */
@Override
protected void extendMessageConverters(List<HttpMessageConverter<?>> converters) {
    log.info("扩展消息转换器");
    //创建一个消息转换器对线
    MappingJackson2HttpMessageConverter converter = new MappingJackson2HttpMessageConverter();
    //设置一个对象转换器 -- 将Java对象序列化为json格式
    converter.setObjectMapper(new JacksonObjectMapper());
    converters.add(0, converter);
}
```



### 4. 提取公共部分

- 自定义注解 ***@AutoFill***，用于标识需要进行公共字段自动填充
- 自定义切面类 ***AutoFilAspect*** ， 统一垃拦截加入 ***AutoFill*** 注解方法，通过反射为公共字段赋值
- 在 Mapper 中方法上加入 ***AutoFill*** 注解

#### 4.1 加创建注解

枚举 *OperationType.java*

```java
public enum OperationType {
    /**
     * 更新操作
     */
    UPDATE,

    /**
     * 插入操作
     */
    INSERT
}
```

注解类

```java
package com.sky.annotation;
import com.sky.enumeration.OperationType;
import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Target(ElementType.METHOD) //作用于方法
@Retention(RetentionPolicy.RUNTIME)
public @interface AutoFill {
    //枚举:UPDATE INSERT
    OperationType value();
}
```



#### 4.2 切面类

```java  
package com.sky.aspect;

import com.sky.annotation.AutoFill;
import com.sky.context.BaseContext;
import com.sky.enumeration.OperationType;
import lombok.extern.slf4j.Slf4j;
import org.aspectj.lang.JoinPoint;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Before;
import org.aspectj.lang.annotation.Pointcut;
import org.aspectj.lang.reflect.MethodSignature;
import org.springframework.stereotype.Component;

import java.lang.reflect.Method;
import java.time.LocalDateTime;

@Aspect
@Component
@Slf4j
public class AutoFillAspect {

    //切入点
    @Pointcut("execution(* com.sky.mapper.*.*(..)) && @annotation(com.sky.annotation.AutoFill)")
    public void autoFillPointCut() {

    }

    //前置通知 : 在通知中进行公共字段的赋值
    @Before("autoFillPointCut()")
    public void autoFill(JoinPoint joinPoint) {
        log.info("开始进行公共字段的填充...");

        //获取当前被拦截的方法上的数据库操作类型
        MethodSignature signature = (MethodSignature) joinPoint.getSignature(); //MethodSignature方法签名
        AutoFill autoFill = signature.getMethod().getAnnotation(AutoFill.class); //获得方法上的注解对象
        OperationType operationType = autoFill.value();//数据库操作类型

        //获取当前被拦截的方法的参数 -- 实体对象
        Object[] args = joinPoint.getArgs();
        if (args != null || args.length == 0) {
            return;
        }
        Object entity = args[0];

        //准备赋值的数据
        LocalDateTime now = LocalDateTime.now();
        Long currentId = BaseContext.getCurrentId();

        //根据当前不同的操作类型，为对应的数值通过反射来赋值
        if (operationType == OperationType.INSERT) {
            try {
                Method setCreateTime = entity.getClass().getDeclaredMethod("setCreateTime", LocalDateTime.class);
                Method setCreateUser = entity.getClass().getDeclaredMethod("setCreateUser", Long.class);
                Method setUpdateTime = entity.getClass().getDeclaredMethod("setUpdateTime", LocalDateTime.class);
                Method setUpdateUser = entity.getClass().getDeclaredMethod("setUpdateUser", Long.class);

                //通过反射为对象属性进行赋值

                setCreateTime.invoke(entity, now);
                setCreateUser.invoke(entity, currentId);
                setUpdateTime.invoke(entity, now);
                setUpdateUser.invoke(entity, currentId);
            } catch (Exception e) {
                throw new RuntimeException(e);
            }
        } else if (operationType == OperationType.UPDATE) {
            try {
                Method setUpdateTime = entity.getClass().getDeclaredMethod("setUpdateTime", LocalDateTime.class);
                Method setUpdateUser = entity.getClass().getDeclaredMethod("setUpdateUser", Long.class);
                //通过反射为对象属性进行赋值
                setUpdateTime.invoke(entity, now);
                setUpdateUser.invoke(entity, currentId);
            } catch (Exception e) {
                throw new RuntimeException(e);
            }
        }
    }
}
```