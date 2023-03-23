---
description: Common issues and solutions for OPENWRT
---

# OPENWRT常见问题和解决

## PPPOE连接异常断开

* 检测到网络断开直接重启网络服务

最简单粗暴的方法是 检测到网络异常就直接重启网络服务，缺点是可能有1分多种的延迟。把下面的命令添加到crontab中，其中的IP地址可以改成你当地比较稳定的IP地址，比如DNS服务：

{% code overflow="wrap" %}
```bash
/bin/ping 223.5.5.5 -c 4 > /dev/null ; [ $? -eq 0 ] || /etc/init.d/network restart
```
{% endcode %}

