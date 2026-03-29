# Week 5-8：接单技能包 + 数据处理脚本

> 目标：掌握最容易变现的技术，能接单赚钱

---

## 📋 本阶段学习内容

```
Week 5-6：HttpClient + 爬虫基础        ← 数据采集，最快变现
Week 7：  Excel 处理（EPPlus/ClosedXML）← 需求稳定，中文友好
Week 8：  项目部署 + 接单攻略          ← 把作品变成收入
```

---

## 一、HttpClient 网络请求

### 1.1 什么是 HttpClient

用于从代码里发 HTTP 请求，是爬虫和 API 对接的基础。

### 1.2 最简单的 GET 请求

```csharp
using System.Net.Http;

// 方式一：直接用（适合一次性请求）
var client = new HttpClient();
var response = await client.GetStringAsync("https://jsonplaceholder.typicode.com/users");
Console.WriteLine(response);

// 方式二：复用 HttpClient（生产环境推荐）
public class MyService
{
    private static readonly HttpClient _client = new();
    
    public async Task<string> GetData()
    {
        return await _client.GetStringAsync("https://api.example.com/data");
    }
}
```

### 1.3 带参数的请求

```csharp
// GET /api/users?page=1&size=10
var url = "https://api.example.com/users?page=1&size=10";
var response = await client.GetStringAsync(url);
```

### 1.4 POST 发送 JSON

```csharp
using System.Text.Json;

var data = new { name = "张三", age = 25 };
var json = JsonSerializer.Serialize(data);
var content = new StringContent(json, System.Text.Encoding.UTF8, "application/json");

var response = await client.PostAsync("https://api.example.com/users", content);
var result = await response.Content.ReadAsStringAsync();
```

### 1.5 设置请求头

```csharp
var request = new HttpRequestMessage(HttpMethod.Get, url);
request.Headers.Add("Authorization", "Bearer your-token");
request.Headers.Add("User-Agent", "Mozilla/5.0");

var response = await client.SendAsync(request);
```

---

## 二、HTML 解析（HtmlAgilityPack）

### 2.1 安装

```powershell
dotnet add package HtmlAgilityPack
```

### 2.2 解析 HTML 示例

```csharp
using HtmlAgilityPack;

var url = "https://www.example.com/products";
var httpClient = new HttpClient();
var html = await httpClient.GetStringAsync(url);

var htmlDoc = new HtmlDocument();
htmlDoc.LoadHtml(html);

// 选择所有商品名称
var productNodes = htmlDoc.DocumentNode
    .SelectNodes("//div[@class='product-name']");

if (productNodes != null)
{
    foreach (var node in productNodes)
    {
        Console.WriteLine(node.InnerText.Trim());
    }
}
```

### 2.3 常用 XPath 选择器

| XPath | 含义 |
|-------|------|
| `//div` | 所有 div 元素 |
| `//div[@class='name']` | class 为 name 的 div |
| `//a[@href]` | 所有带 href 的链接 |
| `//div[1]` | 第一个 div |
| `//div//span` | div 下的所有 span（不限层级） |
| `.//span` | 当前节点下的 span |

### 2.4 提取链接和图片

```csharp
// 提取所有链接
var links = htmlDoc.DocumentNode
    .SelectNodes("//a[@href]")
    ?.Select(a => a.Attributes["href"].Value)
    .ToList();

// 提取所有图片
var images = htmlDoc.DocumentNode
    .SelectNodes("//img[@src]")
    ?.Select(img => img.Attributes["src"].Value)
    .ToList();
```

---

## 三、实战：爬取招聘网站数据

### 3.1 需求

爬取某网站的职位信息，保存为 JSON 文件。

### 3.2 完整示例

