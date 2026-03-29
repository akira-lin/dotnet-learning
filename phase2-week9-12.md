# Week 9-12：WebAPI 进阶 + Redis 缓存

> 目标：掌握性能优化和缓存策略，面试和工作都能用

---

## 一、Redis 缓存

### 1.1 什么是 Redis

Redis 是一个**内存数据库**，读写速度极快（比 SQLite 快 10-100 倍）。

### 1.2 Redis 适用场景

| 场景 | 说明 |
|------|------|
| 热点数据缓存 | 商品信息、用户配置（变化少，读取多） |
| 会话存储 | 登录 Session |
| 消息队列 | 异步任务、延迟处理 |
| 分布式锁 | 多实例竞争同一资源 |

**不适合：** 大数据量存储、冷数据（一次写入几乎不读取）

### 1.3 安装 Redis（Windows）

```powershell
# 方法一：WSL（推荐）
wsl --install
# 然后在 Ubuntu 里：sudo apt install redis

# 方法二：Docker
docker run -d -p 6379:6379 redis
```

### 1.4 安装 Redis 客户端

```powershell
dotnet add package StackExchange.Redis
```

### 1.5 连接 Redis

```csharp
using StackExchange.Redis;

var redis = ConnectionMultiplexer.Connect("localhost:6379");
var db = redis.GetDatabase();

// 测试
await db.StringSetAsync("test", "Hello Redis");
var value = await db.StringGetAsync("test");
Console.WriteLine(value);  // Hello Redis
```

---

## 二、Redis 基本操作

### 2.1 String（字符串）

```csharp
// 存值
await db.StringSetAsync("user:1:name", "张三");
await db.StringSetAsync("user:1:age", "25", TimeSpan.FromMinutes(30)); // 30分钟过期

// 取值
var name = await db.StringGetAsync("user:1:name");

// 数字操作
await db.StringIncrementAsync("page:views");  // +1
var views = await db.StringGetAsync("page:views");
```

### 2.2 Hash（哈希）

适合存储对象：

```csharp
// 存对象
await db.HashSetAsync("user:1", new HashEntries
{
    new HashEntry("name", "张三"),
    new HashEntry("age", "25"),
    new HashEntry("city", "北京")
});

// 读取整个对象
var user = await db.HashGetAllAsync("user:1");
foreach (var field in user)
{
    Console.WriteLine($"{field.Name}: {field.Value}");
}

// 读取单个字段
var name = await db.HashGetAsync("user:1", "name");
```

### 2.3 List（列表）

适合队列：

```csharp
// 添加到队列
await db.ListLeftPushAsync("task:queue", "task:1");
await db.ListLeftPushAsync("task:queue", "task:2");

// 取出队列（先进先出）
var task = await db.ListRightPopAsync("task:queue");
```

### 2.4 Set（集合）

适合去重、标签：

```csharp
// 添加标签
await db.SetAddAsync("product:1:tags", "手机");
await db.SetAddAsync("product:1:tags", "5G");
await db.SetAddAsync("product:1:tags", "旗舰");

// 获取所有标签
var tags = await db.SetMembersAsync("product:1:tags");

// 判断是否存在
var exists = await db.SetContainsAsync("product:1:tags", "手机");
```

---

## 三、在 WebAPI 中集成 Redis

### 3.1 配置 Redis

```csharp
// Program.cs
builder.Services.AddSingleton<ConnectionMultiplexer>(sp =>
    ConnectionMultiplexer.Connect("localhost:6379"));

builder.Services.AddScoped<IRedisService, RedisService>();
```

### 3.2 封装 Redis 服务

```csharp
// Services/RedisService.cs
using StackExchange.Redis;

public interface IRedisService
{
    Task<T?> GetAsync<T>(string key);
    Task SetAsync<T>(string key, T value, TimeSpan? expiry = null);
    Task RemoveAsync(string key);
}

public class RedisService : IRedisService
{
    private readonly IDatabase _db;

    public RedisService(IConnectionMultiplexer redis)
    {
        _db = redis.GetDatabase();
    }

    public async Task<T?> GetAsync<T>(string key)
    {
        var value = await _db.StringGetAsync(key);
        if (value.IsNullOrEmpty) return default;
        
        return System.Text.Json.JsonSerializer.Deserialize<T>(value!);
    }

    public async Task SetAsync<T>(string key, T value, TimeSpan? expiry = null)
    {
        var json = System.Text.Json.JsonSerializer.Serialize(value);
        await _db.StringSetAsync(key, json, expiry ?? TimeSpan.FromMinutes(30));
    }

    public async Task RemoveAsync(string key)
    {
        await _db.KeyDeleteAsync(key);
    }
}
```

### 3.3 在 Controller 中使用缓存

