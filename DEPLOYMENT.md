# 粤味鲜选·岭南特色生鲜商城 - 项目部署文档

> 版本: 1.0.0  
> 日期: 2026-07-06  
> 适用环境: 开发环境 / 测试环境 / 生产环境（需额外安全加固）

---

## 一、项目概述

| 项目 | 说明 |
|------|------|
| 项目名称 | 粤味鲜选·岭南特色生鲜商城 (mall-lingnan) |
| 架构模式 | Spring Cloud Alibaba 微服务架构 |
| 服务数量 | 9 个微服务 + 1 网关 + 2 前端 |
| 数据库 | MySQL 8.0 |
| 缓存 | Redis 5.0+ |
| 注册中心 | Nacos 2.3.2 |
| 分布式事务 | Seata 1.8.0（当前各服务均禁用） |
| 搜索引擎 | Elasticsearch 7.x+ |
| AI 客服 | Spring AI + 阿里云 DashScope (qwen-turbo) |

### 服务清单与端口

| 服务名 | 端口 | 说明 |
|--------|------|------|
| mall-gateway | 8080 | API 网关，统一入口 |
| mall-user | 8081 | 用户服务（注册/登录/地址） |
| mall-product | 8082 | 商品服务（商品/分类/ES搜索） |
| mall-stock | 8083 | 库存服务（库存/入库/出库） |
| mall-cart | 8084 | 购物车服务 |
| mall-order | 8085 | 订单服务（订单/支付回调/WebSocket） |
| mall-pay | 8086 | 支付服务（创建支付/回调签名验证） |
| mall-admin | 8087 | 管理后台服务（统计/发货） |
| mall-ai | 8088 | AI 客服服务（Spring Boot 3.2.0） |
| Nacos | 8848 | 服务注册与配置中心 |
| Seata | 7091 | 分布式事务协调器 |
| Redis | 6379 | 缓存/分布式锁/幂等 |
| MySQL | 3306 | 业务数据库 |
| ES | 9200 | 商品搜索引擎 |
| 用户端前端 | 3000 | Vite + Vue3 + Element Plus |
| 管理端前端 | 3001 | Vite + Vue3 + Element Plus |

---

## 二、环境要求

### 2.1 开发环境

| 组件 | 版本要求 | 说明 |
|------|----------|------|
| JDK | 1.8 (服务) / 17 (mall-ai) | mall-ai 必须使用 Java 17+ |
| Maven | 3.8+ | 使用阿里云镜像加速 |
| Node.js | 18+ | 前端构建 |
| MySQL | 8.0+ | 字符集 utf8mb4 |
| Redis | 5.0+ | |
| Nacos | 2.3.2 | 单机模式 standalone |
| Elasticsearch | 7.x | 商品搜索（可选，未安装时 ES 操作会降级） |

### 2.2 生产环境额外要求

- **服务器**: 至少 4 核 8G（微服务全部部署在一台机器上）
- **JVM 参数**: 每个服务建议 `-Xms256m -Xmx512m`，mall-ai 建议 `-Xms512m -Xmx1g`
- **Nginx**: 反向代理 + SSL 证书 + 静态资源缓存
- **防火墙**: 仅暴露 80/443 和 Nacos 控制台端口

---

## 三、基础设施部署

### 3.1 MySQL 数据库

```bash
# 创建数据库并导入初始化脚本
mysql -u root -p < sql/mall_lingnan.sql
mysql -u root -p < sql/seata_setup.sql
mysql -u root -p < sql/seata_server_tables.sql
```

数据库清单：
- `mall_lingnan` - 业务数据库（用户/商品/订单/库存/支付）
- `seata` - Seata 分布式事务存储（如需启用）

### 3.2 Redis

```bash
# Windows
redis-server.exe redis.windows.conf

# Linux
redis-server /etc/redis/redis.conf
```

### 3.3 Nacos

```bash
# Windows 单机模式
cd nacos/bin
startup.cmd -m standalone

# Linux 单机模式
cd nacos/bin
sh startup.sh -m standalone
```

控制台: http://localhost:8848/nacos  
默认账号: `nacos` / `nacos`

### 3.4 Elasticsearch（可选）

```bash
# 启动 ES（mall-product 依赖）
bin/elasticsearch
```

