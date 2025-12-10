### 1. GatewayApplication
启动类，SpringBoot 非阻塞API。
@EnableDiscoveryClient 用于开启服务注册与发现功能，在服务启动时，作为客户端注册到服务注册中心。服务发现: 开启后，这个服务也具备了发现其他已注册服务的能力。当它需要调用其他服务时，可以通过服务注册中心获取到目标服务的实例列表。
这是一个抽象标识，背后是 DiscoveryClient 接口，具体的注册中心是Nacos, Eureka还是其他都是支持的，后者是这个接口的具体实现。
@EnableFeignClients 用于开启 OpenFeign 客户端扫描功能。扫描 Feign 接口: Spring Cloud 应该扫描应用包路径下所有带有 @FeignClient 注解的接口。

OpenFeign 的本质和更广义的定位是：一个优秀的、声明式的 HTTP 客户端 (HTTP Client)，就是将一个 Java 接口 及其方法调用，映射和转换 成一个标准的 HTTP 请求。
Declarative REST Client: Feign creates a dynamic implementation of an interface decorated with JAX-RS or Spring MVC annotations.
所以说来说去Feign 是一个REST Client 而不是RPC Stub，我在项目里没有发现rpc starter类似的依赖，而Client类都是@FeignClient，我怀疑项目里的微服务间通讯是REST而不是RPC。
反推基础：@FeignClient说明走的REST HTTP，如果有proto文件则说明走了RPC

### 2. RequestValidationFilter
org.springframework.core.Ordered接口 定义控制组件的执行顺序，值越小越先执行。值的范围是整型MIN - MAX。
在 Spring Cloud Gateway 中，如果两个过滤器的 Order 值相同，它们的先后顺序取决于 Bean 的加载顺序（通常是不确定的）

GlobalFilter接口 是 Spring Cloud Gateway 定义的全局过滤器接口。
Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain);
当一个请求进入 Gateway 时，它会通过一个 GatewayFilterChain（过滤器链）。所有实现了 GlobalFilter 的 Bean 都会被添加到这个链中。
当请求到达时，接口的filter() 方法被调用。 在方法内部，可以处理请求（例如进行校验），然后调用 chain.filter(exchange) 将请求向下传递给下一个过滤器或最终的微服务。

### 3. ResponseValidationFilter
注意，其实每一个GlobalFilter实现类都有pre步骤和post阶段，所以理论上 请求和响应的filter可以写在一个类里面，但是为了业务清晰，这个项目是分开实现的。
这两个正好把pre 和post 阶段拆开了。RequestValidationFilter 主要修改请求头，ResponseValidationFilter 主要修改响应体，它们操作的对象互不冲突。
虽然order值相同，业务上不影响。

这两个过滤器的执行顺序：请求进 -> 过滤器 Pre 逻辑 -> 微服务 -> 响应出 -> 过滤器 Post 逻辑。
这两个过滤器不是一个在请求阶段触发，另一个在响应阶段触发，而是按照order顺序在请求阶段分别被触发一次，然后又在响应阶段分别被触发一次。
但是请求过滤器首先没有post逻辑(.then)，所以并没有执行，而响应过滤器也是只有pre逻辑，但是在pre逻辑埋了一个钩子Decorator，这个钩子会在微服务返回的时候触发。
严格来说这两个过滤器其实都没有post阶段的逻辑。

另外，pre 阶段是同步按代码顺序执行的，post阶段是异步，响应回来时触发

### 4. ApiRestrictService
被请求过滤器调用，限流逻辑在这里apiRestrict方法中。
如果请求不在限流url 中，放行，如果在，进行流量检查（lua 脚本在这里实现），如果触发限流，记录kafka，并返回用户操作频繁提示。


### 5. ChannelDataService
渠道数据服务，核心方法getChannelDataByCode, 在请求和响应的过滤器里都有用到，和验证签名有关

### 6. TokenService
核心方法getUser，在请求和响应过滤器里都有用到，去Redis里面用Request Header里的token查用户的信息