```csharp
// Controllers/ProductController.cs
[ApiController]
[Route("api/[controller]")]
public class ProductController : ControllerBase
{
    private readonly AppDbContext _db;
    private readonly IRedisService _redis;

    public ProductController(AppDbContext db, IRedisService redis)
    {
        _db = db;
        _redis = redis;
    }

    [HttpGet("{id}")]
    public async Task<Product?> GetById(int id)
    {
        // 先查 Redis
        var cacheKey = $"product:{id}";
        var cached = await _redis.GetAsync<Product>(cacheKey);
        
        if (cached != null)
        {
            Console.WriteLine("从缓存读取");
            return cached;
        }

        // 缓存没有，查数据库
        var product = await _db.Products.FindAsync(id);
        
        if (product != null)
        {
            // 写入缓存，过期时间 30 分钟
            await _redis.SetAsync(cacheKey, product, TimeSpan.FromMinutes(30));
        }
        
        return product;
    }

    [HttpPut("{id}")]
    public async Task<Product?> Update(int id, [FromBody] UpdateProductDto dto)
    {
        var product = await _db.Products.FindAsync(id);
        if (product is null) return null;

        product.Name = dto.Name;
        product.Price = dto.Price;
        product.Stock = dto.Stock;
        
        await _db.SaveChangesAsync();

        // 更新缓存
        await _redis.SetAsync($"product:{id}", product, TimeSpan.FromMinutes(30));
        
        return product;
    }

    [HttpDelete("{id}")]
    public async Task<bool> Delete(int id)
    {
        var product = await _db.Products.FindAsync(id);
        if (product is null) return false;

        _db.Products.Remove(product);
        await _db.SaveChangesAsync();

        // 删除缓存
        await _redis.RemoveAsync($"product:{id}");
        
        return true;
    }
}
```

---

## 四、RabbitMQ 消息队列

### 4.1 什么是消息队列

消息队列用于**异步处理**，提高系统吞吐量：

```
同步：客户端 → API → 数据库 → 返回（等待）
异步：客户端 → API → 消息队列 → 立即返回（后台处理）
```

### 4.2 RabbitMQ 概念

| 概念 | 说明 |
|------|------|
| Producer | 发送消息的一方 |
| Consumer | 接收消息的一方 |
| Queue | 消息存放的队列 |
| Exchange | 路由消息到哪个队列 |

### 4.3 安装 RabbitMQ（Docker）

```powershell
docker run -d -p 5672:5672 -p 15672:15672 rabbitmq:management
```

访问 http://localhost:15672，用户名密码：`guest/guest`

### 4.4 安装客户端

```powershell
dotnet add package RabbitMQ.Client
```

### 4.5 发送消息

```csharp
using RabbitMQ.Client;

public class MessagePublisher
{
    private readonly IConnection _connection;
    private readonly IModel _channel;

    public MessagePublisher()
    {
        var factory = new ConnectionFactory { HostName = "localhost" };
        _connection = factory.CreateConnection();
        _channel = _connection.CreateModel();
    }

    public void Publish(string queueName, string message)
    {
        // 声明队列
        _channel.QueueDeclare(
            queue: queueName,
            durable: false,
            exclusive: false,
            autoDelete: false,
            arguments: null);

        var body = System.Text.Encoding.UTF8.GetBytes(message);
        
        _channel.BasicPublish(
            exchange: "",
            routingKey: queueName,
            basicProperties: null,
            body: body);
            
        Console.WriteLine($"已发送: {message}");
    }
}
```

### 4.6 接收消息

```csharp
public class MessageConsumer
{
    public void Consume(string queueName)
    {
        var factory = new ConnectionFactory { HostName = "localhost" };
        var connection = factory.CreateConnection();
        var channel = connection.CreateModel();

        channel.QueueDeclare(queue: queueName, durable: false, exclusive: false, autoDelete: false);

        var consumer = new EventingBasicConsumer(channel);
        
        consumer.Received += (model, ea) =>
        {
            var body = ea.Body.ToArray();
            var message = System.Text.Encoding.UTF8.GetString(body);
            Console.WriteLine($"收到消息: {message}");
        };

        channel.BasicConsume(queue: queueName, autoAck: true, consumer: consumer);
        Console.WriteLine($"监听队列: {queueName}");
    }
}
```

---

## 五、接口限流

### 5.1 为什么限流

- 防止恶意刷接口
- 保护服务器不过载
- 控制资源成本

### 5.2 简单限流实现（Redis + 滑动窗口）

