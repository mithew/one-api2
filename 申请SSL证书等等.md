1. 安装必要工具：
```
apt update
apt install certbot nginx docker.io -y
```

2. 申请SSL证书：
```
certbot certonly --standalone -d api.域名.win
```
执行过程中会要求输入邮箱，同意服务条款等。

3. 创建项目所需目录：
```
mkdir -p /home/ubuntu/data/new-api
```

4. 创建Nginx配置：
```
cat > /etc/nginx/conf.d/new-api.conf << 'EOF'
server {
    listen 80;
    server_name api.域名.win;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl;
    server_name api.域名.win;

    ssl_certificate /etc/letsencrypt/live/api.域名.win/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/api.域名.win/privkey.pem;

    location / {
        proxy_pass http://localhost:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
EOF
```

5. 检查并启动Nginx：
```
nginx -t
systemctl restart nginx
```

6. 运行Docker容器：
```
docker run --name new-api -d --restart always \
  -p 3000:3000 \
  -e TZ=Asia/Shanghai \
  -v /home/ubuntu/data/new-api:/data \
  calciumion/new-api:latest
```

7. 设置证书自动续期：
```
(crontab -l 2>/dev/null; echo "0 3 10,25 * * certbot renew --quiet") | crontab -
```

检查自动续期是否设置成功：
```
crontab -l
```

8. 检查服务状态：
```
docker ps
systemctl status nginx
```

完成后，可以通过以下方式访问：
- https://api.域名.win （会自动从http跳转到https）

如果遇到问题，可以查看日志：
```
docker logs new-api
tail -f /var/log/nginx/error.log
```

注意事项：
1. 确保域名已正确解析到服务器IP
2. 确保防火墙允许80和443端口
3. 如果有安全组/防火墙，需要开放这些端口：
```
ufw allow 80
ufw allow 443
```

这样就完成了全部配置，可以通过 https://api.域名.win 安全访问你的服务了。

====================
====================

其他命令：

1. 停止并删除当前Docker容器：
```
docker stop new-api
docker rm new-api
```
```
docker restart new-api
```

2. 五天自动重启
```
(crontab -l 2>/dev/null; echo "30 5 5,10,15,25 * * /sbin/reboot") | crontab -
```

3. 每周日凌晨 2 点清理 72 小时前的 Docker 资源
```
(crontab -l 2>/dev/null; echo "0 2 * * 0 docker system prune -af --filter \"until=72h\"") | crontab - 
```
