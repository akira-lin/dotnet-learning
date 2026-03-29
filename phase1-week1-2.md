# Week 1-2：.NET 基础 + 第一个 WebAPI

> 目标：理解 .NET 开发流程，能写并测试简单接口

---

## 一、环境准备

### 1.1 安装 .NET 9 SDK

1. 访问 https://dotnet.microsoft.com/download/dotnet/9.0
2. 下载 **.NET SDK 9.0**（不是运行时）
3. 安装后打开命令行验证：

```powershell
dotnet --version
# 应该显示：9.0.x
```

### 1.2 安装 Visual Studio 2022

1. 访问 https://visualstudio.microsoft.com/downloads
2. 下载 **Visual Studio 2022 Community**（免费）
3. 安装时勾选：
   - **ASP.NET 和 Web 开发**
   - **.NET 9 个性化设置**（如有）

### 1.3 安装 Postman（API 测试工具）

1. 访问 https://www.postman.com/downloads
2. 下载并注册（免费）
3. 用来测试你写的接口

---

## 二、创建第一个 WebAPI 项目

### 2.1 命令行创建

```powershell
# 创建项目
dotnet new webapi -n MyFirstApi -o MyFirstApi

# 进入项目目录
cd MyFirstApi

# 查看项目结构
dir
```

### 2.2 项目结构说明

```
MyFirstApi/
├── Controllers/          # 控制器文件夹
├── Program.cs            # 入口文件（配置服务、启动）
├── appsettings.json      # 配置文件
└── MyFirstApi.csproj     # 项目文件
```

### 2.3 运行项目

```powershell
dotnet run
```

看到类似输出说明启动成功：
```
Now listening on: http://localhost:5000
Now listening on: https://localhost:5001
```

按 `Ctrl+C` 停止。

---

## 三、理解 HTTP 协议

### 3.1 四种基本请求

| 操作 | HTTP 方法 | 用途 | 例子 |
|------|----------|------|------|
| 查 | GET | 获取数据 | 读取用户信息 |
| 增 | POST | 创建数据 | 新增一条订单 |
| 改 | PUT/PATCH | 更新数据 | 修改密码 |
| 删 | DELETE | 删除数据 | 删除帖子 |

### 3.2 请求和响应格式

**请求（浏览器地址栏只能发 GET）：**
```
GET https://api.example.com/users/1
POST https://api.example.com/users
Content-Type: application/json

{"name": "张三", "age": 25}
```

**响应：**
```json
HTTP/1.1 200 OK
Content-Type: application/json

{"id": 1, "name": "张三", "age": 25}
```

### 3.3 状态码

| 状态码 | 含义 |
|-------|------|
| 200 | 成功 |
| 201 | 创建成功 |
| 400 | 请求错误（参数不对） |
| 404 | 找不到 |
| 500 | 服务器错误 |

---

## 四、第一个接口：Hello World

### 4.1 创建控制器

在 `Controllers` 文件夹下新建文件 `HelloController.cs`：

```csharp
using Microsoft.AspNetCore.Mvc;

namespace MyFirstApi.Controllers;

[ApiController]
[Route("api/[controller]")]
public class HelloController : ControllerBase
{
    // GET api/hello
    [HttpGet]
    public string Get()
    {
        return "Hello World!";
    }

    // GET api/hello/张三
    [HttpGet("{name}")]
    public string GetWithName(string name)
    {
        return $"Hello {name}!";
    }

    // POST api/hello
    [HttpPost]
    public object Post([FromBody] object data)
    {
        return new { message = "收到数据", data = data };
    }
}
```

### 4.2 用 Postman 测试

1. 打开 Postman
2. 新建请求：

**测试 1 - GET：**
```
Method: GET
URL: http://localhost:5000/api/hello
```

**测试 2 - GET 带参数：**
```
Method: GET
URL: http://localhost:5000/api/hello/张三
```

**测试 3 - POST：**
```
Method: POST
URL: http://localhost:5000/api/hello
Body (raw JSON):
{"name": "李四", "age": 30}
```

