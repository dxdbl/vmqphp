# Docker 部署说明

## 一、通过 GitHub 构建镜像

将代码推送到 `master` 分支、推送 `v*`/`V*` 版本标签，或者在 GitHub Actions 页面手动运行 **Build Docker image**。构建完成后会发布以下镜像标签：

- 默认分支：`ghcr.io/dxdbl/vmqphp:latest`
- 版本发布：使用对应的 Git 标签
- 每次构建：生成一个 `sha-*` 标签

公开镜像可以直接拉取。私有镜像需要先在 VPS 登录 GHCR，Token 必须具有 `read:packages` 权限：

```bash
echo "$GHCR_TOKEN" | docker login ghcr.io -u GITHUB_USERNAME --password-stdin
```

## 二、准备宿主机 MySQL

首次部署时，将 `vmq.sql` 导入 VPS 上的 MySQL。请创建独立的 `vmq` 用户，不要使用 `root`。

MySQL 保持监听 `127.0.0.1:3306` 即可。容器使用 host 网络后会与宿主机共享网络，因此容器中的应用可以直接连接该地址。不要在公网防火墙中开放 `3306` 端口。

## 三、启动应用容器

将命令中的数据库名、用户名和密码替换为真实配置：

```bash
docker pull ghcr.io/dxdbl/vmqphp:latest

docker run -d \
  --name vmqphp \
  --restart unless-stopped \
  --network host \
  -e 'DB_DSN=mysql:host=127.0.0.1;port=3306;dbname=vmq;charset=utf8' \
  -e 'DB_USERNAME=vmq' \
  -e 'DB_PASSWORD=请替换为真实密码' \
  -e 'APP_DEBUG=false' \
  -e 'SUPPORT_EMAIL=pay-support@example.com' \
  ghcr.io/dxdbl/vmqphp:latest
```

`SUPPORT_EMAIL` 用于配置支付页面显示的订单异常联系邮箱。未设置或邮箱格式无效时，程序会使用内置默认邮箱。

数据库密码会保留在 Shell 历史记录中。部署后请妥善限制 VPS 登录权限，不要将真实启动命令提交到 GitHub。

`--network host` 仅适用于 Linux VPS。镜像中的 Apache 已固定监听 `127.0.0.1:18080`，因此不会占用 Nginx 的 80 端口，也不会直接向公网暴露 18080 端口。host 网络模式不使用 `-p` 端口映射。

检查容器和应用状态：

```bash
docker ps
docker logs vmqphp
curl http://127.0.0.1:18080/think
```

健康检查接口正常时会返回 `hello,ThinkPHP5!`。

## 四、配置 Nginx 反向代理

将域名解析到 VPS，然后添加 Nginx 站点配置：

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

Apache 只监听 `127.0.0.1:18080`，公网请求必须经过 Nginx，不能直接访问该端口。
