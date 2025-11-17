---
title: "使用acme.sh申请泛域名证书"
date: 2023-11-07T15:40:24+08:00
draft: flase
categories:
  - 编程
---

## 安装acme.sh

```sh
curl https://get.acme.sh | sh
```

## 申请Cloudflare token

申请域名证书需要验证域名所有权，这里域名部署在Cloudflare，直接使用[Cloudflare Token](https://dash.cloudflare.com/wp-content/uploads/profile/api-tokens)验证

![cloudflare令牌界面](https://objects.wangaiguo.com/wp-content/uploads/2023/11/1699342918815.png)

这里创建令牌，然后选择自己托管的域名就好了

![申请令牌](https://objects.wangaiguo.com/wp-content/uploads/2023/11/1699343315236.png)

保存到本地，因为cloudflare只显示一次

### 配置Acme

```sh
#设置CF变量
export CF_Token=""
export CF_Account_ID=""

#调用Cloudfalre添加DNS Record（会自动在域名所属服务器添加解析记录：_acme-challenge，验证后会自动删除）
#要先添加主域名，再添加泛域名
acme.sh --issue --dns dns_cf -d aiguo.buzz -d *.aiguo.buzz

#安装证书到指定位置，可以自动无感续期、自动更新
acme.sh --install-cert -d *.aiguo.buzz \
--key-file       /etc/nginx/ssl/key.pem  \
--fullchain-file /etc/nginx/ssl/cert.pem \
--reloadcmd     "service nginx force-reload"

```

最后重新配置下Nginx的ssl就好了