```csharp
public class RateLimitMiddleware
{
    private readonly RequestDelegate _next;
    private readonly IConnectionMultiplexer _redis;
    private readonly int _maxRequests;
    private readonly TimeSpan _window;

    public RateLimitMiddleware(
        RequestDelegate next,
        IConnectionMultiplexer redis,
        int maxRequests = 100,
        int windowSeconds = 60)
    {
        _next = next;
        _redis = redis;
        _maxRequests = maxRequests;
        _window = TimeSpan.FromSeconds(windowSeconds);
    }

    public async Task InvokeAsync(HttpContext context)
    {
        var clientIp = context.Connection.RemoteIpAddress?.ToString() ?? "unknown";
        var key = $"rate:{clientIp}";

        var db = _redis.GetDatabase();
        var count = await db.StringIncrementAsync(key);

        if (count == 1)
        {
            await db.KeyExpireAsync(key, _window);
        }

        if (count > _maxRequests)
        {
            context.Response.StatusCode = 429;  // Too Many Requests
            await context.Response.WriteAsync("请求太频繁，请稍后再试");
            return;
        }

        await _next(context);
    }
}

// 使用：在 Program.cs 中
app.UseMiddleware<RateLimitMiddleware>();
```

---

## 六、Nginx 反向代理

### 6.1 Nginx 是什么

Nginx 是一个高性能的 HTTP 服务器和反向代理服务器。

### 6.2 常用配置

```nginx
# /etc/nginx/nginx.conf

worker_processes 1;

events {
    worker_connections 1024;
}

http {
    # 负载均衡示例
    upstream backend {
        server 127.0.0.1:5000;
        server 127.0.0.1:5001;
        server 127.0.0.1:5002;
    }

    server {
        listen 80;
        server_name example.com;

        location / {
            proxy_pass http://backend;
            proxy_http_version 1.1;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }

        # 静态文件
        location /static/ {
            alias /var/www/static/;
            expires 30d;
        }
    }
}
```

### 6.3 常用命令

```bash
# 测试配置
nginx -t

# 重载配置
nginx -s reload

# 停止
nginx -s stop
```

---

## 七、JWT 鉴权进阶

### 7.1 JWT 结构

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4ifQ.abc123
     Header                     Payload                    Signature
```

### 7.2 JWT 生成和验证

```csharp
using System.IdentityModel.Tokens.Jwt;
using Microsoft.IdentityModel.Tokens;
using System.Security.Claims;
using System.Text;

public class JwtService
{
    private readonly string _secretKey = "你的密钥，至少32位";
    private readonly int _expiresHours = 24;

    public string GenerateToken(int userId, string username)
    {
        var key = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(_secretKey));
        var credentials = new SigningCredentials(key, SecurityAlgorithms.HmacSha256);

        var claims = new[]
        {
            new Claim(ClaimTypes.NameIdentifier, userId.ToString()),
            new Claim(ClaimTypes.Name, username),
            new Claim("role", "admin")
        };

        var token = new JwtSecurityToken(
            issuer: "MyApi",
            audience: "MyApi",
            claims: claims,
            expires: DateTime.UtcNow.AddHours(_expiresHours),
            signingCredentials: credentials);

        return new JwtSecurityTokenHandler().WriteToken(token);
    }

    public ClaimsPrincipal? ValidateToken(string token)
    {
        var key = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(_secretKey));
        
        var handler = new JwtSecurityTokenHandler();
        try
        {
            var principal = handler.ValidateToken(token, new TokenValidationParameters
            {
                ValidateIssuerSigningKey = true,
                IssuerSigningKey = key,
                ValidateIssuer = true,
                ValidIssuer = "MyApi",
                ValidateAudience = true,
                ValidAudience = "MyApi"
            }, out _);
            
            return principal;
        }
        catch
        {
            return null;
        }
    }
}
```

### 7.3 登录接口

```csharp
[HttpPost("login")]
public async Task<object> Login([FromBody] LoginDto dto)
{
    // 验证用户（假设查数据库）
    var user = await _db.Users.FirstOrDefaultAsync(u => u.Username == dto.Username && u.Password == dto.Password);
    
    if (user == null)
        return Unauthorized("用户名或密码错误");

    var jwt = new JwtService();
    var token = jwt.GenerateToken(user.Id, user.Username);
    
    return new { token, userId = user.Id, username = user.Username };
}
```

---

## 八、作业

### 8.1 必做

- [ ] 搭建 Redis 环境
- [ ] 在 TodoList API 中集成 Redis 缓存
- [ ] 实现 JWT 登录鉴权
- [ ] 实现接口限流
- [ ] 配置 Nginx 反向代理

### 8.2 选做

- [ ] 搭建 RabbitMQ，实现异步消息处理
- [ ] 实现滑动窗口限流
- [ ] 实现 Token 刷新机制

---

## ▶️ 下一步

进入 [Week 13-16：微服务皮毛 + Docker 部署](./phase2-week13-16.md)