### 7. application-pro.yml
路由配置。
application.yml: 默认配置文件。无论什么环境，Spring Boot 总是会加载它。
application-{profile}.yml: 环境特定配置文件。只有当对应的 profile 被激活时，这个文件才会被加载。profile配置文件可以覆盖默认的，和application.properties一样的
路由配置解释举例：
routes:
- id: ${prefix.distinction.name:damai}-base-data-service 这个配置的名称
uri: lb://${prefix.distinction.name:damai}-base-data-service 负载均衡去到的微服务
predicates:
- Path=/damai/basedata/** 断言匹配规则，这个path的回去上面那个uri
filters:
- StripPrefix=2 这个是指剥离掉几层path，2就是/damai + /basedata，后面的是真正的微服务uri
metadata:
title: 基础数据服务
### 8. GatewayProperty
网关的配置常量

### 9. Config
一些Bean的工厂方法，比如线程池，还有其他配置比如CORS
这个线程池的配置参数可以参考，最后一个是lambda表达式实现了ThreadFactory接口，对每一个runnable，创建一个Thread，并分配线程名称
```java
    public ThreadPoolExecutor threadPoolExecutor(){
        return new ThreadPoolExecutor(Runtime.getRuntime().availableProcessors(),
                Runtime.getRuntime().availableProcessors()+10,
                60,
                TimeUnit.SECONDS,
                new LinkedBlockingQueue<>(),
                r -> new Thread(
                        Thread.currentThread().getThreadGroup(), r,
                        "listen-start-thread-" + threadCount.getAndIncrement()));
    }
```
### 10. GatewayDefaultExceptionHandler
实现了ErrorWebExceptionHandler接口的handle 方法。
只要是在网关处理请求的过程中（从接收请求到返回响应），发生了未被捕获的错误（无论是主动抛的，还是系统被动崩的），都会走到这个方法里。
它是统一格式化错误返回的最佳位置。
例如如果请求网关的URL不在任何的路由配置中，网关将直接通过这里返回404。
而对于网关级别的熔断来说，SentinelGatewayBlockExceptionHandler 先执行，GatewayDefaultExceptionHandler 后执行。
也就是说如果是限流问题，会先触发Sentinel的异常，如果是网关的逻辑执行异常，会冒泡到后者抛出。
简单来说Sentinel在网关的最外层。

### 11. HystrixFallBackController
当后端的微服务出故障（挂了、超时了、报错了）时，Spring Cloud Gateway 会自动把请求转发到这个接口，给用户返回一个友好的提示，而不是直接报错。
这个URL理论上外界可以直接请求的。如果是路由URI发生异常，Spring 框架会内部转发到这个接口，实现熔断保护。
从application.yml来看当前的Hystrix是开在Feign通讯里的，网关的控流用的Sentinel，有点分裂。可以全栈使用Sentinel。

总体流程是这样的：外部请求到达网关 -> Sentinel (GatewayFilter) 检查 QPS： 如果太快 -> 抛出 BlockException -> SentinelGatewayBlockExceptionHandler 捕获 -> 返回“限流提示”。
通过 Sentinel 后，请求进入 Hystrix (GatewayFilter)： 如果配置了 Hystrix 线程隔离或熔断 -> Hystrix 生效。
Hystrix 尝试通过 Feign/RPC 调用后端：
如果后端挂了 -> Hystrix 触发 fallback -> 转发给 fallBackHandler。

### 12. RateLimiter
Semaphore（信号量）经典并发工具，有点像停车场前的闸口。Semaphore 用来控制同时访问某个资源的线程数量。
严格来说，这个类实现的并不是QPS限制，而是并发控制。
真正的 Rate Limiter (速率限制, 如 QPS): 限制每秒钟只能有 100 个请求进来。 常用算法：令牌桶 (Token Bucket)、漏桶 (Leaky Bucket)。

这个类的实现 (Semaphore):
限制同一时刻最多只能有 100 个线程在运行。只有新的线程空出来才能接收新的请求，否则抛出异常。

Semaphore vs ReentrantLock
前者像锁因为它具有阻塞型和排他性，如果计数器为1，即只有一个线程可以并发访问，那么信号量就退化成了排他锁。
但信号量是不可重入的，ReentrantLock默认相同的线程在持有锁的情况下可以重复进入，但信号量不可以，计数就是会--。

