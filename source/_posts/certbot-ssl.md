---
title: 免费SSL证书制作-Lets Encrypt
tags:
  - SSL
categories:
  - SSL
date: '2018-09-29 18:04'
abbrlink: 28158
---

腾讯云一年的免费SSL证书到期了，在老大的推荐下看了一下`Let's Encrypt` 家的免费证书，虽然只有三个月，但是支持免费续期，而且支持通配符证书，这倒是很大的福利了

<!--more-->

首先声明，我的系统是`CentOS7` ，系统不一致，可自行Google其他或直接官网教程[https://certbot.eff.org/lets-encrypt/centosrhel7-nginx](https://certbot.eff.org/lets-encrypt/centosrhel7-nginx)

# 下载certbot

~~~
$ wget https://dl.eff.org/certbot-auto
$ chmod +x certbot-auto
~~~

# 制作通配符证书

~~~
$ ./certbot-auto certonly  -d "*.example.com" -d "example.com"  --manual --preferred-challenges dns-01  --server https://acme-v02.api.letsencrypt.org/directory
~~~

这里会自动运行yum安装必要的包，然后会提示如下

~~~
Please choose an account
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
1: dev_test@2016-04-18T23:21:19Z (911e)
2: ip-10-164-131-233.ap-southeast-1.compute.internal@2016-06-13T11:02:16Z (5c1b)
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Select the appropriate number [1-2] then [enter] (press 'c' to cancel): 1
Obtaining a new certificate
Performing the following challenges:
dns-01 challenge for example.com
dns-01 challenge for example.com

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
NOTE: The IP of this machine will be publicly logged as having requested this
certificate. If you're running certbot in manual mode on a machine that is not
your server, please ensure you're okay with that.

Are you OK with your IP being logged?
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
(Y)es/(N)o: Y
~~~

注意这里

~~~
Please deploy a DNS TXT record under the name
_acme-challenge.example.com with the following value:

e35fqmCZcB8L56ID4801hlA3aLx3viXtZo1yA3WVSmg

Before continuing, verify the record is deployed.
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Press Enter to Continue
~~~

需要在后台域名解析之后再按Enter

制作完成后，将制作后的证书拿过用即可

配置如下

~~~
     ssl_certificate cert/fullchain1.pem;
     ssl_certificate_key cert/privkey1.pem;
     ssl_trusted_certificate cert/chain1.pem;
~~~

重启后就能看到HTTPS的安全绿色小球了

# 续期

上面提到了，这个证书有效期只有三个月，所以需要定时重新生成新的证书，然后完成后重启nginx

完整命令：`certbot-auto renew [--cert-name CERTNAME] [options]` 
可选参数：

- `--cert-name CERTNAME`：指定要更新的证书。Certbot 用这个名称来管理证书文件。名称不影响证书内容。可以用 `certbot certificates` 命令查看证书名。
- `--dry-run`：用于测试，只获取测试证书，不保存至磁盘。只有 certonly 和 renew 两个子命令可以用这个参数。仍然会会改写 Apache 或 Nginx 服务器的配置文件并重启服务器。仍然会调用 `--pre-hook` 和 `--post-hook` 命令（只要定义过）。只是不再调用 `--deploy-hook` 命令。
- `--force-renewal, --renew-by-default`：强制更新域名的证书，即使离过期时间还远得很。
- `--allow-subset-of-names`：在域名所有权认证时，即使认证失败，也产生证书。在更新多个域名时有效，因为有可能部分域名不再指向当前主机。注意：不能和参数 `--csr` 同时使用。
- `-q, --quiet`：静默执行。
- `--debug-challenges`：调试模式，提交至 CA 前需要用户确认。
- `--preferred-challenges`：验证域名所有权的方式，”dns” 或 “tls-sni-01,http,dns” 等。每个服务器插件支持有限种类的方式。
- `--pre-hook PRE_HOOK`：在获取证书前要执行的 shell 命令。比如暂时关闭服务器软件以防止可能的冲突。只有在自动获取/更新证书时才会执行。如果更新多个证书时，只执行第一个命令。
- `--post-hook POST_HOOK`：在获取证书后要执行的 shell 命令。比如**部署新证书，或重启服务器软件。**如果更新多个证书时，只执行第一个命令。
- `--deploy-hook DEPLOY_HOOK`：每个有效的认证都会触发一次的 shell 命令。对这个命令，shell 变量 `$RENEWED_LINEAGE` 表示包含域名证书和私钥的配置目录，比如 `/etc/letsencrypt/live/example.com`。`$RENEWED_DOMAINS` 表示空格分隔的刚更新的域名列表，比如 `example.com www.example.com`。
- `--disable-hook-validation`：验证 `--pre-hook/--post-hook/--deploy-hook` 的 shell 命令是否有效，提前发现错误。
- `--no-directory-hooks`：更新证书时不执行 hook 目录中的命令。

写个脚本加入到定时任务里面即可