如未安装 ES，mall-product 的 ES 相关操作会在 catch 块中降级，不影响核心功能。

### 3.5 Seata（当前各服务已禁用，如需启用）

```bash
# 启动 Seata Server
cd seata-server
sh start-seata.bat  # Windows
```

**注意**: 当前所有微服务 `seata.enabled: false`，分布式事务未生效。如需启用，需：
1. 修改各服务 `application.yml` 中 `seata.enabled: true`
2. 确保 Seata Server 已启动并注册到 Nacos
3. 在需要分布式事务的方法上添加 `@GlobalTransactional`

---

## 四、后端服务编译与部署

### 4.1 编译打包

```bash
# 进入项目根目录
cd d:\yueqian\mall-lingnan

# 编译父工程及所有子模块（跳过测试）
mvn clean install -DskipTests

# 如需单独编译某个模块
mvn clean package -pl mall-product -DskipTests
```

编译产物位于各模块 `target/` 目录下：
- `mall-gateway/target/mall-gateway.jar`
- `mall-user/target/mall-user.jar`
- ...（以此类推）

**注意**: mall-ai 使用 Spring Boot 3.2.0，需 Java 17+ 编译运行。如果 `java -jar` 启动失败，改用：
```bash
cd mall-ai
mvn spring-boot:run
```

### 4.2 启动顺序

**必须按顺序启动**，确保依赖服务先就绪：

```
1. MySQL (3306)
2. Redis (6379)
3. Nacos (8848) - 等待 http://localhost:8848/nacos 可访问
4. Elasticsearch (9200) - 可选
5. Seata (7091) - 可选（当前禁用）
6. mall-gateway (8080)
7. mall-user (8081)
8. mall-product (8082)
9. mall-stock (8083)
10. mall-cart (8084)
11. mall-order (8085)
12. mall-pay (8086)
13. mall-admin (8087)
14. mall-ai (8088) - 最后启动，启动较慢
```

### 4.3 启动脚本

项目已提供 `start-all.bat`，但**不完整**，缺少 mall-pay、mall-cart、mall-admin、mall-ai、Redis。建议改用以下完整启动方式：

**Windows 完整启动脚本示例:**

```batch
@echo off
chcp 65001 >nul

echo [1/6] 启动 MySQL / Redis / Nacos ...
:: 请确保 MySQL 和 Redis 已在外部启动
start "Nacos" cmd /k "cd /d d:\yueqian\mall-lingnan\nacos\bin && startup.cmd -m standalone"
timeout /t 30 /nobreak >nul

echo [2/6] 启动 mall-gateway...
start "gateway" cmd /k "java -jar mall-gateway/target/mall-gateway.jar"
timeout /t 5 /nobreak >nul

echo [3/6] 启动基础服务...
start "user" cmd /k "java -jar mall-user/target/mall-user.jar"
start "product" cmd /k "java -jar mall-product/target/mall-product.jar"
start "stock" cmd /k "java -jar mall-stock/target/mall-stock.jar"
timeout /t 5 /nobreak >nul

echo [4/6] 启动业务服务...
start "cart" cmd /k "java -jar mall-cart/target/mall-cart.jar"
start "order" cmd /k "java -jar mall-order/target/mall-order.jar"
start "pay" cmd /k "java -jar mall-pay/target/mall-pay.jar"
start "admin" cmd /k "java -jar mall-admin/target/mall-admin.jar"
timeout /t 5 /nobreak >nul

echo [5/6] 启动 AI 服务...
start "ai" cmd /k "java -jar mall-ai/target/mall-ai-1.0.0.jar"
timeout /t 20 /nobreak >nul

echo [6/6] 所有后端服务已启动！
echo 用户端: http://localhost:3000
echo 管理端: http://localhost:3001
echo Nacos:  http://localhost:8848/nacos
```

**Linux 启动脚本示例:**

