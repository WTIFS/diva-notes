# 安装

环境：CentOS7.6

```bash
yum install nginx

# 默认配置文件目录
/etc/nginx/nginx.conf
```



# 阿里云购买免费版SSL证书

1. 进阿里云搜一下SSL证书，免费申请。等待申请通过后点击下载，然后会有个下载链接，复制链接
2. 登录进服务器后，执行以下命令下载证书到服务器上

```bash
cd /alidata/server/nginx/conf/cert

# 注意在链接前后加引号
wget 'https://cas-documents-service.oss-cn-shanghai.aliyuncs.com/xxxxxx'

# 解压缩上述下载的文件
unzip xxxxxx

# 重启 nginx
nginx -s reload
```





以下为 https 详细配置

```bash
http {
	limit_req_zone $binary_remote_addr zone=ratelimit_ip:10m rate=10r/s; # 限流

    server {
        listen       80;
        rewrite ^(.*)$ https://$host$1; # 转发至https
    }
    
    server {
        listen       443 ssl http2;
        server_name  你的域名;

        ssl_certificate "/alidata/server/nginx/conf/cert/证书.pem";
        ssl_certificate_key "/alidata/server/nginx/conf/cert/证书.key";
        ssl_session_timeout  10m;
        #表示使用的加密套件的类型
        ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:ECDHE:ECDH:AES:HIGH:!NULL:!aNULL:!MD5:!ADH:!RC4;
        #表示使用的TLS协议的类型，您需要自行评估是否配置TLSv1.1协议。
        ssl_protocols TLSv1.1 TLSv1.2 TLSv1.3;

        location / {
            limit_req zone=ratelimit_ip burst=10 nodelay;
            proxy_pass http://localhost:3000;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }
    }
}
```

