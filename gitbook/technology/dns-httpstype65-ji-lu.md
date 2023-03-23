---
description: DNS HTTPS (TYPE65) RECORD
---

# DNS HTTPS(TYPE65) 记录

## 一、HTTPS记录介绍

### 1、什么是HTTPS记录

每当用户在浏览器框中键入 URL 而未指定方案（如“https://”或“http://”）时，浏览器无法在没有严格传输安全 (HSTS) 等先验知识的情况下假设缓存或预加载列表条目，请求的网站是否支持 HTTPS。浏览器将首先尝试使用明文 HTTP 获取资源，只有当网站重定向到 HTTPS URL，或者在初始 HTTP 响应中指定 HSTS 策略时，浏览器才会通过安全连接再次获取资源。

这意味着由于浏览器需要通过 TLS 重新建立连接并重新请求资源，因此获取初始资源（例如，网站的索引页面）时产生的延迟加倍。但更糟糕的是，初始请求以明文形式泄露到网络，可能会被恶意的路径攻击者（想想所有那些不安全的公共 WiFi 网络）修改，以将用户重定向到一个完全不同的网站。实际上，这种弱点有时会被上述不安全的公共 WiFi 网络运营商用来在人们的浏览器中偷偷投放广告。

不幸的是，这还不是全部。此问题还会影响HTTP/3，这是 HTTP 协议的最新版本，可提供更高的性能和安全性。HTTP/3 使用Alt-Svc HTTP 标头进行通告，该标头仅在浏览器已经使用不同且性能可能较低的 HTTP 版本联系源之后才返回。浏览器在第一次访问该网站时最终错过了使用更快的 HTTP/3（尽管它确实存储了以后访问的知识）。

关于其详细介绍请参阅：[draft-ietf-dnsop-svcb-https-11](https://www.ietf.org/archive/id/draft-ietf-dnsop-svcb-https-11.txt)

### 2、为什么要使用HTTPS记录

1. 避免浏览器访问服务器HTTP端口时被劫持
2. 使用非标准端口时可以不需要在URL中写端口号。特别是在某些国家的家庭宽带不能开放443端口的情况下，通过这种方式就可以不用在URL添加端口直接访问_（但说不定哪天就被封了）_。
3. 可以限定连接协议，比如HTTP2、HTTP3
4. 可以设定ech 信息对 TLS 握手信息作加密处理，防止第三方嗅探

{% hint style="info" %}
注意：目前并非所有的DNS授权域服务器、DNS递归服务器和浏览器都支持，使用时需要考虑兼容性。此外，建议和A、AAAA记录一起使用，如果你使用的网络环境都支持且没有DNS超时的话，浏览器会优先使用HTTPS记录。
{% endhint %}

### 3、工作原理

1. 客户端同时发起 A/AAAA 和 HTTPS 记录
2. (可选)如果 HTTPS 记录是别名模式，则需要查询指向域名的 A/AAAA 和 HTTPS 记录
3. (可选)客户端需要查询服务模式下目标域名的 A/AAAA 记录
4. (可选)DNS 递归解析服务器可以通过附加字段提供这些记录
5. (可选)ECH 记录的特殊要求：
   * 客户端需要在收到 HTTPS 查询结果之后再发起 TLS 连接，以防服务端开启 ECH
   * 如果 HTTPS 查询超时，客户端不能假设服务端不支持 ECH，以防降级攻击
6. 浏览器发起连接

## 二、开始使用HTTPS记录

### 1、格式

{% code overflow="wrap" %}
```
优先级 目标域名 [服务参数...]
```
{% endcode %}

例如

```
1 . alpn="h2,h3" ipv4hint="8.8.8.8" ipv6hint="240e::1" port=8443 ech="xxxxxxx"
```

```
1 example.net alpn="h3,h2"
2 example.org alpn="h2"
```

* 优先级：0～65535。0只有在别名模式下使用；其他则是服务模式，数字越小优先级越高；
* 目标域名：别名模式下指定另一个域名，类似于CNAME；服务模式则直接写“.”，表示使用当前记录的内容
* 服务参数有以下内容（格式：key=value）
  * alpn：服务器支持的HTTP协议，比如“h2”（HTTP/3）、“h3”（HTTP/3 QUIC）。其中“h3”还可以细分成“h3-29”，分对应的是HTTP/3版本号29的协议，具体看[ieft](https://datatracker.ietf.org/doc/html/draft-ietf-quic-http-29)。
  * ipv4hint：IPv4地址
  * ipv6hint：IPv6地址（如果客户端和服务器同时支持IPv6，系统会优先走IPv6）
  * port：如果是非标准端口，可以指定端口
  * ech：Encrypted ClientHello，防止嗅探

### 2、使用案例

Cloudflare:

```
1 . alpn="h3,h3-29,h2" ipv4hint=104.16.123.96,104.16.124.96 ipv6hint=2606:4700::6810:7b60,2606:4700::6810:7c60
```

Google:

```
1 . alpn="h2,h3"
```

非标准端口下使用：

```
1 . alpn="h2" ipv4hint=2.2.2.2 ipv6hint=2606::7c60 port=8443
```

### 3、测试

#### DNS解析测试

最新版dig支持（具体哪个版本开始支持的没有仔细研究）查询HTTPS记录

安装测试环境

<pre class="language-bash"><code class="lang-bash"><strong># docker run -it ubuntu:devel bash
</strong><strong>root@a2585f7603e3:/# apt update
</strong><strong>root@a2585f7603e3:/# apt install dnsutils
</strong></code></pre>

DNS解析测试方法

```
# dig test.wzh.me https

; <<>> DiG 9.18.12-1ubuntu1-Ubuntu <<>> test.wzh.me https
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 29627
;; flags: qr rd ra ad; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1372
;; QUESTION SECTION:
;test.wzh.me.			IN	HTTPS

;; ANSWER SECTION:
test.wzh.me.		60	IN	HTTPS	1 . alpn="h2" ipv4hint=150.230.41.228

;; Query time: 203 msec
;; SERVER: 169.254.169.254#53(169.254.169.254) (UDP)
;; WHEN: Sat Mar 11 06:00:14 UTC 2023
;; MSG SIZE  rcvd: 70
```

#### 浏览器测试

使用caddy搭建测试环境

{% code title="Caddyfile" %}
```
test.wzh.me
tls xx@xx.xx
file_server browse {}
```
{% endcode %}

使用浏览器访问

* 查看wireshark抓包结果

<figure><img src="../.gitbook/assets/截屏2023-03-11 15.56.58.png" alt=""><figcaption><p>DNS解析过程</p></figcaption></figure>

<figure><img src="../.gitbook/assets/截屏2023-03-11 15.57.43.png" alt=""><figcaption><p>HTTPS记录解析结果</p></figcaption></figure>

<figure><img src="../.gitbook/assets/截屏2023-03-11 15.55.29.png" alt=""><figcaption><p>浏览器访问解析结果中的IP</p></figcaption></figure>
