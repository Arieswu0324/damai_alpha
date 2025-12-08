# 大麦网 Spring Cloud Gateway 服务

## 项目概述

这是一个基于 Spring Cloud Gateway 构建的高性能微服务网关，为大麦网票务系统提供统一的流量入口。该网关实现了请求路由、安全认证、API限流、数据加密、灰度发布等企业级功能。

### 核心特性

- **统一路由管理**：聚合8个后端微服务，提供统一访问入口
- **多层安全防护**：渠道认证、RSA签名验证、JWT Token认证
- **智能限流系统**��支持普通限流和深度限流（基于时间段的动态限流）
- **数据加密传输**：支持请求解密和响应加密（RSA算法）
- **分布式追踪**：TraceID全链路追踪
- **熔断降级**：Sentinel + Hystrix双重保障
- **灰度路由**：支持灰度服务和生产服务隔离
- **API统计分析**：通过Kafka异步记录API调用数据

---

## 技术栈

### 核心框架
- **Spring Boot** - 应用基础框架
- **Spring Cloud Gateway** - 网关核心（基于Reactor Netty，异步非阻塞）
- **Spring Cloud OpenFeign** - 服务间通信

### 服务治理
- **Alibaba Nacos** - 服务注册与发现
- **Alibaba Sentinel** - 流量控制、熔断降级
- **Hystrix** - 备选熔断方案
- **Spring Cloud LoadBalancer** - 负载均衡

### 中间件
- **Redis** - 缓存、限流计数
- **Kafka** - 异步消息队列（API统计数据）

### 安全与加密
- **RSA** - 数据加密和签名验证
- **JWT** - Token认证
- **Jasypt** - 配置文件加密

### 其他技术
- **Lua** - Redis原子操作脚本
- **Knife4j** - API文档聚合
- **Log4j2** - 日志框架
- **Hutool** - Java工具库
- **FastJSON** - JSON处理

---

## 项目结构

```
damai-gateway-service/
├── src/main/java/com/damai/
│   ├── GatewayApplication.java                    # 网关启动类
│   ├── conf/                                       # 配置类
│   │   ├── Config.java                            # 核心配置（线程池、CORS、异常处理）
│   │   ├── RequestTemporaryWrapper.java           # 请求临时包装器
│   │   └── sentinel/
│   │       └── GatewaySentinelConfiguration.java  # Sentinel限流配置
│   ├── constant/
│   │   └── GatewayConstant.java                   # 网关常量定义
│   ├── controller/
│   │   └── HystrixFallBackController.java         # 熔断降级处理
│   ├── dto/
│   │   └── ApiDataDto.java                        # API调用数据传输对象
│   ├── exception/
│   │   └── GatewayDefaultExceptionHandler.java    # 全局异常处理器
│   ├── filter/                                     # 全局过滤器
│   │   ├── RequestValidationFilter.java           # 请求验证过滤器（核心）
│   │   └── ResponseValidationFilter.java          # 响应验证过滤器
│   ├── kafka/                                      # Kafka消息相关
│   │   ├── ApiDataMessageSend.java                # API数据消息发送
│   │   ├── KafkaTopic.java                        # Kafka主题配置
│   │   └── ProducerConfig.java                    # Kafka生产者配置
│   ├── pro/limit/                                  # 限流模块
│   │   ├── RateLimiter.java                       # 限流器（信号量）
│   │   ├── RateLimiterConfig.java                 # 限流配置
│   │   └── RateLimiterProperty.java               # 限流属性
│   ├── property/
│   │   └── GatewayProperty.java                   # 网关配置属性
│   ├── service/                                    # 业务服务层
│   │   ├── ApiRestrictService.java                # API限制服务（限流核心）
│   │   ├── ApiRestrictData.java                   # 限制数据对象
│   │   ├── ChannelDataService.java                # 渠道数据服务
│   │   ├── TokenService.java                      # Token服务
│   │   └── lua/
│   │       └── ApiRestrictCacheOperate.java       # Lua脚本执行器
│   └── vo/                                         # 值对象
│       ├── DepthRuleVo.java                       # 深度限流规则
│       ├── RuleVo.java                            # 普通限流规则
│       └── UserVo.java                            # 用户信息对象
├── src/main/resources/
│   ├── application.yml                             # 开发环境配置
│   ├── application-pro.yml                         # 生产环境配置
│   ├── log4j2.xml                                 # 日志配置
│   └── lua/
│       └── apiLimit.lua                           # API限流Lua脚本
└── pom.xml                                         # Maven配置
```