```bash
#!/bin/bash
set -e

PROJECT_DIR="/opt/mall-lingnan"
cd $PROJECT_DIR

# 启动顺序函数
start_service() {
    local name=$1
    local jar=$2
    echo "Starting $name..."
    nohup java -Xms256m -Xmx512m -jar $jar > logs/$name.log 2>&1 &
    sleep 3
}

mkdir -p logs

start_service "gateway" "mall-gateway/target/mall-gateway.jar"
start_service "user" "mall-user/target/mall-user.jar"
start_service "product" "mall-product/target/mall-product.jar"
start_service "stock" "mall-stock/target/mall-stock.jar"
start_service "cart" "mall-cart/target/mall-cart.jar"
start_service "order" "mall-order/target/mall-order.jar"
start_service "pay" "mall-pay/target/mall-pay.jar"
start_service "admin" "mall-admin/target/mall-admin.jar"

# mall-ai 需 Java 17
nohup java -Xms512m -Xmx1g -jar mall-ai/target/mall-ai-1.0.0.jar > logs/ai.log 2>&1 &

echo "All services started. Check logs/ directory."
```

---

## 五、前端部署

### 5.1 用户端 (mall-web)

```bash
cd mall-web
npm install
npm run dev        # 开发模式，端口 3000
npm run build      # 生产构建，产物在 dist/ 目录
```

### 5.2 管理端 (mall-admin-web)

```bash
cd mall-admin-web
npm install
npm run dev        # 开发模式，端口 3001
npm run build      # 生产构建
```

### 5.3 生产环境 Nginx 配置示例

```nginx
server {
    listen 80;
    server_name mall.yourdomain.com;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl;
    server_name mall.yourdomain.com;

    ssl_certificate /path/to/cert.pem;
    ssl_certificate_key /path/to/key.pem;

    # 用户端前端
    location / {
        root /opt/mall-lingnan/mall-web/dist;
        index index.html;
        try_files $uri $uri/ /index.html;
    }

    # API 统一走网关
    location /api/ {
        proxy_pass http://127.0.0.1:8080/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }

    # 图片资源
    location /images/ {
        proxy_pass http://127.0.0.1:8082/;
        add_header Cache-Control "no-cache";
    }

    # WebSocket
    location /ws/order {
        proxy_pass http://127.0.0.1:8085;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}

server {
    listen 443 ssl;
    server_name admin.yourdomain.com;

    ssl_certificate /path/to/cert.pem;
    ssl_certificate_key /path/to/key.pem;

    location / {
        root /opt/mall-lingnan/mall-admin-web/dist;
        index index.html;
        try_files $uri $uri/ /index.html;
    }

    location /api/ {
        proxy_pass http://127.0.0.1:8080/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

---

## 六、配置说明

### 6.1 各服务 application.yml 关键配置

所有服务的 MySQL、Redis 连接配置需根据实际环境修改：

```yaml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/mall_lingnan?useUnicode=true&characterEncoding=utf-8&useSSL=false&serverTimezone=Asia/Shanghai&allowPublicKeyRetrieval=true
    username: root
    password: YOUR_PASSWORD_HERE    # 生产环境必须修改！
  data:
    redis:
      host: 127.0.0.1
      port: 6379
