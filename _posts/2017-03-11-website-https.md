---
layout: post
title:  "使用HTTPS给你的网站加密"
date:   2017-03-11
desc: "使用HTTPS给你的网站加密"
keywords: "https,Let's Encrypt,encrypt,website,security"
categories: [Linux]
tags: [Linux,Website,HTTPS]
icon: icon-nginx
---

## 有哪些靠谱的免费 HTTPS 证书提供商？

选择证书提供商有3个主要考量：1. 浏览器和操作系统支持程度 2. 证书类型 3. 维护成本

1. 浏览器和操作系统支持程度
	基本你能查到的热门证书提供商，支持程度都不会太差。例如 Let's Encrypt 的支持可以访问：Which browsers and operating systems support Let’s Encrypt

	可以看到，Android 2.3.6 以上，Firefox 2.0 以上，Windows Vista 以上，iOS 3.1 以上，Google Chrome全平台都是支持的。这一点就不用太担心了，看你你的网站受众情况来决定。对于我来说，我完全不在乎 Windows XP 的 IE 用户。

2. 证书类型
	HTTPS 证书分为3类， 1. DV 域名验证证书 2. OV 组织机构验证证书 3. EV 增强的组织机构验证证书。每类证书在审核和验证方面要求严格程度不同，浏览器会在地址栏给予不同证书不一样的展现。

	一般个人使用DV证书完全够了，浏览器表现为地址栏前会有绿色的小锁。下面聊到的免费证书都是 DV 域名验证证书。

3. 维护成本
	我调研不多，使用过 StartSSL，现在用 Let's Encrypt。StartSSL 的免费证书有效期是1年，1年后需要手动更换。配置过程还挺麻烦的。

更推荐 Let's Encrypt，虽然有效期只有3个月，但可以用 certbot 自动续期，完全不受影响。而且 Let's Encrypt 因为有了 certbot 这样的自动化工具，配置管理起来非常容易。

## 生成 Let's Encrypt 证书(ubuntu)

1. 安装letsencrypt
	`apt-get install letsencrypt`

2. 生成证书(以jarrekk.com为例)
	`letsencrypt -c /etc/letsencrypt/configs/jarrekk.com.conf certonly`
	此时在/etc/letsencrypt/live/vps.jarrekk.com下有如下文件：
	`cert.pem  chain.pem  fullchain.pem  privkey.pem  root.pem  root_ca_cert_plus_intermediates`


## 为Nginx配置HTTPS

