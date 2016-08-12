---
title: 在 OpenWrt 上解决 DNS 污染
date: 2016-07-22 16:04:01
tags:
  - OpenWrt
  - Pcap_DNSProxy
  - Dnsmasq
---

因为某些原因需要解决 DNS 污染问题，本来打算用 Pdnsd + Dnsmasq 组合的，
结果发现 TCP 请求效率太低加上家里网络与那些国外的 DNS 丢包实在是严重，
所以打算用 Pcap_DNSProxy 代替 Pdnsd 。

## Pcap_DNSProxy 的安装与配置

> [Pcap_DNSProxy](https://github.com/chengr28/Pcap_DNSProxy) 
是一个基于 WinPcap/LibPcap 用于过滤 DNS 投毒污染的工具，
提供支持正则表达式的 Hosts 提供更便捷和强大的修改 Hosts 的方法，
以及对 DNSCurve/DNSCrypt 协议、并行和 TCP 协议请求的支持。
多服务器并行请求功能，更可提高在恶劣网络环境下域名解析的可靠性。

### 下载安装
[OpenWrt 编译](https://github.com/wongsyrone/openwrt-Pcap_DNSProxy/tree/prebuilt-ipks)

下载完成后将 ipk 文件传入 OpenWrt 中然后使用 opkg 安装：

```
opkg install pcap-dnsproxy_0.4.6.1-1_ar71xx.ipk
```

如果安装失败请根据提示安装相应依赖：

```
opkg update
opkg install libpcap libsodium libstdcpp
```

对于配置，可以直接使用默认的~~其实就是懒~~，
如果想调整，可以去看[项目文档](https://github.com/chengr28/Pcap_DNSProxy/blob/master/Documents/ReadMe(zh_Hans).txt)，
写的很详细。不过请确保监听端口为非 53 端口。

### 启动程序

修改 `/etc/config/pcap-dnsproxy` 文件的 enabled 值为 1 后

```
/etc/init.d/pcap-dnsproxy enable
/etc/init.d/pcap-dnsproxy start
```

## Dnsmasq 配置

Pcap-DNSProxy 本身就是一个完整的解决 DNS 污染的工具，
不过它的 OpenWrt 版不建议用于对国内域名解析，
所以需要使用 Dnsmasq 来解析国内 IP 。
Dnsmasq 属于 OpenWrt 自带软件，
所以我们只需要修改它的配置文件，让它将国内域名解析指向国内的 DNS ，
国外的其余的指向 Pcap_DNSProxy 的监听端口就好了。

根据自己的需求和实际修改 Dnsmasq 的配置文件 
`/etc/dnsmasq.conf` 如下：

```
no-resolv                 /* 此处防止获取到ISP DNS从而干扰解析 */
no-poll                   /* 此处取消对 resolv.conf 的轮询，用于配合 no-resolv */
domain-needed             /* 此处限制非域名的DNS转发请求 */
no-negcache               /* 此处取消对不存在域名的缓存 */
server=192.168.1.1#1053   /* 此处为网关IP地址，尽量不要使用 127.0.0.1；后面是监听端口 */
all-servers               /* 如果配置了多个上游DNS并且确保均不受污染，可开启此项加速解析 */
cache-size=10000          /* 此处加大 dnsmasq 的内置缓存条数，默认值为 150,一般最大值为 10000 */
conf-dir=/etc/dnsmasq.d   /* 此处指向分流配置文件所在文件夹 */
```

因为 Dnsmasq 可以根据你的配置重定向指定域名到指定 DNS 服务器，
所以我们可以使用 [dnsmasq-china-list](https://github.com/felixonmars/dnsmasq-china-list)
项目来将国内域名指向国内的 DNS 服务器。

根据上面的配置，建立 `/etc/dnsmasq.d/` 文件夹存放配置

```
wget --no-check-certificate https://raw.githubusercontent.com/felixonmars/dnsmasq-china-list/master/accelerated-domains.china.conf
```

> PS: 请重新安装 wget 获取 https 支持。

然后重启 Dnsmasq 就 OK 了。

## 维护

Dnsmasq 的分流列表是需要更新的，这时候我们就需要 Cron 来帮助自动更新。

先在 `/usr/bin` 下新建 `updatelist.sh` 文件

```
rm -rf /etc/dnsmasq.d/accelerated-domains.china.conf
wget --no-check-certificate https://raw.githubusercontent.com/felixonmars/dnsmasq-china-list/master/accelerated-domains.china.conf -P /etc/dnsmasq.d/
```

使用 `chmod +x updatelist.sh` 将其改成可执行。

然后进入 OpenWrt 的 Luci 页面，打开 系统 -> 计划任务 输入：

```
0 1 * * 3 updatelist.sh                     /* 每周三 1:00 更新 Dnsmasq 分流列表 */
0 3 * * 3 /etc/init.d/dnsmasq restart       /* 每周三 3:00 重启 dnsmasq */
0 4 * * 3 /etc/init.d/pcap_dnsproxy restart /* 每周三 4:00 重启 pcap-dnsproxy */
```

## 端口监听

有的时候 Pcap 会崩溃导致 DNS 解析失败，所以需要有一个  Cron 来对它进行监测。

按更新 Dnsmasq 的方法建立 `pcap.sh` 文件

```
#!/bin/sh

netstat -nl | grep 1053

if [ $? -ne 0 ]
then
  /etc/init.d/pcap-dnsproxy restart
fi
```

并在计划任务添加：

```
*/1 * * * * pcap.sh
```

这个脚本会每隔一分钟检查 Pcap 的监听端口是否存在，如果不存在就重启 Pcap 进程。

### Over

## 参考资料

- [OpenWRT 路由器 unbound+dnsmasq 解决 DNS 污染与劫持](https://cokebar.info/archives/246)
- [openwrt 上通过 pdnsd 和 dnsmasq 解决 dns 污染](https://wido.me/sunteya/use-openwrt-resolve-gfw-dns-spoofing)