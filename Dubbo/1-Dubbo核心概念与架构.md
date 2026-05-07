# Dubbo 核心概念与架构

## 1. Dubbo 概述

```
Dubbo 是什么？

┌─────────────────────────────────────────────────────────────┐
│                      Apache Dubbo                           │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  定义: 高性能、轻量级的开源 Java RPC 框架                   │
│                                                             │
│  核心能力:                                                  │
│  ├── RPC 远程调用: 像调用本地方法一样调用远程服务           │
│  ├── 服务治理: 负载均衡、路由、容错、限流                   │
│  ├── 服务发现: 自动注册与发现服务实例                       │
│  └── 高性能: NIO 通信、序列化优化、连接池                   │
│                                                             │
│  核心概念:                                                  │
│  ├── Provider: 服务提供者                                  │
│  ├── Consumer: 服务消费者                                  │
│  ├── Registry: 注册中心（Nacos/Zookeeper/...）             │
│  ├── Monitor: 监控中心                                     │
│  └── Container: 服务运行容器                               │
│                                                             │
│  版本演进:                                                  │
│  ├── Dubbo 2.x: 阿里开源，Spring 集成                      │
│  ├── Dubbo 3.x: Apache 顶级项目，Triple 协议               │
│  └── 支持协议: Dubbo/HTTP/gRPC/Triple                      │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 2. 架构设计

```
Dubbo 架构:

┌─────────────────────────────────────────────────────────────┐
│                    Dubbo 架构                               │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌──────────┐                      ┌──────────┐            │
│  │ Consumer │                      │ Provider │            │
│  │ 服务消费者│                      │ 服务提供者│            │
│  └────┬─────┘                      └────┬─────┘            │
│       │                                 │                   │
│       │ 1.订阅                          │ 2.注册            │
│       ▼                                 ▼                   │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              Registry (注册中心)                     │   │
│  │         Nacos / Zookeeper / Eureka / ...            │   │
│  └─────────────────────────────────────────────────────┘   │
│       │                                 │                   │
│       │ 3.通知                          │                   │
│       ▼                                 │                   │
│  ┌──────────┐                           │                   │
│  │ Consumer │──────4.调用服务──────────→│ Provider │        │
│  └──────────┘                           └──────────┘        │
│       │                                 │                   │
│       │            ┌──────────┐         │                   │
│       └────────────│ Monitor  │─────────┘                   │
│              5.统计 │ 监控中心 │                             │
│                    └──────────┘                             │
│                                                             │
│  调用流程:                                                  │
│  1. Consumer 订阅 Registry 的服务列表                       │
│  2. Provider 注册服务到 Registry                            │
│  3. Registry 通知 Consumer 服务列表变化                     │
│  4. Consumer 直接调用 Provider（RPC）                       │
│  5. Consumer 和 Provider 上报调用统计到 Monitor             │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## 3. 环境配置

### 3.1 Maven 依赖

```xml
<!-- pom.xml -->
<properties>
    <dubbo.version>3.2.10</dubbo.version>
    <nacos.version>2.3.0</nacos.version>
</properties>

<dependencies>
    <!-- Dubbo -->
    <dependency>
        <groupId>org.apache.dubbo</groupId>
        <artifactId>dubbo-spring-boot-starter</artifactId>
        <version>${dubbo.version}</version>
    </dependency>
    
    <!-- Nacos 注册中心 -->
    <dependency>
        <groupId>org.apache.dubbo</groupId>
        <artifactId>dubbo-registry-nacos</artifactId>
        <version>${dubbo.version}</version>
    </dependency>
    
    <!-- 或者使用 Zookeeper -->
    <dependency>
        <groupId>org.apache.dubbo</groupId>
        <artifactId>dubbo-registry-zookeeper</artifactId>
        <version>${dubbo.version}</version>
    </dependency>
</dependencies>
```

### 3.2 Spring Boot 配置

```yaml
# application.yml
dubbo:
  application:
    name: user-service
    qos-enable: false
  
  registry:
    address: nacos://127.0.0.1:8848
    # 或 zookeeper://127.0.0.1:2181
  
  protocol:
    name: dubbo
    port: 20880
  
  consumer:
    check: false  # 启动时不检查服务提供者
    timeout: 5000
    retries: 2
  
  provider:
    timeout: 5000
    retries: 0
```