为http服务配置HTTPS可以使用标准配置文件模板，请访问：[Mozilla SSL Configuration Generator](https://mozilla.github.io/server-side-tls/ssl-config-generator/).这是 Mozilla 搞得一个 HTTPS 配置文件自动生成器，支持 Apache，Nginx 等多种服务器。按照这个配置文件，选择 Intermediate 的兼容性。这里生成的配置文件是业界最佳实践和结果，让 Nginx 打开了各种增加安全性和性能的参数。下面是我的域名（vps.jarrekk.com）配置文件示例：

```
> # cat /etc/nginx/sites-enabled/jarrekk.conf
# nginx server conf
server {
    server_name vps.jarrekk.com;
    listen 80 default_server;
#    listen [::]:80 vps.jarrekk.com;

    # Redirect all HTTP requests to HTTPS with a 301 Moved Permanently response.
    return 301 https://$host$request_uri;
}

server {
    server_name vps.jarrekk.com;
    listen 443 ssl http2;
#    listen [::]:443 ssl http2;

    # certs sent to the client in SERVER HELLO are concatenated in ssl_certificate
    ssl_certificate /etc/letsencrypt/live/vps.jarrekk.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/vps.jarrekk.com/privkey.pem;
    ssl_session_timeout 1d;
    ssl_session_cache shared:SSL:50m;
    ssl_session_tickets off;

    # Diffie-Hellman parameter for DHE ciphersuites, recommended 2048 bits
    ssl_dhparam /etc/nginx/ssl/dhparam.pem;

    # intermediate configuration. tweak to your needs.
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
    ssl_ciphers 'ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA256:ECDHE-ECDSA-AES128-SHA:ECDHE-RSA-AES256-SHA384:ECDHE-RSA-AES128-SHA:ECDHE-ECDSA-AES256-SHA384:ECDHE-ECDSA-AES256-SHA:ECDHE-RSA-AES256-SHA:DHE-RSA-AES128-SHA256:DHE-RSA-AES128-SHA:DHE-RSA-AES256-SHA256:DHE-RSA-AES256-SHA:ECDHE-ECDSA-DES-CBC3-SHA:ECDHE-RSA-DES-CBC3-SHA:EDH-RSA-DES-CBC3-SHA:AES128-GCM-SHA256:AES256-GCM-SHA384:AES128-SHA256:AES256-SHA256:AES128-SHA:AES256-SHA:DES-CBC3-SHA:!DSS';
    ssl_prefer_server_ciphers on;

    # HSTS (ngx_http_headers_module is required) (15768000 seconds = 6 months)
    add_header Strict-Transport-Security max-age=15768000;

    # OCSP Stapling ---
    # fetch OCSP records from URL in ssl_certificate and cache them
    ssl_stapling on;
    ssl_stapling_verify on;

    ## verify chain of trust of OCSP response using Root CA and Intermediate certs
    ssl_trusted_certificate /etc/letsencrypt/live/vps.jarrekk.com/root_ca_cert_plus_intermediates;

    resolver 108.61.10.10;

    charset utf-8;
    client_max_body_size 75M;

    location / {
        root /usr/share/nginx/html;
    }
}

server {
    listen 8443 ssl http2;
#    listen [::]:8443 ssl http2;

    # certs sent to the client in SERVER HELLO are concatenated in ssl_certificate
    ssl_certificate /etc/letsencrypt/live/vps.jarrekk.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/vps.jarrekk.com/privkey.pem;
    ssl_session_timeout 1d;
    ssl_session_cache shared:SSL:50m;
    ssl_session_tickets off;

    # Diffie-Hellman parameter for DHE ciphersuites, recommended 2048 bits
    ssl_dhparam /etc/nginx/ssl/dhparam.pem;

    # intermediate configuration. tweak to your needs.
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
    ssl_ciphers 'ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA256:ECDHE-ECDSA-AES128-SHA:ECDHE-RSA-AES256-SHA384:ECDHE-RSA-AES128-SHA:ECDHE-ECDSA-AES256-SHA384:ECDHE-ECDSA-AES256-SHA:ECDHE-RSA-AES256-SHA:DHE-RSA-AES128-SHA256:DHE-RSA-AES128-SHA:DHE-RSA-AES256-SHA256:DHE-RSA-AES256-SHA:ECDHE-ECDSA-DES-CBC3-SHA:ECDHE-RSA-DES-CBC3-SHA:EDH-RSA-DES-CBC3-SHA:AES128-GCM-SHA256:AES256-GCM-SHA384:AES128-SHA256:AES256-SHA256:AES128-SHA:AES256-SHA:DES-CBC3-SHA:!DSS';
    ssl_prefer_server_ciphers on;

    # HSTS (ngx_http_headers_module is required) (15768000 seconds = 6 months)
    add_header Strict-Transport-Security max-age=15768000;

    # OCSP Stapling ---
    # fetch OCSP records from URL in ssl_certificate and cache them
    ssl_stapling on;
    ssl_stapling_verify on;

    ## verify chain of trust of OCSP response using Root CA and Intermediate certs
    ssl_trusted_certificate /etc/letsencrypt/live/vps.jarrekk.com/root_ca_cert_plus_intermediates;

    resolver 108.61.10.10;

    server_name vps.jarrekk.com;

    charset utf-8;
    client_max_body_size 75M;

    location / {
        uwsgi_pass 127.0.0.1:9090;
        include     /etc/nginx/uwsgi_params;
    }

    #Proxy Settings
    proxy_redirect off;
    proxy_set_header Host \$host;
    proxy_set_header X-Real-IP \$remote_addr;
    proxy_set_header X-Forwarded-For \$proxy_add_x_forwarded_for;
    proxy_next_upstream error timeout invalid_header http_500 http_502 http_503 http_504;
    proxy_max_temp_file_size 0;
    proxy_connect_timeout 90;
    proxy_send_timeout 90;
    proxy_read_timeout 90;
    proxy_buffer_size 4k;
    proxy_buffers 4 32k;
    proxy_busy_buffers_size 64k;
    proxy_temp_file_write_size 64k;
#   client_max_body_size 50m;
}
```

## 自动化定期更新证书

Let's Encrypt 证书有效期是3个月，我们可以通过 certbot 来自动化续期。

在 Arch Linux 上，我们通过 systemd 来自动执行证书续期任务。

```
$ sudo vim /etc/systemd/system/letsencrypt.service
[Unit]
Description=Let's Encrypt renewal

[Service]
Type=oneshot
ExecStart=/usr/bin/certbot renew --quiet --agree-tos
ExecStartPost=/bin/systemctl reload nginx.service
```

然后增加一个 systemd timer 来触发这个服务：

```
$ sudo vim /etc/systemd/system/letsencrypt.timer
[Unit]
Description=Monthly renewal of Let's Encrypt's certificates

[Timer]
OnCalendar=daily
Persistent=true

[Install]
WantedBy=timers.target
```

启用服务，开启 timer：

```
$ sudo systemctl enable letsencrypt.timer
$ sudo systemctl start letsencrypt.timer
```
上面两条命令执行完毕后，你可以通过`systemctl list-timers`列出所有 systemd 定时服务。当中可以找到`letsencrypt.timer`并看到运行时间是明天的凌晨12点。

在其他 Linux 发行版本中，可以使用 crontab 来设定定时任务，自行 Google 吧。

## 用专业在线工具测试你的服务器 SSL 安全性

[Qualys SSL Labs](https://www.ssllabs.com/ssltest/index.html)提供了全面的 SSL 安全性测试，填写你的网站域名，给自己的 HTTPS 配置打个分。
如果你完全按照我上面教程配置，遵循了最佳实践，你应该和我一样得分是[A+](https://www.ssllabs.com/ssltest/analyze.html?d=vps.jarrekk.com&latest)
