---
layout: title
title: acme免费生成https证书
date: 2021-06-08 11:00:23
tags: [运维]
categories: 技术
---

## 生成泛域名证书/HTTPS证书
### 安装
```bash
curl https://get.acme.sh | sh
```

### 生成一个TXT记录，用来检验合法性（使用DNS验证域名是否为你本人所属）。

```bash
acme.sh --issue -d "*.btclass.net" --dns \
--yes-I-know-dns-manual-mode-enough-go-ahead-please
```

### Let's Encrpt 下发如下TXT记录，接着去域名解析控制台新增TXT记录
Add the following TXT record:
Domain: '_acme-challenge.btclass.net'
TXT value: 'pcTXMaAVL1fJhOye77CE-PI7HWNUAJL5hbxK_KvDm5M'

### 最终生成域名证书
```bash
acme.sh --renew -d "*.btclass.net" \
--yes-I-know-dns-manual-mode-enough-go-ahead-please
```

### 输出结果文件，证书使用 fullchain.cer 及密钥 *.btclass.net.key 即可

Your cert is in /Users/heibai/.acme.sh/_.btclass.net/_.btclass.net.cer

Your cert key is in /Users/heibai/.acme.sh/_.btclass.net/_.btclass.net.key

The intermediate CA cert is in /Users/heibai/.acme.sh/*.btclass.net/ca.cer

And the full chain certs is there: /Users/heibai/.acme.sh/*.btclass.net/fullchain.cer

> Let's Encrypt 泛域名证书只支持3个月，需要经常更换。可以使用云厂商的API更新域名DNS解析，acme.sh 配置一下可支持定时自动更新。

> Let's Encrypt 泛域名证书申请及配置：[https://learnku.com/articles/13496/lets-encrypt-pan-domain-name-application-and-configuration](https://learnku.com/articles/13496/lets-encrypt-pan-domain-name-application-and-configuration)