---

## 核心功能详解

### 1. 请求验证与处理流程

**RequestValidationFilter** 是网关的核心过滤器（Order = -2），负责请求的全链路验证和处理。

#### 处理流程

```
1. TraceID生成 → 分布式追踪标识
2. 灰度路由判断 → 区分灰度/生产环境
3. 渠道验证 → 获取渠道密钥信息
4. RSA签名验证 → 防止请求篡改
5. RSA数据解密 → 解密业务数据（支持V2加密方案）
6. Token认证 → 解析JWT获取用户身份
7. API限流检查 → 执行频率限制规则
8. 请求头装饰 → 传递关键参数到下游服务
```

#### 关键实现

```java
// 文件路径：src/main/java/com/damai/filter/RequestValidationFilter.java

// 1. 生成TraceID
String traceId = IdUtils.getSnowflakeNextIdStr();
MDC.put("traceId", traceId);

// 2. 渠道验证和密钥获取
ProgramChannelVo programChannelVo = channelDataService.selectByCode(code);

// 3. RSA签名验证
boolean signVerify = RsaUtil.verify(requestBody, sign, programChannelVo.getSignPublicKey());

// 4. 数据解密
String decryptData = RsaUtil.decryptByPrivateKey(data, programChannelVo.getTicketRsaPrivateKey());

// 5. Token解析
UserVo userVo = tokenService.getToken(token);

// 6. API限流检查
apiRestrictService.check(ApiRestrictData.builder()
    .checkPath(checkPath)
    .userId(userId)
    .ip(ip)
    .build());
```

### 2. 智能限流系统

网关实现了多层级的限流机制，保障系统在高并发场景下的稳定性。

#### 两层限流架构

##### (1) 网关层限流（全局流量控制）

基于Java信号量（Semaphore）实现，控制网关整体并发量。

```java
// 文件路径：src/main/java/com/damai/pro/limit/RateLimiter.java

public void acquire() throws InterruptedException {
    if (!semaphore.tryAcquire(1, timeUnit)) {
        throw new DaMaiFrameException(BaseCode.OPERATION_IS_TOO_FREQUENT);
    }
}

// 配置项（application.yml）
rate.switch: false    # 限流开关
rate.permits: 200     # 最大并发数
```

##### (2) API层限流（细粒度控制）

基于Redis + Lua脚本实现，支持两种限流规则：

**普通限流规则（RuleVo）**
```java
// 文件路径：src/main/java/com/damai/vo/RuleVo.java

statTime: 60          // 统计时间窗口（秒）
threshold: 100        // 请求次数阈值
effectiveTime: 300    // 触发后的禁用时间（秒）
```

**深度限流规则（DepthRuleVo）**
```java
// 文件路径：src/main/java/com/damai/vo/DepthRuleVo.java

// 支持基于时间段的动态限流
示例：高峰期（14:00-18:00）限制50次/分钟
     非高峰期限制100次/分钟
```

#### Lua脚本实现

```lua
-- 文件路径：src/main/resources/lua/apiLimit.lua

-- 核心逻辑：
-- 1. INCRBY 计数器递增
-- 2. 判断是否超过阈值
-- 3. 触发限流则设置限流标记键
-- 4. 检查深度规则（多时间段规则）
-- 5. 返回：触发标志、规则类型、当前计数、阈值、错误消息索引
```

