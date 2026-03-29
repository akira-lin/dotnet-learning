# 阶段三：架构师路线（8个月+）

> 目标：具备系统设计能力，向 .NET 架构师迈进

---

## 📋 学习路线图

基于课程大纲，按优先级排序：

```
第一优先：.NET 底层 + 架构思维（贯穿整个阶段）
第二优先：微服务完整技术栈
第三优先：分布式 + 高并发
按需学习：K8s / Elasticsearch / 行业业务
```

## 🎯 架构师 vs 中级的区别

| 方面 | 中级工程师 | 架构师 |
|------|-----------|--------|
| 问题解决 | 解决具体问题 | 定义问题边界 |
| 技术选型 | 会用现有技术 | 决定用什么技术 |
| 系统设计 | 能看懂架构图 | 能画架构图并讲清为什么 |
| 业务理解 | 理解自己负责的模块 | 理解整个系统业务流 |
| 技术深度 | 知道怎么用 | 知道底层原理 |
| 软技能 | 执行任务 | 影响和说服 |

---

## 一、核心优先：.NET 底层原理

### 1.1 GC 垃圾回收机制

**推荐学习时间：** Week 16-24

```csharp
// GC 不是万能的，了解何时触发很重要
// 1. 内存压力时
// 2. 手动 GC.Collect()
// 3. 代际晋升时

// 常见的 GC 相关问题：
// - 大对象分配（>85KB）直接进入大对象堆
// - 过度装箱/拆箱
// - Finalize 拖慢 GC

// 建议用工具观察
dotnet-gcviz  // 查看 GC 行为
```

**学习资源：**
- .NET 源码：`github.com/dotnet/runtime`
- 书籍：《.NET Core 底层原理》

### 1.2 IOC 容器底层原理

**推荐学习时间：** Week 20-28

```csharp
// 手写一个简单 IOC 容器（理解原理）
public class MyContainer
{
    private readonly Dictionary<Type, Type> _services = new();

    public void Register<TInterface, TImplementation>()
        where TImplementation : TInterface
    {
        _services[typeof(TInterface)] = typeof(TImplementation);
    }

    public TInterface Resolve<TInterface>()
    {
        var interfaceType = typeof(TInterface);
        if (!_services.ContainsKey(interfaceType))
            throw new InvalidOperationException($"未注册: {interfaceType.Name}");

        var implementationType = _services[interfaceType];
        
        // 获取构造函数
        var constructor = implementationType.GetConstructors()[0];
        var parameters = constructor.GetParameters();

        // 递归解析依赖
        var args = parameters.Select(p => Resolve(p.ParameterType)).ToArray();
        
        return (TInterface)constructor.Invoke(args);
    }
}
```

### 1.3 异步编程深层

**推荐学习时间：** Week 16-24

```csharp
// ValueTask vs Task
// ValueTask：避免在异步方法中同步完成的包装开销
public async ValueTask<User> GetUserAsync(int id)
{
    var cached = _cache.Get(id);
    if (cached != null)
        return cached;  // 同步返回，避免包装
        
    return await _db.Users.FindAsync(id);
}

// IValueTaskSource 进阶用法（高性能场景）
// 一般项目用不到，但面试可能问
```

### 1.4 内存管理进阶

```csharp
// Span<T> 和 Memory<T>
Span<byte> span = stackalloc byte[1024];  // 栈上分配，避免 GC

// ArrayPool 复用数组
var pool = ArrayPool<byte>.Shared;
var buffer = pool.Rent(1024);
try
{
    // 使用 buffer
}
finally
{
    pool.Return(buffer);
}

// 了解不安全代码的用法（高性能场景）
```

---

## 二、核心优先：设计模式 + 架构思维

### 2.1 必须掌握的设计模式

