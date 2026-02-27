# TestHub Platform Docker 部署指南

## 📦 项目结构

```
testhub_platform/
├── Dockerfile                 # 后端容器构建文件
├── docker-compose.yml          # 容器编排文件
├── .dockerignore             # Docker 构建忽略文件
├── frontend/
│   ├── Dockerfile            # 前端容器构建文件
│   └── nginx.conf           # Nginx 配置文件
├── backend/                 # Django 后端代码
├── frontend/                # Vue3 前端代码
└── requirements.txt          # Python 依赖包
```

## 🚀 快速部署

### 前置要求

- Docker 已安装
- Docker Compose 已安装
- 至少 2GB 可用内存
- 至少 10GB 可用磁盘空间

### 部署步骤

#### 1. 克隆项目（如果从 Git）

```bash
git clone https://github.com/your-username/testhub_platform.git
cd testhub_platform
```

#### 2. 配置环境变量（可选）

编辑 `docker-compose.yml`，修改数据库密码等敏感信息：

```yaml
environment:
  - DB_PASSWORD=your_secure_password  # 修改数据库密码
  - DJANGO_SECRET_KEY=your_secure_secret_key  # 修改 Django 密钥
  - ALLOWED_HOSTS=your_server_ip  # 修改允许的访问主机
```

#### 3. 启动服务

```bash
# 构建并启动所有服务
docker-compose up -d --build

# 查看服务状态
docker-compose ps

# 查看日志
docker-compose logs -f
```

#### 4. 初始化数据库（首次部署）

```bash
# 进入后端容器
docker-compose exec backend python manage.py migrate

# 创建超级用户
docker-compose exec backend python manage.py createsuperuser

# 收集静态文件
docker-compose exec backend python manage.py collectstatic --noinput
```

## 🌐 访问应用

- **前端**: http://localhost 或 http://your-server-ip
- **后端 API**: http://localhost:8000 或 http://your-server-ip:8000
- **默认登录**: superuser / (需要在容器中设置密码)

## 🔧 常用命令

### 查看日志

```bash
# 查看所有服务日志
docker-compose logs -f

# 查看特定服务日志
docker-compose logs backend
docker-compose logs frontend
docker-compose logs db
```

### 重启服务

```bash
# 重启所有服务
docker-compose restart

# 重启特定服务
docker-compose restart backend
docker-compose restart frontend
```

### 停止服务

```bash
# 停止所有服务
docker-compose down

# 停止并删除容器
docker-compose down -v
```

### 更新代码

```bash
# 拉取最新代码
git pull

# 重新构建并启动
docker-compose up -d --build
```

### 进入容器

```bash
# 进入后端容器
docker-compose exec backend bash

# 进入数据库容器
docker-compose exec db bash

# 进入前端容器
docker-compose exec frontend sh
```

### 数据库备份

```bash
# 备份数据库
docker-compose exec db mysqldump -u root -proot123456 testhub_db > backup.sql

# 恢复数据库
docker-compose exec -T db mysql -u root -proot123456 testhub_db < backup.sql
```

## 🔒 安全配置

### 修改默认密码

生产环境部署前，务必修改 `docker-compose.yml` 中的默认密码：

```yaml
environment:
  - MYSQL_ROOT_PASSWORD=your_secure_root_password
  - MYSQL_PASSWORD=your_secure_password
  - DJANGO_SECRET_KEY=your_long_random_secret_key_at_least_50_chars
```

### 使用环境变量文件（推荐）

创建 `.env` 文件：

```env
MYSQL_ROOT_PASSWORD=your_secure_password
MYSQL_DATABASE=testhub_db
MYSQL_USER=testhub_user
MYSQL_PASSWORD=your_secure_password
DB_PASSWORD=your_secure_password
DB_NAME=testhub_db
DB_USER=testhub_user
REDIS_HOST=redis
REDIS_PORT=6379
DJANGO_SECRET_KEY=your_secure_secret_key_at_least_50_chars
ALLOWED_HOSTS=your_server_ip
```

修改 `docker-compose.yml` 使用 `.env`：

```yaml
services:
  db:
    environment:
      - MYSQL_ROOT_PASSWORD=${MYSQL_ROOT_PASSWORD}
      - MYSQL_DATABASE=${MYSQL_DATABASE}
      - MYSQL_USER=${MYSQL_USER}
      - MYSQL_PASSWORD=${MYSQL_PASSWORD}
```

## 📊 资源配置

### 调整资源限制

编辑 `docker-compose.yml`：

```yaml
services:
  backend:
    deploy:
      resources:
        limits:
          cpus: '2.0'
          memory: 4G
        reservations:
          cpus: '1.0'
          memory: 2G
```

### 数据持久化

项目已配置数据卷：

- `mysql_data`: MySQL 数据库数据
- `media_volume`: 上传的媒体文件
- `logs_volume`: 应用日志

数据存储在 Docker 管理的卷中，容器删除不会丢失数据。

## 🐛 故障排查

### 数据库连接失败

```bash
# 检查数据库容器状态
docker-compose ps db

# 查看数据库日志
docker-compose logs db

# 检查网络连接
docker network inspect testhub-network
```

### 后端启动失败

```bash
# 查看后端日志
docker-compose logs backend

# 检查数据库连接
docker-compose exec backend python manage.py check

# 检查端口占用
netstat -tuln | grep 8000
```

### 前端无法访问

```bash
# 检查前端容器状态
docker-compose ps frontend

# 检查 Nginx 日志
docker-compose logs frontend

# 检查后端连接
docker-compose logs backend | grep Connection
```

## 📦 生产环境部署

### 使用反向代理（Nginx）

```nginx
server {
    listen 80;
    server_name your-domain.com;

    location / {
        proxy_pass http://localhost:80;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    location /api/ {
        proxy_pass http://localhost:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

### 配置 HTTPS

```nginx
server {
    listen 443 ssl;
    server_name your-domain.com;

    ssl_certificate /path/to/cert.pem;
    ssl_certificate_key /path/to/key.pem;

    location / {
        proxy_pass http://localhost:80;
    }
}
```

## 🔧 性能优化

### 启用 Gzip 压缩

编辑 `frontend/nginx.conf`：

```nginx
gzip on;
gzip_types text/plain text/css application/json application/javascript text/xml application/xml;
gzip_min_length 1000;
```

### 配置缓存

```nginx
location ~* \.(css|js|jpg|png|gif|ico)$ {
    expires 30d;
    add_header Cache-Control "public, immutable";
}
```

## 📝 维护建议

1. **定期备份数据库** - 每天备份一次
2. **监控日志文件** - 定期清理旧日志
3. **更新依赖** - 定期更新 Docker 镜像
4. **监控资源使用** - 避免资源耗尽
5. **设置自动重启策略** - 容器崩溃时自动重启

## 📞 支持与帮助

- Docker 官方文档: https://docs.docker.com
- Docker Compose 文档: https://docs.docker.com/compose
- Django 部署指南: https://docs.djangoproject.com/en/stable/howto/deployment/
