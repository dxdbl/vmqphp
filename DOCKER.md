# Docker 部署说明

本文档说明如何使用项目提供的 Docker 镜像在 Linux VPS 上部署 V 免签。当前镜像使用 Apache 提供 PHP 应用，数据库使用宿主机 MySQL，外部请求建议通过 Nginx 转发。

## 1. 构建和获取镜像

GitHub Actions 工作流位于 `.github/workflows/docker-image.yml`，触发条件如下：

- 推送到 `master` 分支；
- 推送 `v*` 或 `V*` 版本标签；
- 在 GitHub Actions 页面手动运行 `Build Docker image`。

镜像发布到 GitHub Container Registry：

- 默认分支：`ghcr.io/dxdbl/vmqphp:latest`；
- Git 标签：使用对应的标签，例如 `ghcr.io/dxdbl/vmqphp:v1.0`；
- 每次构建：生成一个 `sha-*` 标签。

拉取公开镜像：

```bash
docker pull ghcr.io/dxdbl/vmqphp:latest
```

如果镜像为私有镜像，先使用具有 `read:packages` 权限的 Token 登录：

```bash
echo "$GHCR_TOKEN" | docker login ghcr.io -u GITHUB_USERNAME --password-stdin
```

也可以在项目根目录本地构建：

```bash
docker build -t vmqphp:local .
```

## 2. 准备 MySQL

数据库不包含在应用镜像中。首次部署前，在宿主机 MySQL 中创建数据库和独立用户，不要使用 `root` 连接应用：

```bash
mysql -u root -p -e "CREATE DATABASE IF NOT EXISTS vmq CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;"
mysql -u root -p -e "CREATE USER IF NOT EXISTS 'vmq'@'127.0.0.1' IDENTIFIED BY '请替换为强密码'; GRANT ALL PRIVILEGES ON vmq.* TO 'vmq'@'127.0.0.1'; FLUSH PRIVILEGES;"
mysql -u vmq -p vmq < vmq.sql
```

确认 MySQL 在 `127.0.0.1:3306` 监听。使用本文档中的 `--network host` 时，容器可以通过这个地址访问宿主机 MySQL；不要为了应用连接而将 `3306` 暴露到公网。

## 3. 启动应用

当前镜像内的 Apache 监听 `127.0.0.1:${APP_PORT}`，默认端口为 `18080`。Linux VPS 使用 host 网络时，容器与宿主机共享网络命名空间，因此不需要使用 `-p` 映射端口。

建议将环境变量保存到仅管理员可读的文件中：

```bash
sudo install -d -m 700 /etc/vmqphp
sudo nano /etc/vmqphp/vmqphp.env
sudo chmod 600 /etc/vmqphp/vmqphp.env
```

文件内容示例：

```dotenv
APP_PORT=18080
APP_DEBUG=false
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=vmq
DB_USERNAME=vmq
DB_PASSWORD=请替换为真实数据库密码
SUPPORT_EMAIL=pay-support@example.com
```

启动容器：

```bash
docker run -d \
  --name vmqphp \
  --restart unless-stopped \
  --network host \
  --env-file /etc/vmqphp/vmqphp.env \
  ghcr.io/dxdbl/vmqphp:latest
```

数据库也可以通过 `DB_DSN` 配置。如果设置了 `DB_DSN`，它会优先于 `DB_HOST`、`DB_PORT` 和 `DB_DATABASE` 等分项配置。应用支持的主要环境变量如下：

| 变量 | 默认值 | 说明 |
| --- | --- | --- |
| `APP_PORT` | `18080` | Apache 监听端口 |
| `APP_DEBUG` | `false` | 生产环境应保持关闭 |
| `DB_HOST` | `127.0.0.1` | MySQL 地址 |
| `DB_PORT` | `3306` | MySQL 端口 |
| `DB_DATABASE` | `vmq` | 数据库名 |
| `DB_USERNAME` | `root` | 数据库用户，生产环境请显式设置 |
| `DB_PASSWORD` | 空 | 数据库密码，生产环境请显式设置 |
| `DB_DSN` | 空 | 可选的完整 PDO DSN |
| `SUPPORT_EMAIL` | 内置邮箱 | 支付异常联系邮箱 |

`--network host` 仅适用于 Linux。Docker Desktop 的 host 网络行为不同，不能直接套用上面的启动命令；如需在本地开发，建议使用 PHP 内置服务器或自行调整 Apache 监听地址和端口映射。

## 4. 检查运行状态

镜像已配置 Docker 健康检查，请执行：

```bash
docker ps
docker inspect --format '{{json .State.Health}}' vmqphp
docker logs --tail 100 vmqphp
curl http://127.0.0.1:18080/think
```

健康检查接口正常时返回：

```text
hello,ThinkPHP5!
```

如果修改了 `APP_PORT`，上述 `curl` 地址以及 Nginx 的 `proxy_pass` 端口也必须同步修改。

## 5. 配置 Nginx 反向代理

应用端口只监听宿主机回环地址，公网请求应通过 Nginx 接收。将域名解析到 VPS 后，添加类似配置：

```nginx
server {
    listen 80;
    listen [::]:80;
    server_name pay.example.com;

    location / {
        proxy_pass http://127.0.0.1:18080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        client_max_body_size 10m;
        proxy_connect_timeout 10s;
        proxy_read_timeout 60s;
    }
}
```

检查并重新加载 Nginx：

```bash
sudo nginx -t
sudo systemctl reload nginx
```

生产环境应继续配置 HTTPS，并仅在防火墙开放 `80` 和 `443`。不要直接开放应用端口或 MySQL 的 `3306` 端口。

## 6. 升级和回滚

升级到最新镜像：

```bash
docker pull ghcr.io/dxdbl/vmqphp:latest
docker rm -f vmqphp
docker run -d \
  --name vmqphp \
  --restart unless-stopped \
  --network host \
  --env-file /etc/vmqphp/vmqphp.env \
  ghcr.io/dxdbl/vmqphp:latest
```

生产环境建议使用不可变的版本标签或 `sha-*` 标签，而不是直接依赖 `latest`。升级前记录当前镜像：

```bash
docker inspect --format '{{.Config.Image}}' vmqphp
```

回滚时将启动命令中的镜像替换为之前使用的版本标签或 SHA 标签。数据库迁移不会由容器自动执行，升级前请确认代码与 `vmq.sql` 的兼容性并自行备份数据库。

## 7. 常见问题

### 容器不断重启

先查看日志和健康检查状态：

```bash
docker logs vmqphp
docker inspect --format '{{json .State.Health}}' vmqphp
```

重点检查 `APP_PORT` 是否被占用、MySQL 用户密码是否正确，以及数据库是否已导入 `vmq.sql`。

### 数据库连接失败

确认容器使用了 `--network host`，并检查 `DB_HOST`、`DB_PORT`、`DB_DATABASE`、`DB_USERNAME` 和 `DB_PASSWORD`。如果 MySQL 用户限制为 `localhost`，请为 `127.0.0.1` 创建对应授权，或按实际连接地址调整用户授权。

### Nginx 返回 502

确认容器正在运行，且 Nginx 的 `proxy_pass` 端口与 `APP_PORT` 一致：

```bash
ss -lntp | grep 18080
curl http://127.0.0.1:18080/think
```

### 登录后出现数据库或运行时错误

确认 `runtime` 目录可由容器内的 `www-data` 用户写入，并确保生产环境没有开启 `APP_DEBUG`。不要将 `.env`、数据库密码、支付密钥或其他敏感配置提交到 Git。