## 4. 服务定义与实现

### 4.1 API 模块（公共接口）

```java
// user-api 模块
public interface UserService {
    
    UserDTO getUserById(Long id);
    
    List<UserDTO> getUsersByIds(List<Long> ids);
    
    UserDTO createUser(CreateUserRequest request);
}
```

### 4.2 服务提供者

```java
// user-service 模块
@DubboService(version = "1.0.0", group = "user")
public class UserServiceImpl implements UserService {
    
    @Autowired
    private UserRepository userRepository;
    
    @Override
    public UserDTO getUserById(Long id) {
        User user = userRepository.findById(id)
            .orElseThrow(() -> new UserNotFoundException(id));
        return toDTO(user);
    }
    
    @Override
    public List<UserDTO> getUsersByIds(List<Long> ids) {
        return userRepository.findAllById(ids)
            .stream()
            .map(this::toDTO)
            .collect(Collectors.toList());
    }
    
    @Override
    public UserDTO createUser(CreateUserRequest request) {
        User user = new User();
        user.setUsername(request.getUsername());
        user.setEmail(request.getEmail());
        User saved = userRepository.save(user);
        return toDTO(saved);
    }
}
```

### 4.3 服务消费者

```java
// order-service 模块
@Service
public class OrderService {
    
    @DubboReference(version = "1.0.0", group = "user")
    private UserService userService;
    
    public OrderDTO createOrder(CreateOrderRequest request) {
        // 调用远程用户服务
        UserDTO user = userService.getUserById(request.getUserId());
        
        Order order = new Order();
        order.setUserId(user.getId());
        order.setUserName(user.getUsername());
        // ...
        
        return toDTO(order);
    }
}
```

## 5. 注册中心

### 5.1 Nacos 配置

```yaml
# 使用 Nacos 作为注册中心
dubbo:
  registry:
    address: nacos://127.0.0.1:8848
    parameters:
      namespace: public
      group: DEFAULT_GROUP
```

### 5.2 Zookeeper 配置

```yaml
# 使用 Zookeeper 作为注册中心
dubbo:
  registry:
    address: zookeeper://127.0.0.1:2181
    timeout: 10000
```

### 5.3 多注册中心

```yaml
dubbo:
  registries:
    beijing:
      address: nacos://beijing-nacos:8848
      parameters:
        region: beijing
    shanghai:
      address: nacos://shanghai-nacos:8848
      parameters:
        region: shanghai
```

## 6. 协议与序列化

```
Dubbo 支持的协议:

┌──────────────┬──────────────┬──────────────┬──────────────┐
│ 协议         │ 连接方式     │ 适用场景     │ 特点         │
├──────────────┼──────────────┼──────────────┼──────────────┤
│ dubbo        │ 单一长连接   │ 小数据量     │ 高性能       │
│ rmi          │ 多连接       │ Java 原生    │ 兼容 RMI     │
│ hessian      │ 多连接       │ HTTP 通信    │ 跨语言       │
│ http         │ 多连接       │ HTTP 服务    │ 标准 REST    │
│ rest         │ 多连接       │ RESTful API  │ 跨语言       │
│ triple       │ 多连接       │ gRPC 兼容    │ 流式、跨语言 │
└──────────────┴──────────────┴──────────────┴──────────────┘

序列化方式:
- Hessian2: 默认，高性能二进制序列化
- JSON: 跨语言，可读性好
- Protobuf: Google 出品，高性能
- Kryo: Java 高性能序列化
```

```yaml
dubbo:
  protocol:
    name: dubbo
    port: 20880
    serialization: hessian2  # 默认序列化方式
```

## 7. 负载均衡

```
Dubbo 负载均衡策略:

┌──────────────┬──────────────────────────────────────────────┐
│ 策略         │ 说明                                          │
├──────────────┼──────────────────────────────────────────────┤
│ random       │ 随机（默认），按权重设置随机概率              │
│ roundrobin   │ 轮询，按公约后的权重轮询                      │
│ leastactive  │ 最少活跃调用数，慢的机器收到更少请求          │
│ consistenthash│ 一致性hash，相同参数的请求总是发到同一提供者 │
└──────────────┴──────────────────────────────────────────────┘
```