| 模式 | 用途 | .NET 中的应用 |
|------|------|-------------|
| 工厂模式 | 封装对象创建 | `IServiceCollection.AddSingleton<T>` |
| 策略模式 | 运行时切换算法 | 多种支付方式、多种存储 |
| 装饰器模式 | 动态添加功能 | `Stream` 包装、日志中间件 |
| 观察者模式 | 事件通知 | `IObservable<T>`、C# 事件 |
| 建造者模式 | 复杂对象构建 | `StringBuilder`、配置构建 |
| 仓储模式 | 数据访问抽象 | Repository Pattern |

### 2.2 架构思维书籍

**必读：**
- 《Clean Architecture》（Bob 大叔）
- 《Fundamentals of Software Architecture》（Mark Richards）
- 《Designing Data-Intensive Applications》（Kleppmann）

**进阶：**
- 《Domain-Driven Design》（Evans）
- 《Building Microservices》（Newman）

### 2.3 C4 架构图

架构师的核心技能：**能把系统设计讲清楚**

```
Level 1: Context（上下文图）
         - 系统与外部参与者的关系

Level 2: Container（容器图）
         - 应用和数据库的划分

Level 3: Component（组件图）
         - 主要组件的职责

Level 4: Code（代码图）
         - 类图、时序图
```

**工具：** Draw.io、PlantUML、Miro

---

## 三、微服务完整技术栈

### 3.1 服务注册与发现

```
Consul / Nacos / Eureka
```

```yaml
# Consul 配置示例
services:
  consul:
    image: consul
    ports:
      - "8500:8500"
    networks:
      - backend

  api:
    build: .
    depends_on:
      - consul
    environment:
      - CONSUL_URL=http://consul:8500
```

### 3.2 API 网关

```
Ocelot / Kong / Nginx
```

```csharp
// Ocelot 配置示例（ocelot.json）
{
  "Routes": [
    {
      "DownstreamPathTemplate": "/api/{everything}",
      "DownstreamScheme": "http",
      "DownstreamHostAndPorts": [
        { "Host": "user-service", "Port": 5000 }
      ],
      "UpstreamPathTemplate": "/user/{everything}",
      "UpstreamHttpMethod": [ "GET", "POST", "PUT", "DELETE" ]
    }
  ]
}
```

### 3.3 熔断降级

```
Polly
```

```csharp
// 熔断
var circuitBreaker = Policy
    .Handle<HttpRequestException>()
    .CircuitBreakerAsync(
        exceptionsAllowedBeforeBreaking: 5,
        durationOfBreak: TimeSpan.FromSeconds(30));

// 重试
var retry = Policy
    .Handle<HttpRequestException>()
    .WaitAndRetryAsync(3, retryAttempt => 
        TimeSpan.FromSeconds(Math.Pow(2, retryAttempt)));

// 组合使用
var policyWrap = Policy.WrapAsync(retry, circuitBreaker);
```

### 3.4 链路追踪

```
SkyWalking / Jaeger / Zipkin
```

```yaml
# Docker Compose 集成 SkyWalking
oap:
  image: apache/skywalking-oap-server:9.5.0
  ports:
    - "11800:11800"  # GRPC
    - "12800:12800"  # REST

ui:
  image: apache/skywalking-ui:9.5.0
  ports:
    - "8080:8080"
  environment:
    SW_OAP_ADDRESS: oap:12800
```

### 3.5 监控

```
Prometheus + Grafana
```

```yaml
prometheus:
  image: prom/prometheus
  ports:
    - "9090:9090"
  volumes:
    - ./prometheus.yml:/etc/prometheus/prometheus.yml

grafana:
  image: grafana/grafana
  ports:
    - "3000:3000"
  depends_on:
    - prometheus
```

---

## 四、分布式 + 高并发

### 4.1 CAP 定理

分布式系统最多同时满足两个：

```
C（一致性）：所有节点数据一致
A（可用性）：每次请求都能得到响应
P（分区容错）：网络分区时仍能运行

CA（不可能）：单机数据库
CP（大多数选择）：Redis / MongoDB
AP（大多数选择）：Eureka / Cassandra
```

### 4.2 BASE 理论

```
Basically Available：基本可用
Soft state：软状态（中间状态）
Eventually consistent：最终一致性
```

### 4.3 分布式事务