```csharp
using System.Text.Json;
using HtmlAgilityPack;

public class JobInfo
{
    public string Title { get; set; } = "";
    public string Company { get; set; } = "";
    public string Salary { get; set; } = "";
    public string Location { get; set; } = "";
    public string Url { get; set; } = "";
}

public class JobSpider
{
    private static readonly HttpClient _client = new()
    {
        Timeout = TimeSpan.FromSeconds(30)
    };

    public async Task<List<JobInfo>> Crawl(string keyword, int pages = 5)
    {
        var jobs = new List<JobInfo>();

        for (int page = 1; page <= pages; page++)
        {
            var url = $"https://www.example.com/jobs?kw={keyword}&page={page}";
            Console.WriteLine($"正在爬取第 {page} 页...");

            try
            {
                var html = await _client.GetStringAsync(url);
                var doc = new HtmlDocument();
                doc.LoadHtml(html);

                var jobNodes = doc.DocumentNode.SelectNodes("//div[@class='job-item']");

                if (jobNodes == null) continue;

                foreach (var node in jobNodes)
                {
                    var job = new JobInfo
                    {
                        Title = node.SelectSingleNode(".//h3[@class='title']")?.InnerText.Trim() ?? "",
                        Company = node.SelectSingleNode(".//div[@class='company']")?.InnerText.Trim() ?? "",
                        Salary = node.SelectSingleNode(".//span[@class='salary']")?.InnerText.Trim() ?? "",
                        Location = node.SelectSingleNode(".//span[@class='location']")?.InnerText.Trim() ?? "",
                        Url = node.SelectSingleNode(".//a[@href]")?.Attributes["href"].Value ?? ""
                    };

                    jobs.Add(job);
                }

                // 避免请求太快被封，休息 2 秒
                await Task.Delay(2000);
            }
            catch (Exception ex)
            {
                Console.WriteLine($"第 {page} 页失败: {ex.Message}");
            }
        }

        return jobs;
    }

    public async Task SaveToJson(List<JobInfo> jobs, string filePath)
    {
        var json = JsonSerializer.Serialize(jobs, new JsonSerializerOptions
        {
            WriteIndented = true  // 格式化输出
        });
        await File.WriteAllTextAsync(filePath, json);
        Console.WriteLine($"已保存 {jobs.Count} 条数据到 {filePath}");
    }
}

// 使用
var spider = new JobSpider();
var jobs = await spider.Crawl("C# 工程师", pages: 3);
await spider.SaveToJson(jobs, "jobs.json");
```

### 3.3 注意事项

```
⚠️ 爬虫法律风险：
1. 只爬公开数据，不要碰登录后的数据
2. 不要高频请求（设置延时，1-3秒）
3. 遵守网站的 robots.txt
4. 爬来的数据不要商用，只给自己用
```

---

## 四、Excel 处理

### 4.1 安装库

```powershell
dotnet add package ClosedXML  # 推荐，功能强大
dotnet add package EPPlus    # 也常用
```

### 4.2 ClosedXML 读取 Excel

```csharp
using ClosedXML.Excel;

var filePath = "data.xlsx";

using var workbook = new XLWorkbook(filePath);
var worksheet = workbook.Worksheet("Sheet1");

// 读取数据
var rows = worksheet.RangeUsed().RowsUsed().Skip(1); // 跳过表头

foreach (var row in rows)
{
    var name = row.Cell(1).Value.ToString();
    var age = row.Cell(2).Value.GetNumber();
    Console.WriteLine($"{name}, {age}");
}
```

### 4.3 ClosedXML 写入 Excel

```csharp
using ClosedXML.Excel;

var workbook = new XLWorkbook();
var worksheet = workbook.Worksheets.Add("用户数据");

// 写入表头
worksheet.Cell(1, 1).Value = "姓名";
worksheet.Cell(1, 2).Value = "年龄";
worksheet.Cell(1, 3).Value = "城市";

// 写入数据
var users = new[]
{
    new { Name = "张三", Age = 25, City = "北京" },
    new { Name = "李四", Age = 30, City = "上海" },
    new { Name = "王五", Age = 28, City = "深圳" }
};

for (int i = 0; i < users.Length; i++)
{
    worksheet.Cell(i + 2, 1).Value = users[i].Name;
    worksheet.Cell(i + 2, 2).Value = users[i].Age;
    worksheet.Cell(i + 2, 3).Value = users[i].City;
}

// 保存
workbook.SaveAs("output.xlsx");
Console.WriteLine("Excel 已生成！");
```

### 4.4 常见场景

| 场景 | 示例 |
|------|------|
| 数据导入 | 读取 Excel 批量插入数据库 |
| 数据导出 | 查询结果导出 Excel |
| 数据清洗 | 读取、处理、写回 |
| 报表生成 | 格式化的财务报表 |

---

## 五、数据处理工具示例

### 5.1 批量文件重命名

```csharp
using System.Text.RegularExpressions;

var folder = @"C:\Users\Documents\files";
var files = Directory.GetFiles(folder, "*.txt");

foreach (var file in files)
{
    var fileName = Path.GetFileName(file);
    
    // 替换日期格式：20250101 -> 2025-01-01
    var newName = Regex.Replace(fileName, @"(\d{4})(\d{2})(\d{2})", "$1-$2-$3");
    
    var newPath = Path.Combine(folder, newName);
    File.Move(file, newPath);
    
    Console.WriteLine($"{fileName} -> {newName}");
}
```

