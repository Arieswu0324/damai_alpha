### 1. 微服务microservices
monolithic 单体服务拆分成多个独立的服务，可以实现独立部署和功能解耦。
微服务是一种架构风格，至于每个微服务之间的通讯方式是RPC，还是REST这是另外实现层面的事情。
之前MOP项目应该算是比较简单的微服务架构，现代开发应该非常少用到单体服务了。
现代开发中的选择原则是：
“先用单体，直到你证明需要微服务 (Start with a monolith, until you prove you need microservices.)”

### 2. SpringCloud / SpringCloud Alibaba
Spring Cloud 提供了一站式的、开箱即用的解决方案，帮助利用 Java/Spring 生态快速搭建起一套可运行、可运维的微服务系统。
就是个框架，把微服务生态需要的东西基本在这个框架里提供好了（全家桶），接了就能用，就像SpringAI提供了一些上层的封装，下面无论调用什么LLM，向量数据库都能直接用。

SpringCloud Alibaba又是针对于高并发的场景对传统Spring Cloud框架进行了一些整合和增强。
主要就是引入了一些适用于高并发场景的组件，比如Nacos, Sentinel，RocketMQ, OSS啥的，国内Java微服务的主流架构。

### 3. Nacos是什么
阿里开源的服务发现和配置中心一站式平台。理论上 Zookeeper 提供的 API 可以实现服务注册、服务发现、配置管理等所有功能。
Nacos 就是在 Zookeeper 的能力基础上，针对微服务场景进行了封装和优化，把服务发现和配置管理这些繁琐的通用逻辑集成在一起，变成了开箱即用的功能。
本着能用封装不用底层的原则，Nacos替代Zookeeper

### 4. 什么是Cloud Native
云原生是一套设计、构建和运行应用程序的方法论和技术体系，旨在最大化地发挥云计算的优势。
主要包括Containerization(Docker), Orchestration(K8S), Microservices(REST, RPC), CI CD/ DevOps (Jenkins, Harness)

### 5. 什么是RocketMQ
Kafka对家消息队列，阿里巴巴（Alibaba）开发并开源，最初是为电商交易场景设计的。使用场景：需要严格的消息顺序、高可靠性和分布式事务支持（如订单支付流程），可以直接集成在SpringCloud Alibaba生态。


### 6. 什么是Circuit Breaking 熔断
熔断是一种服务容错的模式，如果当前服务因为某种原因出现了故障，为了引起整条调用链上的雪崩效应（Cascading Failure），而采取的一种保护机制。
核心目的： 牺牲局部，保全整体。 当一个依赖服务出现问题时，熔断机制可以快速切断与其的连接，释放本地资源，从而避免整个系统的崩溃。
熔断保护：“降级”是指在系统资源紧张或依赖服务不可用时，暂时关闭一些非核心功能，或者使用备用逻辑、缓存数据等方式，以保证核心功能的正常运行。
比如Fail Fast，直接返回失败而不等异常服务处理，或者直接返回一个静态的资源进行降级。

### 7. 什么是Sentinel
Sentinel 是一个非常关键的微服务组件，它专注于服务容错，是 Spring Cloud Alibaba 生态中的核心流量控制、熔断降级框架。
它的核心设计目标是：
流量控制 (Flow Control)： 根据系统的负载情况，动态地限制进入应用的请求数量，防止系统被“压垮”。
服务容错 (Fault Tolerance)： 应对外部依赖服务（如数据库、其他微服务）的故障或延迟，防止故障蔓延。
非国内的话，有Resilience4j,类似功能的组件，但不深度绑定SCA，功能包括Circuit Breaker, Rate Limiter(限制总请求量), Retry, Bulkhead

### 8. Bulkhead的作用
隔离资源，比如对线程池的保护： 为每个依赖服务分配一个独立的线程池。如果对服务 A 的调用阻塞或延迟，只有服务 A 的线程池会被耗尽，其他服务的线程池仍可正常工作。
它确保一个服务内的部分资源异常，只在它自己的隔离范围里失败，不会影响整个服务。

### 9. SpringBootAdmin是什么
如果有多个 Spring Boot 微服务在运行，SpringBoot Admin 就是一个集中的、图形化的控制中心，可以一站式地查看它们的健康状况、配置信息和运行指标。是一个本地化的轻量级工具。
DataDog是企业级的监控平台。
SpringBoot内嵌有Micrometer收集器，Micrometer 自动收集 JVM、CPU、Tomcat、Spring Bean 等指标。所以代码中不需要手动调用记录指标这样的接口，Actuator会自动暴露端点，SBA会定时拉取数据，但是SBA本身不对时序数据做持久化。

### 10. Actuator是什么
Spring Boot 框架自带的一个子项目，它为生产环境提供了一套用于监控和管理应用程序的强大功能。对于SBA场景，SBA本身是Server， 微服务是Client，执行器是Client侧负责收集指标数据，和响应Server HTTP请求的组件。
Admin 会定期用PULL模式请求Client侧，Actuator会计算或返回指标结果。

### 11. ShardingSphere是什么
是一个分布式数据库生态系统。ShardingSphere 的作用是解决传统关系型数据库（如 MySQL、PostgreSQL）在处理海量数据和高并发时的扩展性问题。








