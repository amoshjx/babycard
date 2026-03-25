# 宝宝黑白卡 — Ubuntu + Nginx 部署指南

## 1. 安装 Nginx

```bash
sudo apt update && sudo apt install nginx -y
sudo systemctl enable nginx
sudo systemctl start nginx
```

## 2. 上传文件

```bash
# 创建网站目录
sudo mkdir -p /var/www/baby-flashcards

# 上传 HTML（从本地 scp 到服务器）
scp baby-flashcards.html user@your-server:/var/www/baby-flashcards/index.html

# 设置权限
sudo chown -R www-data:www-data /var/www/baby-flashcards
sudo chmod -R 755 /var/www/baby-flashcards
```

## 3. Nginx 配置

```bash
sudo nano /etc/nginx/sites-available/baby-flashcards
```

粘贴以下完整配置：

```nginx
server {
    listen 80;
    listen [::]:80;
    server_name your-domain.com;    # ← 替换为你的域名或公网 IP
    root /var/www/baby-flashcards;
    index index.html;

    # ══════════════════════════════════
    # Gzip 压缩 — HTML 75KB → ~18KB
    # ══════════════════════════════════
    gzip on;
    gzip_vary on;
    gzip_proxied any;
    gzip_comp_level 6;
    gzip_min_length 256;
    gzip_types
        text/html
        text/css
        text/plain
        text/javascript
        application/javascript
        application/json
        application/ld+json
        image/svg+xml
        application/xml;

    # ══════════════════════════════════
    # 浏览器缓存策略
    # ══════════════════════════════════

    # HTML 主文件 — 短缓存 + 每次验证（保证更新及时生效）
    location = /index.html {
        add_header Cache-Control "public, no-cache, must-revalidate";
        add_header ETag "";
        expires 1h;
    }

    location / {
        try_files $uri $uri/ /index.html;

        # 默认静态资源 — 长缓存
        add_header Cache-Control "public, max-age=2592000, immutable";
        expires 30d;
    }

    # Google Fonts CSS — 缓存 7 天
    # (字体由 Google CDN 处理，这里控制的是 CSS @import)
    location ~* \.(woff2?|ttf|eot|otf)$ {
        add_header Cache-Control "public, max-age=604800";
        expires 7d;
        add_header Access-Control-Allow-Origin "*";
    }

    # 图片/图标（如果后续添加）
    location ~* \.(ico|png|jpg|jpeg|gif|webp|svg)$ {
        add_header Cache-Control "public, max-age=2592000, immutable";
        expires 30d;
    }

    # ══════════════════════════════════
    # 安全头
    # ══════════════════════════════════
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;

    # ══════════════════════════════════
    # 性能优化
    # ══════════════════════════════════
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;

    # 错误页
    error_page 404 /index.html;

    # 禁止访问隐藏文件
    location ~ /\. {
        deny all;
        access_log off;
        log_not_found off;
    }
}
```

## 4. 启用站点

```bash
# 创建软链接
sudo ln -sf /etc/nginx/sites-available/baby-flashcards /etc/nginx/sites-enabled/

# 删除默认站点（可选）
sudo rm -f /etc/nginx/sites-enabled/default

# 测试配置
sudo nginx -t

# 重新加载
sudo systemctl reload nginx
```

## 5. HTTPS（强烈推荐）

```bash
# 安装 certbot
sudo apt install certbot python3-certbot-nginx -y

# 自动获取证书并配置 HTTPS
sudo certbot --nginx -d your-domain.com

# 自动续期（certbot 默认已配置 cron）
sudo certbot renew --dry-run
```

配置完成后 certbot 会自动修改 Nginx 配置，添加 443 端口和 HTTP→HTTPS 301 跳转。

## 6. 验证部署

```bash
# 检查服务状态
sudo systemctl status nginx

# 测试 Gzip 是否生效
curl -H "Accept-Encoding: gzip" -I http://your-domain.com/

# 应该看到: Content-Encoding: gzip

# 测试缓存头
curl -I http://your-domain.com/
# 应该看到: Cache-Control: public, no-cache, must-revalidate
```

## 7. 更新部署

```bash
# 后续更新只需一条命令
scp baby-flashcards.html user@your-server:/var/www/baby-flashcards/index.html

# 无需 reload nginx（HTML 设置了 no-cache，浏览器会自动获取新版本）
# 如果修改了 nginx 配置则需要:
# sudo nginx -t && sudo systemctl reload nginx
```

## 8. 性能预估

| 指标 | 值 |
|-----|-----|
| HTML 原始大小 | ~75 KB |
| Gzip 压缩后 | ~18 KB |
| 首屏加载（4G） | < 1 秒 |
| 首屏加载（3G） | < 3 秒 |
| 懒加载策略 | 首屏 12 张，滚动时按需加载 |
| 分批渲染 | 每批 8 张，requestIdleCallback |
| 字体加载 | Google CDN，@import async |

## 备选：Docker 一键部署

```bash
# 创建 Dockerfile
cat > Dockerfile << 'EOF'
FROM nginx:alpine
COPY baby-flashcards.html /usr/share/nginx/html/index.html
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
EOF

# 创建 nginx.conf
cat > nginx.conf << 'EOF'
server {
    listen 80;
    root /usr/share/nginx/html;
    index index.html;
    gzip on;
    gzip_types text/html text/css application/javascript image/svg+xml application/json;
    gzip_comp_level 6;
    location = /index.html {
        add_header Cache-Control "public, no-cache, must-revalidate";
    }
}
EOF

# 构建并运行
docker build -t baby-flashcards .
docker run -d -p 80:80 --restart=always --name flashcards baby-flashcards
```