#### 限流数据统计

当触发限流时，API调用信息会异步发送到Kafka用于数据分析：

```java
// 文件路径：src/main/java/com/damai/service/ApiRestrictService.java

ApiDataDto apiDataDto = ApiDataDto.builder()
    .ip(ip)
    .url(checkPath)
    .createTime(new Date())
    .build();
apiDataMessageSend.send(apiDataDto);
```

### 3. 安全认证机制

#### 三层认证体系

##### (1) 渠道认证
```java
// 验证渠道code是否合法
// 获取渠道的加密密钥和签名公钥
ProgramChannelVo channelVo = channelDataService.selectByCode(code);
```

##### (2) RSA签名验证
```java
// 使用渠道公钥验证请求签名，防止数据篡改
boolean verified = RsaUtil.verify(requestBody, sign, publicKey);
```

##### (3) JWT Token认证
```java
// 解析Token获取用户身份
UserVo userVo = tokenService.getToken(token);
```

#### 认证配置

```yaml
# application.yml

skip.check.token.paths:          # 不需要Token的路径（如登录、注册）
  - /damai/user/login
  - /damai/user/register

skip.check.parameter.paths:      # 不需要参数验证的路径

userId.paths:                     # 需要UserId的路径（如个人中心）
  - /damai/user/info

allow.normal.access: false        # 是否允许无签名访问
```

### 4. 数据加密传输

#### 请求解密流程

```java
// 文件路径：src/main/java/com/damai/filter/RequestValidationFilter.java

// 使用渠道私钥解密请求数据
String decryptData = RsaUtil.decryptByPrivateKey(
    encryptedData,
    channelVo.getTicketRsaPrivateKey()
);
```

#### 响应加密流程

```java
// 文件路径：src/main/java/com/damai/filter/ResponseValidationFilter.java

// 当请求指定需要加密（encrypt=v2）时，对响应数据进行RSA加密
if ("v2".equals(encrypt)) {
    String encryptedResponse = RsaUtil.encryptByPublicKey(
        responseBody,
        publicKey
    );
}
```

### 5. 服务路由配置

网关聚合了8个后端微服务，提供统一访问入口。

#### 路由列表

| 服务名称 | 路由路径 | 目标服务 | 说明 |
|---------|---------|---------|------|
| 基础数据服务 | `/damai/basedata/**` | `damai-base-data-service` | 系统基础数据 |
| 定制化服务 | `/damai/customize/**` | `damai-customize-service` | 业务定制化功能 |
| 任务服务 | `/damai/job/**` | `damai-job-service` | 后台任务执行 |
| 订单服务 | `/damai/order/**` | `damai-order-service` | 订单管理 |
| 支付服务 | `/damai/pay/**` | `damai-pay-service` | 支付处理 |
| 节目服务 | `/damai/program/**` | `damai-program-service` | 演出节目信息 |
| 用户服务 | `/damai/user/**` | `damai-user-service` | 用户管理 |
| 管理服务 | `/damai/admin/**` | `damai-admin-service` | 后台管理 |

#### 路由配置示例

```yaml
# 文件路径：src/main/resources/application-pro.yml

spring:
  cloud:
    gateway:
      routes:
        - id: damai-order-service
          uri: lb://damai-order-service      # 使用Nacos服务发现
          predicates:
            - Path=/damai/order/**
          filters:
            - StripPrefix=2                   # 移除 /damai/order/ 前缀
```

**路由特性**：
- 基于 `lb://` 协议实现Nacos服务发现
- 自动负载均衡（Spring Cloud LoadBalancer）
- 路径前缀剥离（StripPrefix）
- 支持Knife4j网关文档聚合

### 6. 异常处理

**GatewayDefaultExceptionHandler** 提供统一的异常处理机制。

