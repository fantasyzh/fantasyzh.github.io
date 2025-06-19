---
layout: post
title:  "TLS & HTTPS"
date:   2025-06-18 18:24:00 +0800
---

## 测试用无CA自签证书

一行命令生成私钥和证书：

```
openssl req -x509 -newkey rsa:4096 -keyout key.pem -out cert.pem -days 365 -nodes -subj /CN=localhost
```
