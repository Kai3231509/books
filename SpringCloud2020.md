### 一、SpringCloud介绍

#### 1.1 微服务架构

> 1. 微服务只是一种样式，一种风格
>
> 2. 将一个完整的项目，拆分成多个模块分别开发
>
> 3. 每一个模块都是单独运行再自己的容器中
>
> 4. 每一个模块都是需要相互通讯的http、RPC、MQ等
>
> 5. 每一个模块之间是没有依赖关系的，可单独部署
>
> 6. 模块与模块间的多语言支持，不同模块通过接口互通
>
> 7. 支持不同的数据库和存储技术，不同模块可以使用多个MySQL数据库，也可使用redis、ES等NoSQL数据库存储数据
>
>    总结：将复杂臃肿的单体应用进行细粒度的划分，每个划分出来的服务各自打包部署

#### 1.2 SpringCloud

> SpringCloud是微服务架构落地的一套技术栈
>
> SpringCloud中的大多数技术都是基于Netflix公司技术的二次开发
>
> 1. SpringCloud的中文社区网站http://springcloud.cn
> 2. SpringCloud中文网http://springcloud.cc
> 3. 八大技术
>    1. Eureka-服务的注册与发现
>    2. Robbin-服务之间的负载均衡
>    3. Feign-服务间的通讯
>    4. Hystrix-服务的线程隔离以及断路器
>    5. Zuul-服务网关
>    6. config-动态配置
>    7. Sluenth-服务的追踪

### 二、服务的注册与发现-Eureka

#### 2.1 引言

> 帮助维护服务的信息，以便服务间的相互调用

![image-20201129202447617](typora-user-images/image-20201129202447617-1606704300872.png)

#### 2.2 Eureka快速入门

##### 2.2.1 创建Eureka-Service

依赖[pom.xml]()

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

配置文件[application.yml]()

```yml
server:
  port: 8762

eureka:
  instance:
    hostname: localhost
  client:
    register-with-eureka: true # 注册到eureka上
    fetch-registry: true # 从eureka上拉取信息,存在多个Eureka时设置为true,单机版设置为false
    service-url:
      defaultZone: http://root:root@localhost:8761/eureka

# 指定用户名和密码
spring:
  security:
    user:
      name: root
      password: root
```

启动类

```java
@SpringBootApplication
@EnableEurekaServer
public class EurakeApplication {
    public static void main(String[] args) {
        SpringApplication.run(EurakeApplication.class, args);
    }
}
```

##### 2.2.2 创建Eureka-Client

依赖[pom.xml]()

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

配置文件[application.yml]()

```yml
eureka:
  client:
    service-url:
      defaultZone: http://root:root@localhost:8761/eureka,http://root:root@localhost:8762/eureka
      
spring:
  application:
    name: MAIL
```

启动类添加注解@EnableEurekaClient

#### 2.3 Eureka的安全性（访问登录及CSRF校验）

导入依赖

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
```

编写配置类（CSRF校验路径忽略eureka路径，避免因安全校验导致微服务无法注册的问题）

```java
@EnableWebSecurity
public class WebSecurityConfig extends WebSecurityConfigurerAdapter {
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        // 忽略掉/eureka/**路径
        http.csrf().ignoringAntMatchers("/eureka/**");
        super.configure(http);
    }
}
```

编写配置文件（登录Eureka的用户名和密码）

```yml
# 指定用户名和密码
spring:
  security:
    user:
      name: root
      password: root