```java
// 服务级别配置
@DubboService(loadbalance = "roundrobin")
public class UserServiceImpl implements UserService {
    // ...
}

// 消费者级别配置
@DubboReference(loadbalance = "leastactive")
private UserService userService;

// 方法级别配置
@DubboReference(methods = {
    @Method(name = "getUserById", loadbalance = "consistenthash")
})
private UserService userService;
```

## 8. 集群容错

```
Dubbo 集群容错模式:

┌──────────────┬──────────────────────────────────────────────┐
│ 模式         │ 说明                                          │
├──────────────┼──────────────────────────────────────────────┤
│ failover     │ 失败自动切换（默认），重试其他服务器          │
│ failfast     │ 快速失败，只发起一次调用，失败立即报错        │
│ failsafe     │ 失败安全，失败时忽略，返回空结果              │
│ failback     │ 失败自动恢复，记录失败请求，定时重发          │
│ forking      │ 并行调用多个服务器，只要一个成功即返回        │
│ broadcast    │ 广播调用所有提供者，任意一台报错则报错        │
└──────────────┴──────────────────────────────────────────────┘
```

```java
// 服务级别配置
@DubboService(cluster = "failover")
public class UserServiceImpl implements UserService {
    // ...
}

// 消费者级别配置
@DubboReference(cluster = "failfast", retries = 3)
private UserService userService;
```

## 9. 超时与重试

```yaml
# 全局配置
dubbo:
  consumer:
    timeout: 5000      # 超时时间 5秒
    retries: 2         # 重试次数 2次
  
  provider:
    timeout: 5000
    retries: 0         # 提供者不重试（避免重复写入）
```

```java
// 接口级别配置
@DubboReference(
    timeout = 3000,
    retries = 2,
    mock = "return null"  # 降级策略
)
private UserService userService;

// 方法级别配置
@DubboReference(methods = {
    @Method(name = "getUserById", timeout = 1000),
    @Method(name = "createUser", timeout = 5000, retries = 0)
})
private UserService userService;
```

## 10. 监控与管理

```yaml
# Dubbo Admin 配置
dubbo:
  application:
    qos-enable: true
    qos-port: 22222
    qos-accept-foreign-ip: false
  
  monitor:
    protocol: registry  # 从注册中心获取监控地址
```

```
Dubbo 管理命令:

# 连接 Dubbo QOS
telnet 127.0.0.1 22222

# 查看服务列表
ls

# 查看服务详情
ls -l com.example.UserService

# 查看服务方法
invoke com.example.UserService.getUserById(1)

# 查看连接信息
connection

# 查看服务提供者
ps

# 查看服务消费者
ss
```

## 11. 生产环境配置

```yaml
# 生产环境完整配置
dubbo:
  application:
    name: ${spring.application.name}
    qos-enable: false
    register-mode: instance
  
  registry:
    address: nacos://${NACOS_ADDRESS:127.0.0.1}:8848
    parameters:
      namespace: ${ENV:prod}
  
  protocol:
    name: dubbo
    port: -1  # 自动分配端口
    serialization: hessian2
  
  consumer:
    check: false
    timeout: 5000
    retries: 2
    loadbalance: leastactive
    cluster: failover
  
  provider:
    timeout: 5000
    retries: 0
    loadbalance: roundrobin
    cluster: failfast
    executes: 200  # 并发执行上限
  
  # 服务级别配置
  services:
    userService:
      version: 1.0.0
      group: user
      timeout: 3000
      retries: 1
```

## 12. 最佳实践

```
Dubbo 使用 Checklist:

□ 接口设计:
  - 接口粒度适中，不要太细也不要太粗
  - 使用 DTO 传输对象，不要暴露内部实体
  - 接口版本管理，向前兼容

□ 配置管理:
  - 超时时间根据业务调整
  - 写操作不重试，读操作可重试
  - 使用分组隔离不同环境

□ 服务治理:
  - 配置合理的负载均衡策略
  - 配置合适的集群容错模式
  - 使用降级策略保证可用性

□ 性能优化:
  - 使用高效的序列化方式
  - 合理设置连接池大小
  - 启用数据压缩（大数据量）

□ 监控告警:
  - 启用 Dubbo Admin 监控
  - 配置服务调用统计
  - 设置超时和异常告警
```