```

### 6.2 必须修改的配置项

| 配置项 | 位置 | 建议 |
|--------|------|------|
| MySQL 密码 | 所有服务 `application.yml` | 使用环境变量或 Nacos 配置中心 |
| Redis 地址 | 所有服务 `application.yml` | 如 Redis 在远程服务器需修改 |
| AI API Key | `mall-ai/application.yml` | 替换为自己的 DashScope API Key |
| ES 地址/密码 | `mall-product/application.yml` | 如使用远程 ES 集群需修改 |
| 图片上传路径 | `mall-product/application.yml` | Linux 环境改为 `/opt/mall-lingnan/images/products/` |
| JWT Secret | 各服务 `jwt.secret` | 生产环境必须使用强随机字符串 |
| 支付回调密钥 | `mall-pay/application.yml` | 生产环境必须更换 |

### 6.3 Nacos 配置中心（可选）

项目已提供 `nacos-config/` 目录下的各服务配置，可导入 Nacos 配置中心：

```bash
# 在 Nacos 控制台 -> 配置管理 -> 配置列表中导入
# Data ID: mall-user-dev.yml
# Group: DEFAULT_GROUP
# 配置格式: YAML
```

使用配置中心后，各服务本地 `application.yml` 可精简为仅保留 `spring.cloud.nacos.config` 相关配置。

---

## 七、代码审查问题清单

### 严重问题（生产部署前必须修复）

| # | 问题 | 风险 | 修复建议 |
|---|------|------|----------|
| 1 | **数据库密码硬编码** | 所有 8 个服务 + seata 的 `application.yml` 和 `nacos-config/` 中硬编码密码 `cjy20050814` | 使用环境变量 `${MYSQL_PASSWORD}` 或 Nacos 配置中心 |
| 2 | **AI API Key 硬编码** | `mall-ai/application.yml` 中阿里云 API Key 明文存储 | 使用环境变量 `${DASHSCOPE_API_KEY}` |
| 3 | **网关 globalcors 配置冲突** | `gateway/application.yml` 中 `allowedOriginPatterns: "*"` 与 `CorsConfig.java` 限制配置并存，可能冲突 | 删除 `globalcors` 配置，统一使用 `CorsConfig.java` 管理 |
| 4 | **管理端前端绕过网关** | `mall-admin-web/vite.config.js` 直接代理到各微服务（8081-8088），绕过网关鉴权 | 管理端前端也应统一走网关 `8080`，由网关做鉴权和路由 |
| 5 | **start-all.bat 不完整** | 缺少 mall-pay、mall-cart、mall-admin、mall-ai、Redis 启动 | 使用文档中提供的完整启动脚本 |
| 6 | **actuator 全暴露** | 所有服务 `management.endpoints.web.exposure.include: "*"` | 生产环境仅暴露 `health`，并添加安全认证 |
| 7 | **MyBatis-Plus SQL 日志输出到控制台** | 多个服务配置 `log-impl: org.apache.ibatis.logging.stdout.StdOutImpl` | 生产环境关闭或改用文件日志 |

### 中等问题（建议修复）

| # | 问题 | 说明 | 修复建议 |
|---|------|------|----------|
| 8 | **Seata 全部禁用** | 所有服务 `seata.enabled: false`，`@GlobalTransactional` 注解不生效 | 如需分布式事务，启用 Seata 并确保 TC 正常注册 |
| 9 | **mall-ai Java 版本不兼容** | mall-ai 使用 Spring Boot 3.2.0（需 Java 17），其他服务使用 2.6.15（Java 8） | 统一升级至 Spring Boot 3.x + Java 17，或 mall-ai 独立部署 |
| 10 | **网关路由写死 IP** | 网关使用 `http://127.0.0.1:port` 而非 `lb://service-name` | 改为 `uri: lb://mall-user` 等服务发现方式 |
| 11 | **ES 密码硬编码** | `mall-product/application.yml` 中 `elastic/elastic` | 使用环境变量 |
| 12 | **图片上传路径写死 Windows 路径** | `upload.image-path: d:/yueqian/...` | Linux 部署时需修改为绝对路径 |
| 13 | **Nacos 配置中心未真正使用** | 服务本地 `application.yml` 已包含完整配置，未从 Nacos 读取 | 将敏感配置迁移到 Nacos，本地仅保留 bootstrap 配置 |

### 轻微问题

| # | 问题 | 说明 |
|---|------|------|
| 14 | mall-admin 排除了 Redis 自动配置 | `autoconfigure.exclude` 中排除了 Redis，但未提供替代方案，如果需要缓存会失败 |
| 15 | 各服务 Redis 配置不一致 | 部分服务缺少 `lettuce.pool` 配置，部分缺少 `database` 设置 |
| 16 | Druid 监控未配置 | 未开启 Druid StatViewServlet，无法查看 SQL 监控和连接池状态 |

---

## 八、安全加固建议

### 8.1 必须执行

1. **修改所有默认密码**
   - MySQL root 密码
   - Redis 密码（配置 `requirepass`）
   - Nacos 默认账号 `nacos/nacos`
   - Seata 控制台密码
   - Elasticsearch `elastic` 密码

2. **敏感配置外置**
   ```yaml
   # 示例：使用环境变量
   spring:
     datasource:
       password: ${MYSQL_PASSWORD:default_password}
   ```

3. **关闭开发调试端点**
   ```yaml
   management:
     endpoints:
       web:
         exposure:
           include: health  # 仅暴露健康检查
   ```

4. **启用 HTTPS**
   - 生产环境必须使用 SSL 证书
   - 用户端和管理端强制 HTTPS 跳转

5. **图片上传安全**
   - 限制上传文件类型（仅允许 jpg/png/webp）
   - 限制文件大小（当前 10MB）
   - 对上传文件名进行随机重命名（已实现 UUID）