```

#### 2.4 Eureka的高可用

> 如果程序正在运行，突然Eureka宕机
>
> 1. 如果调用方访问过一次被调用方，Eureka宕机不会影响到功能
> 2. 如果调用方没有访问过被调用方，Eureka的宕机会造成当前功能不可用

-------

> 搭建高可用
>
> 1. 准备多台Eureka
> 2. 让服务注册到多台Eureka
> 3. 让多台Eureka之间相互通讯

#### 2.5 Eureka细节

> 1. EurekaClient启动时，将自己的注册信息注册到Eureka-Server上，EurekaServer就会存储上EurekaClient的注册信息
> 2. 当EurekaClient调用服务时，本地没有注册信息的缓存时，去EurekaServer中获取注册信息
> 3. EurekaClient会通过心跳方式去和EurekaServer建立连接(默认30s会发送一次心跳请求，如果超过90s还没有发送心跳信息的话，EurekaServer就任务你宕机了，将当前的EurekaClient从注册表中移除)
> 4. EurekaClient会每隔30s去EurekaServer中去更新本地的注册表缓存信息
> 5. Eureka的自我保护机制，15分钟内，如果一个服务的心跳发送比例低于85%， EurekaServer会开启自我保护机制：
>    1. Eureka不会移除长时间没有收到心跳的服务
>    2. Eureka还是可以正常提供服务的
>    3. 网络比较稳定时，EurekaServer才会将自己的信息被其他节点同步过去

![image-20201129212955264](typora-user-images\image-20201129212955264.png)

### 三、服务间的负载均衡

#### 3.1引言

> Robbin帮助我们实现服务于服务自检的负载均衡
>
> 客户端负载均衡：客户模块将search模块全部拉取到本地的缓存，在Customer中自己做一个负载均衡的策略
>
> 服务端负载均衡：在注册中心中直接根据你选的的负载均衡策略帮助客户端选一个服务模块返回给客户端

![image-20201130155507103](typora-user-images/image-20201130155507103.png)

#### 3.2 快速入门

![image-20201130160719792](typora-user-images/image-20201130160719792.png)

![image-20201130160942233](typora-user-images/image-20201130160942233.png)

#### 3.3 Robbin负载均衡策略

> 负载均衡策略
>
> 1. RandomRule随机策略
> 2. RoundRobbinRule 轮询策略
> 3. WeightedResponseTimeRule 默认会采用轮询的策略，后续会根据服务的响应时间，自动给你分配权限
> 4. BestAvailableRule 根据被调用并发数最小的去分配

![image-20201130161255438](typora-user-images/image-20201130161255438.png)

![image-20201130162007177](typora-user-images/image-20201130162007177.png)

### 四、服务间的调用-Feign

#### 4.1 引言

> Feign可以帮助我们实现面向接口编程，就直接调用其他的服务，简化开发

#### 4.2 Feign的快速入门

<img src="typora-user-images/image-20201130164053684.png" alt="image-20201130164053684"  />

![image-20201130164212451](typora-user-images/image-20201130164212451.png)

#### 4.3 Feign传递参数方式

![image-20201130164524215](typora-user-images/image-20201130164524215.png)

#### 4.4  Feign的FallBack

> FallBack可以帮助我们在使用Feign去调用一个服务时，如果出现了问题，走服务降级，返回一个错误数据，避免因为一个服务出现问题，全部失效

![image-20201130172548387](typora-user-images/image-20201130172548387.png)

![image-20201130172911480](typora-user-images/image-20201130172911480.png)

![image-20201130173354097](typora-user-images/image-20201130173354097.png)

### 五、服务的隔离和断路器-Hystrix

#### 5.1 引言

![image-20201017182514529](typora-user-images\image-20201017182514529.png)

#### 5.2 降级机制实现

1. 导入依赖

   ```xml
   <dependency>
       <groupId>com.netflix.hystrix</groupId>
       <artifactId>hystrix-javanica</artifactId>
   </dependency>
   <dependency>
       <groupId>org.springframework.cloud</groupId>
       <artifactId>spring-cloud-netflix-hystrix</artifactId>
   </dependency>
   ```

   

2. 添加一个注解@EnableCircuitBreaker

3. 针对某一个接口编写它的降级方法

   ```java
   // findById的降级方法，方法描述要和接口一致
   public Customer findByIdFallback(@PathVariable Integer id){
       return new Customer(-1, "", 0);
   }
   ```

   

4. 在接口上添加注解（导入依赖hystrix-javanica）

   ```java
   @HystrixCommand(fallbackMethod = "findByIdFallback")
   ```

   

5. 测试

   ![image-20201017192855645](typora-user-images\image-20201017192855645.png)

#### 5.3 线程隔离

> 如果使用Tomcat线程池去接收用户的请求，使用当前线程去执行其他服务的功能，如果一个服务出现了故障，导致Tomcat线程大量的堆积	，导致Tomcat无法处理其他业务功能。
>
> 解决方案：
>
> 1. Hystrix线程池（默认），接收用户请求用Tomcat线程池，执行业务代码，调用其他请求时用Hytrix线程池
> 2. 信号量，使用的还是Tomcat线程池，帮助我们去管理Tomcat线程池

- Hstrix线程池的配置(具体的配置name属性需要查看HystrixCommandProperties类)

  | 属性含义                             | 属性键                                             | 属性值                             |
  | ------------------------------------ | -------------------------------------------------- | ---------------------------------- |
  | 线程隔离策略                         | ”execution.isolation.strategy“                     | ”HREAD“/”SEMAPHORE“,默认是”THREAD“ |
  | 指定超时时间（针对线程池）           | ”execution.isolation.thread.timeoutInMilliseconds“ | 默认是”1000“（ms）                 |
  | 是否开启超时时间配置（针对线程池）   | ”execution.timeout.enabled“                        | 默认属性是true                     |
  | 超时时候是否中断线程（针对线程池）   | ”execution.isolation.thread.interruptOnTimeout“    | 默认值是true                       |
  | 取消任务后是否中断线程（针对线程池） | ”execution.isolation.thread.interruptOnCancel“     | 默认值为false                      |

  

- 信号量的配置信息

| 属性含义                   | 属性键                                                | 属性值     |
| -------------------------- | ----------------------------------------------------- | ---------- |
| 指定信号量的最大并发请求数 | ”execution.isolation.semaphore.maxConcurrentRequests“ | 默认为“10” |

#### 5.4 断路器

##### 5.4.1 断路器介绍

> 在调用指定服务时，如果说这个服务的失败率达到你输入的一个阈值，将断路器从close状态转变为open状态，指定服务是暂时无法被访问的，如果你访问就走fallback方法，在一定时间内断路器由open状态转变为helf open状态，允许一个请求发送到指定服务，如果成功，断路器转变为closed状态，如果失败则再次转变为open状态，直到断路器转变为closed状态为止。

![image-20201017220702949](typora-user-images\image-20201017220702949.png)

##### 5.4.2 配置断路器的监控界面

1. 导入依赖

   ```xml
   <dependency>
       <groupId>org.springframework.cloud</groupId>
       <artifactId>spring-cloud-starter-netflix-hystrix-dashboard</artifactId>
   </dependency>
   ```

   

2. 启动类中添加一个注解

   ```java
   @EnableHystrixDashboard
   ```

   

3. 配置一个servlet

   ```java
   @WebServlet("/hystrix.stream")
   public class HystrixServlet extends HystrixMetricsStreamServlet {
   }
   
   // ----------------------------------
   // 在启动类上添加扫描servlet的注解
   @ServletComponentScan("com.example.servlet")
   ```

   

4. 测试

   直接访问http://host:port/hystrix

   <img src="typora-user-images\image-20201017230134359.png" alt="image-20201017230134359" style="zoom:50%;" />

   ​	在当前位置输入映射好的servlet路径

##### 5.4.3 配置断路器属性

| 属性含义                        | 属性键                                     | 属性值             |
| ------------------------------- | ------------------------------------------ | ------------------ |
| 指定断路器的开关                | ”circuitBreaker.enable“                    | 默认是true         |
| 触发断路器的最少请求数(10s之内) | ”circuitBreaker.requestVolumeThreshold“    | 默认是”20“         |
| 触发断路器的最低失败率          | ”circuitBreaker.errorThresholdPercentage“  | 默认是“50”         |
| 断路器open状态的持续时长        | ”circuitBreaker.sleepWindowInMilliseconds“ | 默认值是"5000"(ms) |
| 强制让服务拒绝请求              | ”circuitBreaker.forceOpen“                 | 默认值为false      |
| 强制让服务接收请求              | ”circuitBreaker.forceClosed“               | 默认值为false      |

------

具体配置方式

```java
@HystrixCommand(fallbackMethod = "findByIdFallback", commandProperties = {
    @HystrixProperty(name = "execution.isolation.strategy",value = "THREAD"),
    @HystrixProperty(name = "execution.isolation.thread.timeoutInMilliseconds", value = "3000"),
    @HystrixProperty(name = "circuitBreaker.requestVolumeThreshold", value = "10"),
    @HystrixProperty(name = "circuitBreaker.errorThresholdPercentage", value = "70"),
    @HystrixProperty(name = "circuitBreaker.sleepWindowInMilliseconds", value = "5000")
})
```

#### 5.5 请求缓存

##### 5.5.1 请求缓存介绍

> 1. 请求缓存的生命周期是一次请求
> 2. 请求缓存是缓存当前线程中的一个方法，将方法参数作为key，方法返回结果作为value
> 3. 在一次请求中，目标方法被调用过一次，以后就都会被缓存

![image-20201018081339353](typora-user-images\image-20201018081339353.png)

##### 5.5.2 请求缓存的实现

1.  创建一个service，并在service中调用search服务

   ```java
   @Service
   public class CustomerService {
   
       @Autowired
       private SearchClient searchClient;
   
       @CacheResult
       @HystrixCommand(commandKey = "findById")
       public Customer findById(@CacheKey Integer id) {
           return searchClient.findById(id);
       }
   
       @CacheRemove(commandKey = "findById")
       @HystrixCommand
       public void clearFindById(@CacheKey Integer id){
           System.out.println("findById缓存被清空");
       }
   }
   
   ```

   

2. 使用请求缓存的注解

   ```java
   @CacheResult：帮助我们缓存当前方法的返回结果（必须和@HystrixCommand配合使用）
   @CacheRemove：帮助我们清除某一个缓存信息（基于commandKey）
   @CacheKey：指定哪个方法参数作为缓存Key(默认是所有参数)
   ```

   

3. 修改search模块的返回结果

   ```java
   return new Customer(id, "张三",(int) (Math.random()*1000));
   ```

   

4. 编写Filter,去构建HystrixRequestContext对象

   ```java
   @WebFilter("/*")
   public class HystrixRequestContextInitFilter implements Filter {
       @Override
       public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain) throws IOException, ServletException {
           HystrixRequestContext.initializeContext();
           filterChain.doFilter(servletRequest, servletResponse);
       }
   }
   ```

   

5. 修改Controller

   ```java
   public Customer findById(@PathVariable Integer id) throws InterruptedException {
       // Thread.sleep(3000);
       if(id==1) {
           int i = 1/0;
       }
       System.out.println(customerService.findById(id));
       System.out.println(customerService.findById(id));
       customerService.clearFindById(id);
       System.out.println(customerService.findById(id));
       System.out.println(customerService.findById(id));
       return customerService.findById(id);
   }
   
   ```

   

6. 测试结果

   ![image-20201018090743846](typora-user-images\image-20201018090743846.png)

### 六、服务的网关-Zuul

#### 6.1 引言

> 1.  客户端维护大量的ip和port信息，直接访问指定服务
> 2.  认证和授权操作，需要在每一个模块中添加，成本太高
> 3.  项目的迭代，服务的拆分与合并，需要客户端进行大量的变化
> 4.  统一把安全性校验放在Zuul中

![image-20201018091454946](typora-user-images\image-20201018091454946.png)

#### 6.2 Zuul的快速入门

1.  创建maven项目，修改为SpringBoot

2. 导入依赖

   ```xml
   <dependency>
       <groupId>org.springframework.cloud</groupId>
       <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
   </dependency>
   <dependency>
       <groupId>org.springframework.cloud</groupId>
       <artifactId>spring-cloud-starter-netflix-zuul</artifactId>
   </dependency>
   ```

   

3.  添加一个注解

   ```
   @EnableEurekaClient
   @EnableZuulProxy
   ```

   

4.  编写配置文件

   ```yml
   # 指定Eureka服务地址
   eureka:
     client:
       service-url:
         defaultZone: http://root:root@localhost:8761/eureka,http://root:root@localhost:8762/eureka
   
   # 指定服务的名称
   spring:
     application:
       name: ZUUL
   
   server:
     port: 8083
   ```

   

5.  直接测试

   ![image-20201018093916109](typora-user-images\image-20201018093916109.png)

#### 6.3 Zuul中常用配置信息

##### 6.3.1 Zuul的监控界面

1.  导入依赖

   ```xml
   <dependency>
       <groupId>org.springframework.boot</groupId>
       <artifactId>spring-boot-starter-actuator</artifactId>
   </dependency>
   ```

   

2. 编写配置文件

   ```yml
   # 配置Zuul的监控界面（开发时配置为*， 上线时不要配置）
   management:
     endpoint:
     endpoints:
       web:
         exposure:
           include: "*"
   ```

   

3. 直接访问

   ![image-20201018095427043](typora-user-images\image-20201018095427043.png)

##### 6.3.2 忽略服务配置

```yml
zuul:
  # 基于服务名忽略，监控界面查看不到,"*"会忽略掉默认配置的全部路径，自定义服务的配置是无法忽略掉的
  ignored-services: eureka
  # 基于路径忽略， 监控界面仍然看得到，但访问时报404
  ignored-patterns: /**/search/**
```

##### 6.3.3 自定义服务配置

```yml
 # 指定自定义服务访问路径(方式1）
#  routes:
#    search: /ss/**
#    customer: /cc/**

  # 指定自定义服务访问路径(方式2）
  routes:
    kehu:
      path: /ccc/** # 映射路径
      serviceId: customer # 服务名
```

##### 6.3.4 灰度发布

1.  配置不同版本号的映射路径（使用bean的方式）

   ```java
   @Bean
   public PatternServiceRouteMapper serviceRouteMapper() {
       return new PatternServiceRouteMapper(
           "(?<name>^.+)-(?<version>v.+$)",
           "${version}/${name}");
   }
   ```

   

2.  安装指定格式配置服务名称

   ```yml
   # 指定服务的名称
   spring:
     application:
       name: CUSTOMER-${version}
   ```

   

3.  准备一个服务，添加两个版本

   ```java
   @GetMapping("/version")
   public String getVersion(){
       return  serviceVersion;
   }
   ```

   <img src="typora-user-images\image-20201018115945123.png" alt="image-20201018115945123" style="zoom:50%;" />

4.   测试

   ![image-20201018120034486](typora-user-images\image-20201018120034486.png)

   ![image-20201018120105617](typora-user-images\image-20201018120105617.png)

   

#### 6.4 Zuul过滤器的执行流程

> 客户端请求会发送到zuul服务器上，首先通过PreFilter链, 如果正常放行，会把请求转发给RouteFilter,由RouteFilter转发到指定服务上，在指定服务响应一个结果后，会再次走PostFilter过滤器链，最终再将响应信息交给客户端

![image-20201018121220421](typora-user-images\image-20201018121220421.png)



#### 6.5 Zuul过滤器入门

1. 创建一个POJO类，继承ZuulFilter抽象类

   ```java
   public class TestZuulFilter extends ZuulFilter {}
   ```

   

2. 指定当前过滤器的类型

   ```java
   @Override
   public String filterType() {
       return FilterConstants.PRE_TYPE;
   }
   ```

   

3. 指定过滤器的执行顺序

   ```java
   @Override
   public int filterOrder() {
       return FilterConstants.PRE_DECORATION_FILTER_ORDER-1;
   }
   
   ```

   

4. 配置是否启用

   ```java
   @Override
   public boolean shouldFilter() {
       // 开启当前过滤器
       return true;
   }
   ```

   

5. 指定过滤器中的具体业务代码

   ```java
   @Override
   public Object run() throws ZuulException {
       System.out.println("prefix过滤器执行");
       return null;
   }
   ```

   

6. 测试

   ![image-20201018123856171](typora-user-images\image-20201018123856171.png)

#### 6.6 PreFilter实现token校验

1. ​	准备访问路径，请求参数携带token
2. ​    创建AuthenticationFilter
3. ​    在run方法中编写具体的业务逻辑代码
4. ​    测试

#### 6.7 Zuul的降级

1. 创建一个类，实现接口FallbackProvider

   ```java
   public class ZuulFallBack implements FallbackProvider {}
   ```

   

2. 实现接口方法

   ```java
   @Override
   public String getRoute() {
       return "*";
   }
   
   @Override
   public ClientHttpResponse fallbackResponse(String route, Throwable cause) {
       System.out.println("降级的服务" + route);
       cause.printStackTrace();
       return new ClientHttpResponse() {
           @Override
           public HttpStatus getStatusCode() throws IOException {
               return HttpStatus.INTERNAL_SERVER_ERROR;
           }
   
           @Override
           public int getRawStatusCode() throws IOException {
               return HttpStatus.INTERNAL_SERVER_ERROR.value();
           }
   
           @Override
           public String getStatusText() throws IOException {
               return HttpStatus.INTERNAL_SERVER_ERROR.getReasonPhrase();
           }
   
           @Override
           public void close() {
   
           }
   
           @Override
           public InputStream getBody() throws IOException {
               // 给用户响应信息
               String msg = "当前服务:"+ route + "出现问题";
               return new ByteArrayInputStream(msg.getBytes());
           }
   
           @Override
           public HttpHeaders getHeaders() {
               // 指定响应头信息
               HttpHeaders headers = new HttpHeaders();
               headers.setContentType(MediaType.APPLICATION_JSON);
               return headers;
           }
       };
   }
   ```

   

3. 测试

   ![image-20201018202227989](typora-user-images\image-20201018202227989.png)

#### 6.8 Zuul动态路由

1. 创建一个过滤器 (执行顺序最好放在Pre过滤器的最后面)

2. 在run方法中编写业务逻辑

   ```java
   @Override
   public Object run() throws ZuulException {
       // 1.获取Request对象
       RequestContext context = RequestContext.getCurrentContext();
       HttpServletRequest request = context.getRequest();
   
       // 2.获取参数redisKey
       String redisKey = request.getParameter("redisKey");
   
       // 3.直接判断
       if(redisKey != null && redisKey.equalsIgnoreCase("customer")){
           // http://localhost:8080/customer
           context.put(FilterConstants.SERVICE_ID_KEY, "customer-v1");
           context.put(FilterConstants.REQUEST_URI_KEY, "/customer");
       } else if("search".equalsIgnoreCase(redisKey)){
           // http://localhost:8080/search/1
           context.put(FilterConstants.SERVICE_ID_KEY, "search");
           context.put(FilterConstants.REQUEST_URI_KEY, "/search/1");
       }
       return null;
   }
   ```

   

3. 测试

   ![image-20201018205236694](typora-user-images\image-20201018205236694.png)



### 七、多语言支持-Sidecar

#### 7.1 引言

> 在SpringCloud的项目中，需要接入一下非JAVA程序和第三方接口，无法接入eureka、hystrix、feign等等组件、启动一个代理的微服务，代理微服务去和非java程序和第三方接口交流，通过代理的微服务去计入SpringCloud的相关组件。

![image-20201019090115755](typora-user-images\image-20201019090115755.png)



#### 7.2 Sidecar的实现

1. 创建一个第三方服务

   ```python
   # -*- coding:utf-8 -*-
   
   from flask import Flask
   
   app = Flask(__name__)
   
   @app.route("/index")
   def hello():
       return "hello world!"
   
   app.run(host="0.0.0.0", port=8888)
   ```

   

2. 创建Maven工程，修改为SpringBoot

3. 导入依赖

   ```xml
   <dependency>
       <groupId>org.springframework.cloud</groupId>
       <artifactId>spring-cloud-netflix-sidecar</artifactId>
   </dependency>
   <dependency>
       <groupId>org.springframework.cloud</groupId>
       <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
   </dependency>
   ```

   

4. 添加注解

   ```java
   @SpringBootApplication
   @EnableSidecar
   public class SidecarApplication {
   
       public static void main(String[] args) {
           SpringApplication.run(SidecarApplication.class, args);
       }
   }
   ```

   

5. 编写配置文件

   ```yml
   server:
     port: 8090
   
   # 指定Eureka服务地址
   eureka:
     client:
       service-url:
         defaultZone: http://root:root@localhost:8761/eureka,http://root:root@localhost:8762/eureka
   
   spring:
     application:
       name: other-service
   
   sidecar:
     port: 8888
   ```

   

6. 通过customer调用第三方服务

   ```java
   @FeignClient("OTHER-SERVICE")
   public interface OtherServiceClient {
   
       @GetMapping("/index")
       String hello();
   
   }
   
   // -------------------------
   
   @Autowired
   private OtherServiceClient otherServiceClient;
   
   @GetMapping("/index")
   public String hello(){
       return otherServiceClient.hello();
   }
   ```

   

7. 测试

   ![image-20201019145848388](typora-user-images\image-20201019145848388.png)

### 八、服务间的消息传递-Stream

#### 8.1 引言

> 为各服务提供消息队列的统一的接口，屏蔽底层细节

![image-20201031225507838](typora-user-images\image-20201031225507838.png)

#### 8.2 Stream快速入门

> 1. 启动	RabbitMQ

> 2. 消费者-导入依赖
>
>    ```xml
>    <dependency>
>        <groupId>org.springframework.cloud</groupId>
>        <artifactId>spring-cloud-starter-stream-rabbit</artifactId>
>    </dependency>
>    ```

> 3. 消费者-编写配置文件
>
> ```yml
> rabbitmq:
>   host: 180.102.132.142
>   port: 5672
>   username: test
>   password: test
>   virtual-host: /test
> ```
>
> 

> 4. 消费者-监听的队列
>
>    ```java
>    public interface StreamClient {
>    
>        @Input("my")
>        SubscribableChannel input();
>    }
>    
>    // ------------------------------------------------
>    
>    @Component
>    @EnableBinding(StreamClient.class)
>    public class StreamReceiver {
>    
>        @StreamListener("my")
>        public void msg(Object msg){
>            System.out.println("收到的消息"+msg);
>        }
>    }
>    ```
>
>    

> 5. 生产者-导入依赖

> 6. 生产者-编写配置文件

> 7.  生产者-发布消息
>
> ```java
> public interface StreamClient {
> 
>     @Output("my")
>     MessageChannel output();
> 
> }
> 
> //------------------------------------------------------------
> 
> @RestController
> public class MessageController {
> 
>     @Autowired
>     private StreamClient streamClient;
> 
>     @GetMapping("/send")
>     public String send(){
>         streamClient.output().send(MessageBuilder.withPayload("hello").build());
>         return "消息发送成功";
>     }
> }
> 
> //-------------------------------------------------------------
> // 启动类上添加注解
> @EnableBinding(StreamClient.class)
> 
> ```

#### 8.3 Stream重复消费问题

> 只需要添加一个配置去指定消费者组，就可以了
>
> ```yml
> spring:
>   cloud:
>     stream:
>       bindings:
>         my: 
>           group: group-customer  #指定消费者组的名称
> ```

#### 8.4 Stream消费者手动ack

> 1. ​	编写配置
>
> ```yml
> spring:
>   cloud:
>       # 实现手动ack
>       rabbit:
>         bindings:
>           my:
>             consumer:
>               acknowledge-mode: MANUAL
> ```

> 2. 修改消费端方法
>
> ```java
> @StreamListener("my")
> public void msg(Object msg,
>                 @Header(name = AmqpHeaders.CHANNEL) Channel channel,
>                 @Header(name=AmqpHeaders.DELIVERY_TAG) Long tag) throws IOException {
>     System.out.println("收到的消息"+msg);
>     channel.basicAck(tag, false);
> }
> ```

### 九、服务的动态配置-Config

#### 9.1 引言

> 1. 配置文件分布在不同的项目中，不方便维护
> 2. 配置文件的安全性无法保证
> 3. 配置文件修改后无法立即生效，必须重启项目

![image-20201020092412470](typora-user-images\image-20201020092412470.png)

> **请注意：远程仓库和本地仓库必须是一一对应的关系，不能混用，否则配置文件访问不到。**

#### 9.2 搭建Config-Server

> 1. 创建Maven工程，修改为SpringBoot

> 2. 导入依赖
>
> ```xml
> <dependency>
>     <groupId>org.springframework.cloud</groupId>
>     <artifactId>spring-cloud-config-server</artifactId>
> </dependency>
> <dependency>
>     <groupId>org.springframework.cloud</groupId>
>     <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
> </dependency>
> ```

> 3. 添加注解
>
> ```java
> @SpringBootApplication
> @EnableConfigServer
> public class ConfigApplication {
> 
>     public static void main(String[] args) {
>         SpringApplication.run(ConfigApplication.class, args);
>     }
> }
> ```

> 4.  编写配置文件(git的操作)
>
>    ```yml
>    # 指定Eureka服务地址
>    eureka:
>      client:
>        service-url:
>          defaultZone: http://localhost:8761/eureka,http://root:root@localhost:8762/eureka
>
>    # 指定服务的名称
>    spring:
>      application:
>        name: CONFIG
>
>      cloud:
>        config:
>          server:
>            git:
>              basedir: D:\baseDir   #本地仓库地址
>              username: 15342603509  # 远程仓库的用户名
>              password: 231509lkm    #远程仓库的密码
>              uri: https://gitee.com/lu-kaiming/config-test.git         #远程仓库地址
>    ```
>
> 
>
>    # 设置端口号
>    server:
>      port: 8090
>
>    ```
> 
>    ```

> 5. 测试
>
>    /{label}/{application}-{profile}.yml
>
>    e.g:    /master/customer-v.yml
>
>    ![image-20201101101324775](typora-user-images\image-20201101101324775.png)

#### 9.3 修改Customer连接Config

> 1. 导入依赖
>
> ```xml
> <dependency>
>     <groupId>org.springframework.cloud</groupId>
>     <artifactId>spring-cloud-config-client</artifactId>
> </dependency>
> ```

> 2. 编写配置文件
>
> ```yml
> spring:
>   cloud:
>     config:
>       discovery:
>         enabled: true
>         service-id: CONFIG
>       profile: dev
> ```

> 3. 修改配置文件
>
> 配置文件更名为bootstrap.yml
>
> 将部分配置放在远程git仓库上

> 4. 测试

#### 9.4 动态配置

##### 9.4.1 实现原理

![image-20201101121903176](typora-user-images\image-20201101121903176.png)

##### 9.4.2 服务连接RabbitMQ

> 1. 导入依赖
>
> ```xml
> <dependency>
>     <groupId>org.springframework.cloud</groupId>
>     <artifactId>spring-cloud-starter-bus-amqp</artifactId>
> </dependency>
> ```

> 2. 编写配置文件连接RabbitMQ信息
>
> ```yml
> spring:
>   rabbitmq:
>     host: 180.102.132.142
>     port: 5672
>     username: test
>     password: test
>     virtual-host: /test
> ```

##### 9.4.3 实现手动刷新

> 1. 导入依赖
>
> ```xml
> <dependency>
>     <groupId>org.springframework.boot</groupId>
>     <artifactId>spring-boot-starter-actuator</artifactId>
> </dependency>
> ```

> 2. 编写配置文件
>
> ```yml
> management:
>   endpoints:
>     web:
>       exposure:
>         include: "*"
> ```

> 3. 为customer添加一个controller
>
> ```java
> @RefreshScope
> public class CustomerController {
> 
>     @Value("${env}")
>     private String env;
> 
> 
>     @GetMapping("/env")
>     public String getEnv(){
>         return env;
>     }
> }
> ```

> 4.  测试
>
>    CONFIG在GITEE修改之后就自动拉取最新的配置信息
>
>    其他模块更新的话，需要手动发送POST请求http://ip:port/actuator/bus-refresh, 不重启项目即可获取最新的配置信息（注：ip和port是config服务的ip和port）
>
>    ```bash
>    curl http://localhost:8090/actuator/bus-refresh -X POST
>    ```

##### 9.4.4 内网穿透

> 1、在natapp 官网开通一条隧道，配置映射

> 2、下载客户端

> 3、在natapp.exe同级目录下编写配置文件config.ini

```ini
#将本文件放置于natapp同级目录 程序将读取 [default] 段
#在命令行参数模式如 natapp -authtoken=xxx 等相同参数将会覆盖掉此配置
#命令行参数 -config= 可以指定任意config.ini文件
[default]
authtoken=7ef05d7e1c8b0fde  #对应一条隧道的authtoken
clienttoken=                    #对应客户端的clienttoken,将会忽略authtoken,若无请留空,
log=none                        #log 日志文件,可指定本地文件, none=不做记录,stdout=直接屏幕输出 ,默认为none
loglevel=ERROR                  #日志等级 DEBUG, INFO, WARNING, ERROR 默认为 DEBUG
http_proxy=                     #代理设置 如 http://10.123.10.10:3128 非代理上网用户请务必留空
```

> 4、启动natapp.exe，本地指定端口会被一个分配的域名映射，并可通过公网访问。

##### 9.4.5 自动刷新

> 1、配置git远程仓库中的WorkHooks
>
> ![image-20201220150853396](typora-user-images/image-20201220150853396.png)

> 2、给Config添加一个过滤器，将git仓库发送POST请求时默认携带的数据过滤掉
>
> ```java
> @WebFilter(urlPatterns = "/actuator/bus-refresh")
> public class UrlFilter implements Filter {
>     @Override
>     public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain) throws IOException, ServletException {
>         HttpServletRequest httpServletRequest= (HttpServletRequest) servletRequest;
>         String url=httpServletRequest.getRequestURI();
>         System.out.println(url);
>         String body=(httpServletRequest).toString();
>         System.out.println("original body: "+ body);
>         RequestWrapper requestWrapper=new RequestWrapper(httpServletRequest);
>         filterChain.doFilter(requestWrapper,servletResponse);
>     }
> 
>     private class RequestWrapper extends HttpServletRequestWrapper {
>         public RequestWrapper(HttpServletRequest request) {
>             super(request);
>         }
> 
>         @Override
>         public ServletInputStream getInputStream() throws IOException {
>             byte[] bytes = new byte[0];
>             ByteArrayInputStream byteArrayInputStream = new ByteArrayInputStream(bytes);
>             ServletInputStream servletInputStream = new ServletInputStream() {
>                 @Override
>                 public int read() throws IOException {
>                     return byteArrayInputStream.read();
>                 }
> 
>                 @Override
>                 public boolean isFinished() {
>                     return byteArrayInputStream.read() == -1 ? true : false;
>                 }
> 
>                 @Override
>                 public boolean isReady() {
>                     return false;
>                 }
> 
>                 @Override
>                 public void setReadListener(ReadListener listener) {
> 
>                 }
>             };
>             return servletInputStream;
>         }
>     }
> }
> ```
>
> 

> 3、测试

### 十、服务的追踪Sleuth	

#### 10.1 引言

> 在整个微服务架构中，微服务很多，一个请求可能需要调用很多很多服务，最终才能完成一个功能，如果说整个功能出现了问题，在那么多服务中如何定位问题的所在点，出现问题的原因是什么。
>
> 1. Sleuth本身可以获得整个服务链路的信息。
> 2. Zipkin通过图形化界面看到信息。
> 3. Sleuth将日志信息存储到数据库中。

![image-20201101214351478](typora-user-images\image-20201101214351478.png)

#### 10.2 Sleuth的使用

> 导入依赖
>
> ```xml
> <dependency>
>     <groupId>org.springframework.cloud</groupId>
>     <artifactId>spring-cloud-starter-sleuth</artifactId>
> </dependency>
> ```

> 编写配置文件
>
> ```yml
> logging:
>   level:
>     org.springframework.web.servlet.DispatcherServlet: DEBUG
> ```
>
> 

> 测试