```java
// 文件路径：src/main/java/com/damai/exception/GatewayDefaultExceptionHandler.java

// 支持的异常类型：
// 1. 404错误（路由不存在）
// 2. DaMaiFrameException（业务异常）
// 3. ArgumentException（参数校验异常）
// 4. 其他未知异常
```

#### 异常响应格式

```json
{
  "code": "错误代码",
  "message": "错误描述",
  "data": "异常详情（可选）"
}
```

### 7. 灰度路由

支持灰度服务和生产服务的流量隔离。

```java
// 文件路径：src/main/java/com/damai/filter/RequestValidationFilter.java

// 从请求头获取灰度标记
String gray = serverHttpRequest.getHeaders().getFirst(GrayConstant.GRAY);

// 传递给下游服务
exchange.getRequest().mutate()
    .header(GrayConstant.GRAY, gray)
    .build();
```

### 8. 分布式追踪

使用TraceID实现全链路追踪。

```java
// 生成唯一TraceID
String traceId = IdUtils.getSnowflakeNextIdStr();

// 设置到MDC（日志上下文）
MDC.put("traceId", traceId);

// 传递到下游服务
exchange.getRequest().mutate()
    .header("traceId", traceId)
    .build();
```

### 9. 熔断降级

#### Sentinel配置

```java
// 文件路径：src/main/java/com/damai/conf/sentinel/GatewaySentinelConfiguration.java

// 配置Sentinel网关适配器
// 支持从Nacos加载限流规则
// 自定义熔断异常处理器
```

#### Hystrix降级

```java
// 文件路径：src/main/java/com/damai/controller/HystrixFallBackController.java

@RestController
public class HystrixFallBackController {

    @GetMapping("/fallback")
    public BaseResponse<String> fallback() {
        return BaseResponse.fail("服务暂时不可用，请稍后重试");
    }
}
```

---

## 数据流转完整链路

```
用户请求
    ↓
[RequestValidationFilter - Order: -2]
  ├─ 生成TraceID（分布式追踪）
  ├─ 读取请求体（Reactive异步）
  ├─ 渠道验证 → ChannelDataService（Redis缓存 + Feign远程调用）
  ├─ RSA签名验证（防篡改）
  ├─ RSA数据解密（业务数据）
  ├─ Token解析 → TokenService（Redis会话）
  ├─ API限流检查 → ApiRestrictService（Lua脚本 + Redis）
  │   └─ 触发限流 → Kafka异步发送统计数据
  └─ 装饰请求头（传递traceId、userId、gray等）
    ↓
[Spring Cloud Gateway路由]
  ├─ 路由匹配（Path Predicate）
  ├─ 服务发现（Nacos）
  ├─ 负载均衡（LoadBalancer）
  └─ 转发请求到目标服务
    ↓
[目标微服务]
  ├─ 业务逻辑处理
  └─ 返回响应
    ↓
[ResponseValidationFilter - Order: -2]
  └─ RSA加密响应数据（如果请求需要）
    ↓
[GatewayDefaultExceptionHandler]
  └─ 统一异常处理（如果出错）
    ↓
返回响应给用户
```

---

## 配置说明

### application.yml（开发环境）

```yaml
server:
  port: 6085                                  # 网关端口

spring:
  application:
    name: damai-gateway-service
  profiles:
    active: pro                               # 激活生产环境配置
  redis:
    host: 127.0.0.1
    port: 6379
  cloud:
    nacos:
      discovery:
        server-addr: 127.0.0.1:8848           # Nacos地址
    sentinel:
      eager: true                             # 启动时加载Sentinel
      transport:
        dashboard: 127.0.0.1:8082             # Sentinel控制台

# API限流路径配置
api.limit.paths:
  - /damai/order/create
  - /damai/pay/submit

# 网关层限流配置
rate:
  switch: false                               # 限流开关
  permits: 200                                # 最大并发数

# 认证配置
skip.check.token.paths:                       # 免Token路径
  - /damai/user/login
  - /damai/user/register

allow.normal.access: false                    # 是否允许无签名访问

# Feign配置
feign:
  hystrix:
    enabled: true                             # 启用Hystrix熔断
  okhttp:
    enabled: true                             # 使用OkHttp连接池

# Knife4j文档聚合
knife4j:
  gateway:
    enabled: true
```

