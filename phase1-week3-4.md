# Week 3-4：数据库操作（EF Core + SQLite）

> 目标：学会用代码操作数据库，完成数据的增删改查

---

## 一、为什么需要数据库

### 内存 vs 数据库

| 存储方式 | 优点 | 缺点 | 适用场景 |
|---------|------|------|---------|
| 内存变量 | 快、简单 | 程序关闭数据丢失 | 测试、缓存 |
| 数据库 | 持久化、可查询 | 需要学习 SQL | 所有真实项目 |

**一个原则：** 只要数据需要保存超过一次运行，就必须用数据库。

---

## 二、SQLite 介绍

### 2.1 什么是 SQLite

- 轻量级数据库，整个数据库就是一个 `.db` 文件
- **无需安装数据库服务**，适合桌面应用和小型网站
- 数据存在本地文件，不适合高并发

### 2.2 SQLite 适用场景

✅ 适合：
- 小型 WebAPI
- 桌面工具
- 学习练习

❌ 不适合：
- 高并发系统（用 SQL Server / PostgreSQL）
- 多进程同时写入（用真正的数据库）

---

## 三、添加 EF Core 和 SQLite

### 3.1 在项目目录执行

```powershell
cd MyFirstApi
dotnet add package Microsoft.EntityFrameworkCore.Sqlite
dotnet add package Microsoft.EntityFrameworkCore.Design
```

### 3.2 验证安装

打开 `.csproj` 文件，应该看到：

```xml
<ItemGroup>
  <PackageReference Include="Microsoft.EntityFrameworkCore.Sqlite" Version="9.x.x" />
  <PackageReference Include="Microsoft.EntityFrameworkCore.Design" Version="9.x.x" />
</ItemGroup>
```

---

## 四、创建数据模型

### 4.1 产品模型

```csharp
// Models/Product.cs
namespace MyFirstApi.Models;

public class Product
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public decimal Price { get; set; }
    public int Stock { get; set; }
    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
}
```

### 4.2 订单模型（一对多关系）

```csharp
// Models/Order.cs
public class Order
{
    public int Id { get; set; }
    public string OrderNumber { get; set; } = string.Empty;
    public decimal TotalAmount { get; set; }
    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
    
    // 导航属性：一个订单包含多个明细
    public List<OrderItem> Items { get; set; } = new();
}

public class OrderItem
{
    public int Id { get; set; }
    public int OrderId { get; set; }
    public int ProductId { get; set; }
    public int Quantity { get; set; }
    public decimal UnitPrice { get; set; }
    
    // 导航属性
    public Order? Order { get; set; }
    public Product? Product { get; set; }
}
```

---

## 五、创建数据库上下文

### 5.1 什么是 DbContext

DbContext 是 EF Core 的核心类，**相当于数据库的代言人**：

```csharp
// Data/AppDbContext.cs
using Microsoft.EntityFrameworkCore;
using MyFirstApi.Models;

namespace MyFirstApi.Data;

public class AppDbContext : DbContext
{
    // 通过构造函数接收配置（后面会用到）
    public AppDbContext(DbContextOptions<AppDbContext> options) : base(options)
    {
    }

    // 对应数据库中的表
    public DbSet<Product> Products => Set<Product>();
    public DbSet<Order> Orders => Set<Order>();
    public DbSet<OrderItem> OrderItems => Set<OrderItem>();
}
```

### 5.2 配置 Program.cs

```csharp
// Program.cs
using Microsoft.EntityFrameworkCore;
using MyFirstApi.Data;

var builder = WebApplication.CreateBuilder(args);

// 添加 SQLite
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlite("Data Source=app.db"));

var app = builder.Build();

// 自动创建数据库（开发阶段用）
using var scope = app.Services.CreateScope();
var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
db.Database.EnsureCreated();

app.Run();
```

### 5.3 运行并验证

```powershell
dotnet run
```

看到项目启动后，检查项目目录，会生成 `app.db` 文件。

---

## 六、CRUD 完整示例

### 6.1 Product 控制器