### 5.2 CSV 处理

```csharp
using CsvHelper;
using CsvHelper.Configuration;

public class Person
{
    public string Name { get; set; } = "";
    public int Age { get; set; }
    public string City { get; set; } = "";
}

// 读取 CSV
using var reader = new StreamReader("people.csv");
using var csv = new CsvReader(reader, new CsvConfiguration(System.Globalization.CultureInfo.InvariantCulture));
var people = csv.GetRecords<Person>().ToList();

// 写入 CSV
using var writer = new StreamWriter("output.csv");
using var csvOut = new CsvWriter(writer, System.Globalization.CultureInfo.InvariantCulture);
csvOut.WriteRecords(people);
```

---

## 六、项目部署

### 6.1 发布为单文件

```powershell
# Windows
dotnet publish -c Release -r win-x64 --self-contained true -p:PublishSingleFile=true

# Linux
dotnet publish -c Release -r linux-x64 --self-contained true -p:PublishSingleFile=true
```

### 6.2 部署到 IIS

1. 安装 .NET Hosting Bundle
2. 在 IIS 中新建网站
3. 设置应用程序池为"无托管代码"
4. 指向发布目录

### 6.3 部署到 Linux (Nginx + Systemd)

```bash
# 1. 安装 Nginx
sudo apt install nginx

# 2. 配置反向代理
sudo nano /etc/nginx/sites-available/default
```

```nginx
server {
    listen 80;
    server_name yourdomain.com;

    location / {
        proxy_pass http://localhost:5000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection keep-alive;
        proxy_set_header Host $host;
    }
}
```

```bash
# 3. 重启 Nginx
sudo systemctl restart nginx

# 4. 设置开机自启
sudo systemctl enable nginx
```

---

## 七、接单全攻略

### 7.1 接单平台推荐

| 平台 | 特点 | 适合阶段 |
|------|------|---------|
| **程序员客栈** | 有保障，不跑单 | 新手首选 |
| **开源众包** | 项目多，竞争激烈 | 练手期 |
| **某宝接单** | 流量大，价格透明 | 想快速起步 |
| **闲鱼** | 可以卖模板 | 有作品后 |
| **熟人介绍** | 最靠谱 | 任何阶段 |

### 7.2 接单流程

```
1. 看到需求 → 2. 评估工时 → 3. 报价 → 4. 谈妥 → 5. 开始 → 6. 交付 → 7. 收款
```

### 7.3 报价策略

| 阶段 | 策略 | 原因 |
|------|------|------|
| 前3单 | 不赚钱，只做作品 | 积累评价和案例 |
| 3-10单 | 按市场价的 70% 报价 | 建立口碑 |
| 10单后 | 按市场价报价 | 有评价支撑 |

**报价公式：**
```
报价 = 预估工时(小时) × 单价(¥80-300) × 难度系数(1.0-1.5)
```

### 7.4 防止被白嫖

- [ ] 需求文档要书面确认（截图、聊天记录）
- [ ] 先收 30-50% 订金再开始
- [ ] 分阶段交付，每阶段收款
- [ ] 交付时提供可运行版本，源码收到尾款再给
- [ ] 明确"免费修改次数"（建议 2-3 次）

### 7.5 常用话术

```
客户：这个多少钱？
你：功能清单确认一下，我先评估工时，明早给你报价。

客户：能不能便宜点？
你：已经是新人价了，保证代码质量，后续维护费用另算。

客户：做好了我再加需求。
你：后续需求按实际工时另计，我们先把这个版本交付了。
```

---

## 八、作业

### 8.1 必做

- [ ] 写一个爬虫脚本（采集公开数据）
- [ ] 完成 Excel 导入导出功能
- [ ] 把 TodoList API 部署到服务器
- [ ] 在程序员客栈注册，完善个人资料
- [ ] 投 3 个标的（别怕被拒）

### 8.2 选做

- [ ] 做一个数据处理工具（爬虫 + Excel 导出）
- [ ] 在闲鱼发布服务
- [ ] 接一个真实小单（哪怕只收 ¥100）

---

## ▶️ 下一步

进入 [阶段二：往高价值走（4-8个月）](./phase2.md)

阶段一完成了，你已经能接单赚钱了。接下来要学更高价值的技术。