### application-pro.yml（生产环境）

包含8个微服务的路由配置，支持Knife4j OpenAPI 3.0规范文档聚合。

---

## 性能优化设计

### 1. 异步非阻塞

- **Reactor Netty**：Spring Cloud Gateway基于Netty，全异步非阻塞I/O
- **响应式编程**：使用Mono/Flux处理请求和响应
- **异步线程池**：远程调用（Feign）使用独立线程池

### 2. 缓存策略

- **渠道信息缓存**：Redis缓存 + 10秒超时，减少远程调用
- **API限流规则缓存**：规则存储在Redis，高速访问
- **用户会话缓存**：Token对应的用户信息缓存

### 3. 限流机制

- **网关层限流**：信号量控制并发量，防止过载
- **API层限流**：Lua脚本实现原子性操作，避免竞态条件
- **Sentinel分布式限流**：支持从Nacos动态加载规则

### 4. 连接池优化

- **Feign + OkHttp**：优化HTTP连接复用
- **自定义线程池**：异步任务隔离，防止线程阻塞

### 5. 日志与监控

- **TraceID追踪**：MDC传递，便于日志关联
- **Kafka数据上报**：限流触发时异步上报，不阻塞主流程
- **Log4j2异步日志**：提升日志性能

---

## 关键文件说明

| 文件路径 | 功能描述 | 重要性 |
|---------|---------|-------|
| `src/main/java/com/damai/GatewayApplication.java` | 网关启动入口 | ⭐⭐⭐ |
| `src/main/java/com/damai/filter/RequestValidationFilter.java` | 请求验证过滤器（核心） | ⭐⭐⭐⭐⭐ |
| `src/main/java/com/damai/filter/ResponseValidationFilter.java` | 响应加密过滤器 | ⭐⭐⭐⭐ |
| `src/main/java/com/damai/service/ApiRestrictService.java` | API限流核心服务 | ⭐⭐⭐⭐⭐ |
| `src/main/resources/lua/apiLimit.lua` | 限流Lua脚本 | ⭐⭐⭐⭐⭐ |
| `src/main/java/com/damai/service/ChannelDataService.java` | 渠道数据服务 | ⭐⭐⭐⭐ |
| `src/main/java/com/damai/service/TokenService.java` | Token认证服务 | ⭐⭐⭐⭐ |
| `src/main/resources/application-pro.yml` | 路由配置 | ⭐⭐⭐⭐ |
| `src/main/java/com/damai/conf/Config.java` | 核心配置（CORS、异常处理） | ⭐⭐⭐ |
| `src/main/java/com/damai/exception/GatewayDefaultExceptionHandler.java` | 全局异常处理 | ⭐⭐⭐ |

---

## 启动与部署

### 前置依赖

1. **Nacos**（服务注册中心）- 默认端口：8848
2. **Redis**（缓存、限流）- 默认端口：6379
3. **Sentinel Dashboard**（监控）- 默认端口：8082
4. **Kafka**（消息队列）- 默认端口：9092

### 启动步骤

```bash
# 1. 确保依赖服务已启动
# - Nacos
# - Redis
# - Sentinel Dashboard（可选）
# - Kafka（可选，如需API统计功能）

# 2. 编译打包
mvn clean package -DskipTests

# 3. 启动网关
java -jar target/damai-gateway-service.jar

# 4. 验证启动
# 访问：http://localhost:6085
# Knife4j文档：http://localhost:6085/doc.html
```

### 配置调整

