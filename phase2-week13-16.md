# Week 13-16：微服务皮毛 + Docker 部署

> 目标：了解微服务架构，会容器化部署

---

## 一、Docker 基础

### 1.1 什么是 Docker

Docker 是**容器化技术**，让你可以把应用和依赖打包成一个"集装箱"，在任何环境运行。

### 1.2 Docker vs 传统部署

| 方面 | 传统部署 | Docker |
|------|---------|--------|
| 环境 | 每台机器配置 | 一次构建，到处运行 |
| 依赖 | 手动安装，容易冲突 | 隔离打包，不冲突 |
| 扩容 | 买新机器，配置环境 | 一键启动多个容器 |
| 回滚 | 手动恢复 | 回到任意版本 |

### 1.3 安装 Docker

- **Windows：** 下载 [Docker Desktop](https://www.docker.com/products/docker-desktop)
- **安装后打开**，确保任务栏图标显示 Running

### 1.4 Docker 基本概念

```
镜像（Image）：  类的模板，只读的
容器（Container）：镜像的实例，运行中的应用
仓库（Registry）：存放镜像的地方，Docker Hub 是最大的公共仓库
```

---

## 二、Docker 命令

### 2.1 镜像操作

```powershell
# 查看本地镜像
docker images

# 拉取镜像
docker pull redis:latest
docker pull mysql:8.0

# 删除镜像
docker rmi redis:latest

# 构建镜像（稍后讲）
docker build -t myapp:latest .
```

### 2.2 容器操作

```powershell
# 运行容器
docker run -d -p 6379:6379 --name myredis redis:latest
# -d: 后台运行
# -p 6379:6379: 端口映射（主机:容器）
# --name myredis: 容器名称

# 查看运行中的容器
docker ps

# 查看所有容器（包括停止的）
docker ps -a

# 停止容器
docker stop myredis

# 启动已停止的容器
docker start myredis

# 删除容器
docker rm myredis

# 查看容器日志
docker logs -f myredis

# 进入容器内部
docker exec -it myredis bash
```

### 2.3 常用示例

```powershell
# MySQL
docker run -d -p 3306:3306 --name mysql -e MYSQL_ROOT_PASSWORD=123456 mysql:8.0

# Redis
docker run -d -p 6379:6379 --name redis redis:latest

# PostgreSQL
docker run -d -p 5432:5432 -e POSTGRES_PASSWORD=123456 --name postgres postgres:15
```

---

## 三、把 .NET 应用 Docker 化

### 3.1 创建 Dockerfile

在项目根目录创建 `Dockerfile`（无扩展名）：

```dockerfile
# 阶段一：构建
FROM mcr.microsoft.com/dotnet/sdk:9.0 AS build
WORKDIR /src

# 复制项目文件并恢复依赖
COPY *.csproj ./
RUN dotnet restore

# 复制代码并构建
COPY . ./
RUN dotnet publish -c Release -o /app/publish

# 阶段二：运行
FROM mcr.microsoft.com/dotnet/aspnet:9.0 AS runtime
WORKDIR /app

# 复制构建产物
COPY --from=build /app/publish .

# 暴露端口
EXPOSE 5000

# 启动命令
ENTRYPOINT ["dotnet", "MyApi.dll"]
```

### 3.2 .dockerignore

创建 `.dockerignore` 文件，避免把不需要的文件复制进镜像：

```
bin/
obj/
*.md
.git
```

### 3.3 构建镜像

```powershell
cd MyApi
docker build -t myapi:latest .
```

### 3.4 运行容器

```powershell
docker run -d -p 8080:5000 --name myapi myapi:latest
```

访问 http://localhost:8080

---

## 四、Docker Compose 多容器编排

### 4.1 什么是 Docker Compose

一次启动多个容器，并配置它们之间的通信。

### 4.2 docker-compose.yml 示例

```yaml
version: '3.8'

services:
  # Web API 服务
  api:
    build: .
    ports:
      - "8080:5000"
    environment:
      - ASPNETCORE_ENVIRONMENT=Production
      - ConnectionStrings__DefaultConnection=Server=db;Database=MyApp;User=sa;Password=StrongPass123
    depends_on:
      - db
      - redis
    networks:
      - mynetwork

  # 数据库服务
  db:
    image: mcr.microsoft.com/mssql/server:2022-latest
    environment:
      - ACCEPT_EULA=Y
      - SA_PASSWORD=StrongPass123
    ports:
      - "1433:1433"
    volumes:
      - sqldata:/var/opt/mssql
    networks:
      - mynetwork

  # Redis 缓存
  redis:
    image: redis:latest
    ports:
      - "6379:6379"
    networks:
      - mynetwork

  # Nginx 反向代理
  nginx:
    image: nginx:latest
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      - api
    networks:
      - mynetwork

volumes:
  sqldata:

networks:
  mynetwork:
    driver: bridge
```

### 4.3 Docker Compose 命令

```powershell
# 启动所有服务
docker-compose up -d

# 查看运行状态
docker-compose ps

# 查看日志
docker-compose logs -f

# 停止所有服务
docker-compose down

# 重新构建并启动
docker-compose up -d --build
```

---

## 五、Linux 基础

### 5.1 为什么需要 Linux

- Docker 大多运行在 Linux 上
- 很多服务器是 Linux
- 常用命令必须会

### 5.2 常用命令

```bash
# 文件操作
ls -la              # 列出文件
cd /path            # 切换目录
pwd                 # 当前目录
mkdir folder        # 创建文件夹
rm -rf folder       # 删除文件夹
cp file1 file2      # 复制文件
mv file1 file2      # 移动/重命名

# 文本操作
cat file            # 查看文件内容
grep "keyword" file  # 搜索
nano file           # 编辑

# 系统操作
top                 # 查看进程
kill -9 pid         # 强制终止进程
systemctl status    # 查看服务状态
curl http://api     # 测试接口

# 网络
netstat -tlnp       # 查看端口
ps aux | grep name  # 查找进程
```

### 5.3 在 Windows 上练习 Linux

```powershell
# 方法一：WSL（推荐）
wsl --install
# 然后打开 Ubuntu 终端

# 方法二：Docker 里练
docker run -it ubuntu bash
```

---

## 六、微服务入门

### 6.1 什么是微服务

把一个大型应用拆分成多个**小型服务**，每个服务独立运行、独立开发：

```
单体架构：所有功能在一个应用中
           ┌─────────────────────┐
           │  用户 │ 订单 │ 支付  │
           │       │      │       │
           └───────┴──────┴───────┘

微服务架构：每个功能独立成服务
    ┌──────┐  ┌──────┐  ┌──────┐
    │ 用户 │  │ 订单 │  │ 支付 │
    │ 服务 │  │ 服务 │  │ 服务 │
    └──┬───┘──┴───┘──┴───┘
       │
    ┌──┴──┐
    │网关 │
    └─────┘
```

### 6.2 微服务的优缺点

| 优点 | 缺点 |
|------|------|
| 独立部署，不影响其他服务 | 复杂度高（服务间通信） |
| 技术栈可以不同 | 分布式事务问题 |
| 容错性好（一个挂不影响全部） | 运维成本高 |
| 团队独立迭代 | 需要完整的 CI/CD |

### 6.3 什么时候用微服务

✅ 适合：
- 大型团队（20人以上）
- 高并发场景
- 需要独立扩缩容

❌ 不适合：
- 小团队（5人以下）
- 初创项目（变化太快）
- 低并发场景

---

## 七、微服务常用组件

### 7.1 服务注册与发现

服务启动时注册自己，服务调用时查找：

```
Consul / Nacos / Eureka
```

```yaml
# Consul docker-compose 示例
consul:
  image: consul
  ports:
    - "8500:8500"
  command: agent -server -ui -bootstrap-expect=1 -client=0.0.0.0
```

### 7.2 API 网关

统一入口，处理认证、限流、路由：

```
Ocelot / Kong / Nginx
```

### 7.3 熔断降级

防止级联故障：

```
Polly（.NET 库）
```

```csharp
// 使用 Polly 熔断
var policy = Policy
    .Handle<HttpRequestException>()
    .CircuitBreaker(
        exceptionsAllowedBeforeBreaking: 3,
        durationOfBreak: TimeSpan.FromSeconds(30));

var result = policy.Execute(() => httpClient.GetAsync("http://service/api/data"));
```

### 7.4 链路追踪

追踪请求在各个服务间的流转：

```
SkyWalking / Jaeger / Zipkin
```

---

## 八、CI/CD 流水线基础

### 8.1 什么是 CI/CD

| 概念 | 说明 |
|------|------|
| CI（持续集成） | 代码提交后自动构建、测试 |
| CD（持续交付） | 自动部署到测试/预生产环境 |
| CD（持续部署） | 自动部署到生产环境 |

### 8.2 GitHub Actions 示例

创建 `.github/workflows/deploy.yml`：

```yaml
name: Build and Deploy

on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup .NET
        uses: actions/setup-dotnet@v3
        with:
          dotnet-version: '9.0.x'
      
      - name: Restore
        run: dotnet restore
      
      - name: Build
        run: dotnet build --configuration Release
      
      - name: Test
        run: dotnet test
      
      - name: Publish
        run: dotnet publish -c Release -o ./publish
      
      - name: Docker Build
        run: |
          docker build -t myapi:${{ github.sha }} .
          docker tag myapi:${{ github.sha }} myregistry/myapi:latest
          docker push myregistry/myapi:latest
```

---

## 九、作业

### 9.1 必做

- [ ] 安装 Docker Desktop
- [ ] 把自己的 WebAPI 项目 Docker 化
- [ ] 使用 Docker Compose 启动 MySQL + Redis + WebAPI
- [ ] 学习基本 Linux 命令
- [ ] 了解微服务常用组件（注册中心、网关、熔断）

### 9.2 选做

- [ ] 搭建 Consul 服务注册中心
- [ ] 配置 Ocelot API 网关
- [ ] 写一个 GitHub Actions 流水线
- [ ] 部署一个完整的小型微服务架构

---

## ▶️ 下一步

进入 [阶段三：架构师路线（8个月+）](./phase3.md)

你已经具备了中级工程师的技能，接下来向架构师方向深耕。