### 8.2 建议执行

1. **日志集中收集**: 使用 ELK 或 Loki 收集各服务日志
2. **服务监控**: 接入 Prometheus + Grafana 监控 JVM、接口 QPS、响应时间
3. **限流熔断**: Sentinel 已引入但未配置规则，建议配置流控和降级规则
4. **数据库连接池监控**: 开启 Druid 监控页面，配置访问白名单

---

## 九、验证部署成功

服务全部启动后，按以下顺序验证：

```bash
# 1. Nacos 服务注册检查
curl http://localhost:8848/nacos/v1/ns/service/list?pageNo=1&pageSize=10
# 期望返回 8 个服务（mall-ai 可能不注册在 Nacos）

# 2. 用户端首页
curl http://localhost:3000

# 3. 管理端首页
curl http://localhost:3001

# 4. 网关白名单接口（无需登录）
curl http://localhost:8080/api/product/list?pageNum=1&pageSize=5

# 5. 用户注册/登录流程
# 使用浏览器或 Postman 测试完整购物流程

# 6. WebSocket 连接测试
# 浏览器控制台: new WebSocket('ws://localhost:8085/ws/order?token=YOUR_JWT')
```

---

## 十、故障排查

| 现象 | 可能原因 | 解决方案 |
|------|----------|----------|
| 服务启动后 Nacos 看不到 | Nacos 未启动或服务注册失败 | 检查 Nacos 日志，确认 `spring.cloud.nacos.discovery` 配置 |
| 网关 404 | 路由配置错误或目标服务未启动 | 检查网关 `application.yml` 路由配置，确认 StripPrefix 过滤器 |
| 数据库连接失败 | 密码错误或 MySQL 未启动 | 检查 `application.yml` 数据库配置，确认 MySQL 服务状态 |
| mall-ai 启动失败 | Java 版本不兼容 | 确保使用 Java 17+，或改用 `mvn spring-boot:run` |
| 支付回调 403 | 缺少内部调用标识或签名错误 | 检查 `X-Internal-Call` 和 `X-Sign` 请求头 |
| WebSocket 连接失败 | Token 无效或拦截器拒绝 | 检查 JWT Token 是否有效，是否通过 URL 参数传递 |
| 图片上传失败 | 目录不存在或无写入权限 | 确保 `upload.image-path` 目录存在且服务有写入权限 |
| 订单创建后库存未扣减 | Seata 禁用或 Feign 调用失败 | 检查 Seata 配置，确认 mall-stock 服务正常 |

---

## 十一、附录

### 11.1 项目目录结构

```
mall-lingnan/
├── mall-common/          # 公共模块（工具类/实体/异常）
├── mall-gateway/         # API 网关（路由/鉴权/CORS）
├── mall-user/            # 用户服务
├── mall-product/         # 商品服务（含 ES 搜索）
├── mall-stock/           # 库存服务
├── mall-cart/            # 购物车服务
├── mall-order/           # 订单服务（含 WebSocket）
├── mall-pay/             # 支付服务（含签名验证）
├── mall-admin/           # 管理后台服务
├── mall-ai/              # AI 客服服务（Spring Boot 3.x）
├── mall-web/             # 用户端前端
├── mall-admin-web/       # 管理端前端
├── nacos/                # Nacos 服务端（内嵌）
├── seata-server/         # Seata 服务端
├── sql/                  # 数据库初始化脚本
├── nacos-config/         # Nacos 配置文件（未完全启用）
├── images/               # 图片上传存储目录
└── jmeter-test/          # JMeter 压测脚本与报告
```

### 11.2 技术栈版本

| 技术 | 版本 |
|------|------|
| Spring Boot | 2.6.15（mall-ai 使用 3.2.0） |
| Spring Cloud | 2021.0.8 |
| Spring Cloud Alibaba | 2021.0.6.2 |
| MyBatis-Plus | 3.5.5 |
| MySQL Connector | 8.0.33 |
| Druid | 1.2.20 |
| JJWT | 0.11.5 |
| Hutool | 5.8.25 |
| Vue | 3.4.0 |
| Vite | 5.0.0 |
| Element Plus | 2.4.4 |

---

**文档维护**: 部署前请核对上述问题清单，确保严重问题已修复。生产环境部署强烈建议使用 Docker / Kubernetes 容器化方案。