根据实际环境修改 `application.yml` 中的配置：

```yaml
# Nacos地址
spring.cloud.nacos.discovery.server-addr: YOUR_NACOS_HOST:8848

# Redis地址
spring.redis.host: YOUR_REDIS_HOST
spring.redis.port: 6379

# Sentinel控制台
spring.cloud.sentinel.transport.dashboard: YOUR_SENTINEL_HOST:8082
```

---

## 自定义扩展

### 1. 添加新的路由

编辑 `application-pro.yml`：

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: your-new-service
          uri: lb://your-service-name
          predicates:
            - Path=/damai/yourpath/**
          filters:
            - StripPrefix=2
```

### 2. 自定义过滤器

```java
@Component
public class CustomFilter implements GlobalFilter, Ordered {

    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        // 前置处理
        return chain.filter(exchange).then(Mono.fromRunnable(() -> {
            // 后置处理
        }));
    }

    @Override
    public int getOrder() {
        return -1;  // 设置过滤器执行顺序
    }
}
```

### 3. 添加API限流规则

在 `application.yml` 中配置需要限流的路径：

```yaml
api.limit.paths:
  - /damai/order/create        # 订单创建接口
  - /damai/pay/submit          # 支付提交接口
```

然后在Redis或配置中心设置对应的限流规则（RuleVo、DepthRuleVo）。

---

## 常见问题

### Q1: 网关启动失败，提示连接Nacos失败

**解决方案**：
- 检查Nacos是否启动（默认端口8848）
- 验证 `application.yml` 中的Nacos地址配置是否正确
- 检查网络连通性

### Q2: 请求被限流，返回"操作过于频繁"

**解决方案**：
- 检查是否触发了网关层限流（`rate.switch=true` 且并发超过 `rate.permits`）
- 检查是否触发了API层限流（查看Redis中的限流规则）
- 查看Kafka中的限流统计数据，分析触发原因

### Q3: RSA签名验证失败

**解决方案**：
- 确认渠道code是否正确
- 验证签名算法是否一致（SHA256withRSA）
- 检查请求体和签名是否匹配
- 确认渠道公钥是否正确配置

### Q4: 如何查看网关日志

**解决方案**：
- 日志配置文件：`src/main/resources/log4j2.xml`
- 通过TraceID关联整个请求链路
- 使用 `MDC.get("traceId")` 在日志中获取TraceID

### Q5: 如何添加新的微服务路由

**解决方案**：
1. 确保新服务已注册到Nacos
2. 在 `application-pro.yml` 中添加路由配置
3. 重启网关服务
4. 验证路由是否生效

---

## 学习建议

如果你是第一次学习这个网关项目，建议按以下顺序阅读代码：

### 第1步：理解请求流程
1. `GatewayApplication.java:20` - 启动类
2. `RequestValidationFilter.java:89` - 请求过滤器主流程
3. `ResponseValidationFilter.java:46` - 响应过滤器

### 第2步：深入核心服务
4. `ApiRestrictService.java:63` - 限流核心逻辑
5. `apiLimit.lua:1` - Lua脚本实现
6. `ChannelDataService.java:47` - 渠道数据获取
7. `TokenService.java:35` - Token认证

### 第3步：理解配置与路由
8. `application-pro.yml:1` - 路由配置
9. `GatewayProperty.java:21` - 配置属性
10. `Config.java:32` - 核心配置（CORS、异常处理）

### 第4步：异常与降级
11. `GatewayDefaultExceptionHandler.java:37` - 异常处理
12. `HystrixFallBackController.java:17` - 熔断降级

### 第5步：性能优化
13. `RateLimiter.java:29` - 信号量限流
14. `KafkaTopic.java:21` / `ApiDataMessageSend.java:31` - Kafka异步上报

---

## 联系与支持

如有问题或建议，请联系项目维护团队。

---

**最后更新日期**：2025年12月

**文档版本**：v1.0
