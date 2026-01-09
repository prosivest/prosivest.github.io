---
date : 2026-01-09T09:24:51+08:00
lastmod : 2026-01-09T09:24:51+08:00
title : 'VPS通过Certbot自动申请及更新HTTPS网站SSL证书'
description : "" # 文章描述，与搜索优化相关
author : ["想上天的鱼"]
tags: ["VPS","SSL证书"]
ShowToc: true
draft : false
---

> 参考：[https://developer.aliyun.com/article/1676749](https://developer.aliyun.com/article/1676749)

# 安装 Certbot

不同系统安装方式略有不同，这里给你 CentOS / RHEL 和 Debian / Ubuntu 的常用方法。

**CentOS / RHEL 7/8/9**

```bash
# 安装 EPEL
sudo yum install epel-release -y
# 安装 Certbot 和 Nginx 插件（如果用 Apache 就换成 python3-certbot-apache）
sudo yum install certbot python3-certbot-nginx -y
```

**Debian / Ubuntu**

```bash
sudo apt update
sudo apt install certbot python3-certbot-nginx -y
```

# 申请 SSL 证书

如果你用 Nginx：

```bash
sudo certbot --nginx -d [example.com](http://example.com/) -d [www.example.com](http://www.example.com/)
```

- -d 参数后面写你的域名（可以多个）
- Certbot 会自动修改 Nginx 配置并申请证书
- 申请过程中会让你选择是否重定向 HTTP → HTTPS
注意：这里需要你的nginx的conf文件里面的sercername指定对应的域名，如果你配置的是下划线 ""，会导致该命令运行失败，报下面的错误。
    
    ```bash
    Could not automatically find a matching server block for [zhangfeidezhu.com](http://zhangfeidezhu.com/). Set the server_name directive to use the Nginx installer.
    ```
    

**注意：如果nginx未安装在默认目录，需使用以下命令：**

```bash
sudo certbot --nginx --nginx-server-root /usr/local/nginx/conf -d yhmxx.cn -d www.yhmxx.cn
```

使用`--nginx-server-root`参数指定nginx配置文件地址即可。

# 测试证书是否生效

申请完成后，可以在浏览器访问你的域名，查看是否已经是有效的 Let's Encrypt 证书。

# 自动更新证书

Let's Encrypt 证书有效期是 90 天，所以要自动续签。

Certbot 自带定时任务（通常是 /etc/cron.d/certbot 或 systemd 定时器），也可以手动添加 crontab：

```bash
sudo crontab -e

0 3 1 * * /usr/bin/certbot renew --quiet --deploy-hook "/usr/local/nginx/sbin/nginx -t && /usr/local/nginx/sbin/nginx -s reload"
```

解释：

- 0 3 1 * * 每隔 1个月的1号3点执行一次
- --quiet → 静默模式
- --deploy-hook：只有续签成功并部署了新证书才执行
- 用绝对路径，且 reload 前先 -t 校验配置，避免坏配置导致 reload 失败。

# 检查自动续签是否正常

手动测试续签：

```bash
sudo certbot renew --dry-run
```

如果提示 **Congratulations, all simulated renewals succeeded:** ，说明没问题。

✅ 这样配置后，网站 HTTPS 证书会在到期前自动续签，Nginx 会自动加载新证书，几乎可以做到“一劳永逸”。