```csharp
// Controllers/ProductController.cs
using Microsoft.AspNetCore.Mvc;
using Microsoft.EntityFrameworkCore;
using MyFirstApi.Data;
using MyFirstApi.Models;

[ApiController]
[Route("api/[controller]")]
public class ProductController : ControllerBase
{
    private readonly AppDbContext _db;

    public ProductController(AppDbContext db)
    {
        _db = db;
    }

    // ========== C R E A T E ==========
    [HttpPost]
    public async Task<Product> Create([FromBody] CreateProductDto dto)
    {
        var product = new Product
        {
            Name = dto.Name,
            Price = dto.Price,
            Stock = dto.Stock
        };

        _db.Products.Add(product);
        await _db.SaveChangesAsync();  // 异步保存到数据库
        return product;
    }

    // ========== R E A D ==========
    // 获取所有
    [HttpGet]
    public async Task<List<Product>> GetAll()
    {
        return await _db.Products.ToListAsync();
    }

    // 获取单个
    [HttpGet("{id}")]
    public async Task<Product?> GetById(int id)
    {
        return await _db.Products.FindAsync(id);
    }

    // 条件查询：GET /api/product?name=手机
    [HttpGet]
    public async Task<List<Product>> Search([FromQuery] string? name)
    {
        var query = _db.Products.AsQueryable();
        
        if (!string.IsNullOrEmpty(name))
        {
            query = query.Where(p => p.Name.Contains(name));
        }
        
        return await query.ToListAsync();
    }

    // ========== U P D A T E ==========
    [HttpPut("{id}")]
    public async Task<Product?> Update(int id, [FromBody] UpdateProductDto dto)
    {
        var product = await _db.Products.FindAsync(id);
        if (product is null) return null;

        product.Name = dto.Name;
        product.Price = dto.Price;
        product.Stock = dto.Stock;
        
        await _db.SaveChangesAsync();
        return product;
    }

    // ========== D E L E T E ==========
    [HttpDelete("{id}")]
    public async Task<bool> Delete(int id)
    {
        var product = await _db.Products.FindAsync(id);
        if (product is null) return false;

        _db.Products.Remove(product);
        await _db.SaveChangesAsync();
        return true;
    }
}

// ========== D T O ==========
// 创建时接收的数据
public record CreateProductDto(string Name, decimal Price, int Stock);

// 更新时接收的数据
public record UpdateProductDto(string Name, decimal Price, int Stock);
```

---

## 七、异步编程基础

### 7.1 为什么用 async/await

| 写法 | 特点 |
|------|------|
| 同步 | 等待数据库时阻塞线程 |
| 异步 | 等待时线程可以处理其他请求 |

**结论：** WebAPI 几乎都用异步，否则并发能力差。

### 7.2 规则

```csharp
// 方法返回 Task<T> 而不是 T
public async Task<Product> GetById(int id)
{
    // await 等待异步操作完成
    return await _db.Products.FindAsync(id);
}

// 调用时用 await
var product = await productController.GetById(1);
```

### 7.3 常见的异步方法

| 同步方法 | 异步版本 | 说明 |
|---------|---------|------|
| .ToList() | .ToListAsync() | 查询列表 |
| .FindAsync() | .FindAsync() | 按主键查 |
| .SaveChanges() | .SaveChangesAsync() | 保存 |
| .Add() | .AddAsync() | 添加 |

---

## 八、进阶查询

### 8.1 分页查询

```csharp
[HttpGet("page")]
public async Task<object> GetPage(
    [FromQuery] int page = 1,
    [FromQuery] int pageSize = 10)
{
    var total = await _db.Products.CountAsync();
    var items = await _db.Products
        .OrderBy(p => p.Id)
        .Skip((page - 1) * pageSize)
        .Take(pageSize)
        .ToListAsync();

    return new
    {
        total,
        page,
        pageSize,
        items
    };
}
```

### 8.2 排序

```csharp
// 按价格升序
_db.Products.OrderBy(p => p.Price)

// 按价格降序
_db.Products.OrderByDescending(p => p.Price)

// 按创建时间降序
_db.Products.OrderByDescending(p => p.CreatedAt)
```

### 8.3 聚合查询

```csharp
// 统计数量
var count = await _db.Products.CountAsync();

// 计算平均价格
var avgPrice = await _db.Products.AverageAsync(p => p.Price);

// 计算最高价格
var maxPrice = await _db.Products.MaxAsync(p => p.Price);
```

---

## 九、作业

### 9.1 必做

- [ ] 在现有项目中添加 EF Core + SQLite
- [ ] 完成 Product 的完整 CRUD
- [ ] 实现分页查询
- [ ] 用 Postman 测试所有接口
- [ ] 实现模糊查询（根据名称搜索）

### 9.2 选做

- [ ] 添加 Category（分类）表，和 Product 关联（一对多）
- [ ] 实现分类的 CRUD
- [ ] 查询某分类下的所有产品

---

## ▶️ 下一步

进入 [Week 5-8：接单技能包 + 数据处理脚本](./phase1-week5-8.md)

学完这部分，你就能真正开始接单赚钱了！