| 方案 | 适用场景 | 复杂度 |
|------|---------|--------|
| 2PC / 3PC | 强一致，但效率低 | 高 |
| TCC | 业务参与 | 中 |
| 本地消息表 | 最终一致 | 中 |
| Saga | 长流程 | 中 |
| Seata | 开源方案 | 低 |

### 4.4 高并发场景

```csharp
// 1. 库存扣减（乐观锁）
await db.ExecuteAsync(@"
    UPDATE products 
    SET stock = stock - @quantity 
    WHERE id = @id AND stock >= @quantity",
    new { id, quantity });

// 2. 分布式锁（Redis）
var lock = await _redis.AcquireLockAsync("product:lock:" + id, TimeSpan.FromSeconds(10));
try
{
    // 业务逻辑
}
finally
{
    await lock.ReleaseAsync();
}

// 3. 幂等性设计（防重复提交）
public async Task<Order> CreateOrder([FromBody] CreateOrderDto dto)
{
    // 幂等 key
    var idempotentKey = $"order:idempotent:{dto.IdempotentToken}";
    var exists = await _redis.ExistsAsync(idempotentKey);
    if (exists) throw new Exception("订单已提交");
    
    await _redis.SetAsync(idempotentKey, "1", TimeSpan.FromHours(1));
    
    // 创建订单逻辑...
}
```

---

## 五、按需学习（先用后学）

### 5.1 Kubernetes（K8s）

**何时学：** 有相关项目需求

```
投入时间：2-3个月
学习曲线：陡峭
适用场景：大规划容器编排
```

**前期基础够用：**
- 先把 Docker + Docker Compose 用熟
- 了解 K8s 核心概念（Pod/Service/Deployment）
- 用 Minikube 本地练习

### 5.2 Elasticsearch

**何时学：** 有搜索功能需求

```
投入时间：1-2个月
学习曲线：中
适用场景：全文搜索、日志分析
```

### 5.3 Kafka

**何时学：** 消息量大、需要高吞吐

```
投入时间：1-2个月
学习曲线：中
适用场景：日志收集、实时分析
```

**RabbitMQ 够用吗？**
- 小项目：RabbitMQ 够用
- 大数据量：Kafka 更适合

### 5.4 Azure 云

**何时学：** 想出海或去外企

```
投入时间：持续学习
适用场景：海外业务、外企工作
```

---

## 六、架构师日常 + 面试

### 6.1 架构师做什么

```
日常：
- 参与技术选型评审
- 设计系统架构
- 写核心代码（不是 CRUD）
- 解决技术难点
- 指导初级工程师
- 写技术文档
- 跨团队协调
```

### 6.2 架构师面试常问

| 问题 | 考察点 |
|------|--------|
| 描述一个系统设计过程 | 设计思维 |
| 如何保证接口高并发 | 缓存/异步/限流 |
| 数据库拆分策略 | 数据分片 |
| 微服务拆分的原则 | 领域划分 |
| 如何做服务降级 | 容错设计 |
| CAP 定理如何取舍 | 分布式理论 |

### 6.3 积累架构经验

```
1. 主动参与架构评审（即使只是旁听）
2. 试着自己设计一个中型系统
3. 读优秀开源项目的架构（eShopOnContainers）
4. 画架构图、写设计文档
5. 尝试给别人讲清楚设计
```

---

## 七、里程碑总结

| 阶段 | 时间 | 目标 | 能力 |
|------|------|------|------|
| 阶段一 | 4个月 | 能接单 | CRUD + 简单缓存 |
| 阶段二 | 8个月 | 中级工程师 | 分布式 + 容器化 |
| 阶段三 | 1年+ | 架构师 | 系统设计 + 技术深度 |

---

## 📌 最后的建议

```
1. 不要追新框架，基础永远比框架重要
2. 不要停止写代码，架构师不能脱离实践
3. 不要忽视软技能，架构师是技术沟通者
4. 找到自己的领域（电商/金融/物流/工厂）
5. 持续学习，这行没有终点
```

---

*教程完成！祝学习顺利 🎉*