---

## 五、用 JSON 返回数据

### 5.1 定义返回模型

```csharp
// Models/User.cs
namespace MyFirstApi.Models;

public class User
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public int Age { get; set; }
}
```

### 5.2 改写控制器

```csharp
// Controllers/UserController.cs
using Microsoft.AspNetCore.Mvc;
using MyFirstApi.Models;

[ApiController]
[Route("api/[controller]")]
public class UserController : ControllerBase
{
    // 模拟数据库
    private static List<User> _users = new()
    {
        new User { Id = 1, Name = "张三", Age = 25 },
        new User { Id = 2, Name = "李四", Age = 30 }
    };

    // GET api/user
    [HttpGet]
    public List<User> GetAll()
    {
        return _users;
    }

    // GET api/user/1
    [HttpGet("{id}")]
    public User? GetById(int id)
    {
        return _users.FirstOrDefault(u => u.Id == id);
    }

    // POST api/user
    [HttpPost]
    public User Create([FromBody] CreateUserDto dto)
    {
        var user = new User
        {
            Id = _users.Max(u => u.Id) + 1,
            Name = dto.Name,
            Age = dto.Age
        };
        _users.Add(user);
        return user;
    }
}

// DTO：只接收需要的数据
public record CreateUserDto(string Name, int Age);
```

### 5.3 Postman 测试

**GET 列表：**
```
GET http://localhost:5000/api/user
```

**GET 单个：**
```
GET http://localhost:5000/api/user/1
```

**POST 新增：**
```
POST http://localhost:5000/api/user
Body (raw JSON):
{"name": "王五", "age": 28}
```

---

## 六、依赖注入入门（会用即可）

### 6.1 什么是依赖注入

简单理解：**不用 `new`，让系统帮你创建对象**

```csharp
// ❌ 不用这种方式（紧耦合，难以测试）
public class UserService
{
    private readonly UserDAL _dal = new UserDAL();
}

// ✅ 依赖注入（松耦合，便于测试和替换）
public class UserService
{
    private readonly UserDAL _dal;
    
    public UserService(UserDAL dal)
    {
        _dal = dal;
    }
}
```

### 6.2 .NET 中的依赖注入

在 `Program.cs` 中注册服务：

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

// 注册服务
builder.Services.AddScoped<IUserService, UserService>();

// 或者单例（整个程序只有一个实例）
builder.Services.AddSingleton<IConfig, Config>();

// 或者瞬态（每次请求都创建新实例）
builder.Services.AddTransient<IEmailService, EmailService>();

var app = builder.Build();
```

### 6.3 三种生命周期的区别

| 方式 | 使用场景 | 例子 |
|------|---------|------|
| AddSingleton | 全局只创建一个 | 配置类、日志 |
| AddScoped | 每个请求创建一个 | 数据库上下文 |
| AddTransient | 每次使用都创建 | 工具类 |

**初期不用深究，用 AddScoped 就行。**

---

## 七、作业

### 7.1 必做

- [ ] 创建自己的 WebAPI 项目
- [ ] 实现以下接口：

| 方法 | URL | 功能 |
|------|-----|------|
| GET | /api/product | 获取产品列表 |
| GET | /api/product/{id} | 获取单个产品 |
| POST | /api/product | 新增产品 |
| PUT | /api/product/{id} | 更新产品 |
| DELETE | /api/product/{id} | 删除产品 |

产品字段：`Id`、`Name`（名称）、`Price`（价格）、`Stock`（库存）

- [ ] 用 Postman 测试所有接口
- [ ] 把代码传到 GitHub

### 7.2 选做

- [ ] 实现分页查询：`GET /api/product?page=1&size=10`
- [ ] 实现条件查询：`GET /api/product?name=手机`

---

## ▶️ 下一步

进入 [Week 3-4：数据库操作（EF Core + SQLite）](./phase1-week3-4.md)

学完数据库操作后，你的接口才能真正保存数据！
