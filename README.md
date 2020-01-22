# Spring Boot Starter Netty Web

NettyWeb是我**个人自用**的一款基于SpringBoot、Netty开发的轻量级Web API框架，最大的特点就是代码量少、实现简单，以稍微妥协易用性为代价**尽量少的使用反射**。

> 个人自用指的是，所有的私人项目和当公司不强制绑定技术栈时我都会使用。



NettyWeb实现了一个Web框架所应该具有的最基本功能：

- 请求路由
- 参数绑定、验证
- 为每一个请求自动生成唯一id, 方便日志统计

同时，作为一个API框架，NettyWeb仅支持返回**固定的JSON格式**的数据，不具备页面渲染功能。



## 快速开始

在常规的SpringBoot2.x项目中加入以下依赖便可随Spring启动:

```xml
<dependency>
    <groupId>cn.wanghongfei</groupId>
    <artifactId>spring-boot-starter-nettyweb</artifactId>
    <version>1.1-SNAPSHOT</version>
</dependency>
```

> 未发布到中央仓库



### 定义请求处理器

为了避免使用反射，NettyWeb没有像SpringMVC那样支持将任意方法指定为请求处理器，而是必须定义一个类实现`RequestHandler`接口，同时加上`@HttpApi`注解，如：

```java
@HttpApi(path = "/", paramType = DemoRequest.class, method = "POST")
public class DemoApi implements RequestHandler<DemoRequest, String> {
    @Override
    public String serveRequest(DemoRequest request) {
        return request.toString();
    }
}

```

在注解`@HttpApi`中：

- path: 请求路径。这里是精确匹配，不支持通配符;
- paramType: 参数对象类型。NettyWeb会自动将参数值绑定到对象的字段中; 如果此接口不需要参数，可以不赋值或者赋值为`Void.class`；
- method: 请求方法，只支持`GET`, `POST`两个取值；

`serveRequest()`是请求处理的入口方法，`DemoRequest`是用户自定义的参数对象，框架会自动将请求参数转换成此对象。

当请求方法为`GET`时， NettyWeb会从URL里的`Query String`中解析参数并**忽略请求体**；如果是`POST`请求则会将请求体视为JSON并将其反序列化为参数对象。

响应的JSON格式是固定的，如下：

```json
{
    "code": 0,
    "data": "this is data",
    "message": "ok"
}
```

请求处理方法返回的业务数据对象会在`data`中体现，如果返回String, 则`data`的值是string类型，如果返回复杂对象，则`data`的值是object类型。



### 修改响应Header

如果需要修改响应头，则需要实现`RequestHander`的`default`方法`modifyHeader`：

```json
@HttpApi(path = "/", paramType = DemoRequest.class, method = "POST")
public class DemoApi implements RequestHandler<DemoRequest, String> {
    @Override
    public String serveRequest(DemoRequest request) {
        return request.toString();
    }
    
    @Override
    public void modifyHeader(HttpHeaders headers, String data) {
        headers.add("My-Header", data);
    }
}
```





### 特定参数注入

为了避免使用反射，NettyWeb通过探测参数对象是否实现了指定接口来"注入"一些特定参数，如完整的请求头和请求唯一id。

如果需要获取框架给当前请求自动生成的唯一id, 可以让参数对象实现`InjectRequestId`接口：

```json
public class DemoRequest implements InjectRequestId {
    @JSONField(serialize = false, deserialize = false)
    private Long requestId;
    
    @Override
    public Long getRequestId() {
        return requestId;
    }

    @Override
    public void setRequestId(Long reqId) {
        this.requestId = reqId;
    }
}
```

框架会调用`setRequestId(Long)`将id设置到参数对象中。如果嫌麻烦，可以直接继承`RequestIdVO`：

```java
public class DemoRequest extends RequestIdVO {
}
```

就不需要写前面一堆getXXX, setXXX了。

如果想获取请求头，可以实现`InjectHeaders`接口:

```java
public class DemoRequest implements InjectHeaders {
    @JSONField(serialize = false, deserialize = false)
    private Map<String, String> headerMap;

    @Override
    public Map<String, String> headerMap() {
        return headerMap;
    }

    @Override
    public void setHeaderMap(Map<String, String> map) {
        this.headerMap = map;
    }
}

```

同样，如果嫌麻烦，可以直接继承`RequestHeaderVO`类：

```json
public class DemoRequest extends RequestHeaderVO {
}
```



### 参数验证

这块如果不用反射会大大降低易用性，所以该反射还得反射。NettyWeb支持注解声明式的参数验证，如：

```java
@Validation // 启用注解验证
@Data
public class DemoRequest {
    @StringLen(min = 1,  max = 2, message = "name长度1~2")
    private String name;

    @StringCandidate(candidates = {"boy", "girl"}, message = "boy/girl")
    private String gender;
}
```

其中`@StringLen`表示`name`字段：

- 不为null
- 长度>=1, 且<=2
- 当不满足要求时将message作为响应的一部分返回

`@StringCandidate`表示`gender`字段：

- 不为null
- 值只能在`boy`或`girl`中二选一
- 当不满足要求时将message作为响应的一部分返回



NettyWeb支持的全部验证注解有：

- @NutNull
- @NumberSize(Short, Integer, Long的取值范围限制)
- @CollectionNotEmpty
- @StringCandidate
- @StringLen
- @StringReg(匹配正则表达式)

如果参数对象的字段中还嵌套了一个复杂类型，NettyWeb同样支持递归验证，只需要在这个复杂字段上再次添加`@Validation`注解即可：

```java
@Validation // 启用注解验证
@Data
public class DemoRequest {    
    @NotNull
    @Validation
    private UserInfo userInfo;
}
```



## 配置项

```java
public class NettyWebProp {
    /**
     * netty boss线程数量
     */
    private Integer bossGroupThreadCount = 2;
    /**
     * netty worker线程数量
     */
    private Integer workGroupThreadCount = 0;

    /**
     * http请求最大字节数
     */
    private Integer httpObjectMaxSize = 1024 * 1024 * 5;
    /**
     * 用于标识登录状态的header名
     */
    private String loginTokenHeaderName = "Login-Token";

    /**
     * 业务线程池CoreSize
     */
    private Integer servicePoolCoreSize = 5;
    /**
     * 业务线程池MaxSize
     */
    private Integer servicePoolMaxSize = 10;
    /**
     * 业务线程池任务队列最大长度
     */
    private Integer servicePoolQueueSize = 50;

    /**
     * 是否启动Server
     */
    private Boolean startWebServer = true;
}

```



## 单元测试

如果在跑测试的时候不希望启动HTTP Server, 可以在配置文件中把`nettyweb.startwebServer`设为false:

```yaml
nettyweb:
  start-web-server: false
```

