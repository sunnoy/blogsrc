---
title: https抓包
date: 2020-02-14 12:12:28
tags:
- net
---

# 配置ssl抓包

<!--more-->

```bash
openssl ciphers -V | column -t | grep "Kx=RSA" | grep "TLSv1.2"


0x00,0x9D  -  AES256-GCM-SHA384              TLSv1.2  Kx=RSA         Au=RSA    Enc=AESGCM(256)    Mac=AEAD
0x00,0x3D  -  AES256-SHA256                  TLSv1.2  Kx=RSA         Au=RSA    Enc=AES(256)       Mac=SHA256
0x00,0x9C  -  AES128-GCM-SHA256              TLSv1.2  Kx=RSA         Au=RSA    Enc=AESGCM(128)    Mac=AEAD
0x00,0x3C  -  AES128-SHA256                  TLSv1.2  Kx=RSA         Au=RSA    Enc=AES(128)       Mac=SHA256

# nginx配置
ssl_ciphers         AES128-SHA256;
```

# wirshake

导入服务端私钥即可