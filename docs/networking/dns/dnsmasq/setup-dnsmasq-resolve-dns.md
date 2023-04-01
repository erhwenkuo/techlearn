# 如何使用 Dnsmasq 設置本地 DNS 解析器

原文: [如何在Ubuntu 20.04上使用Dnsmasq設置本地DNS解析器](https://0xzx.com/zh-tw/202011112016945244.html)

![](./assets/Install_Dnsmaq_in_Ubuntu_18.04.webp)

Dnsmasq 代表「DNS 偽裝」的縮寫，是一種用於小型網路的簡單，輕便且易於使用的 DNS 轉發器。可以將其配置為 DNS 緩存和 DHCP 伺服器，並支持 IPv4 和 IPv6 協議。 當它收到任何 DNS 查詢時，它將從其緩存中答覆它們，或轉發到其他 DNS 伺服器。

Dnsmasq 由三個子系統組成：

- DNS 子系統：用於緩存不同的記錄類型，包括 A，AAAA，CNAME 和 PTR。
- DHCP 子系統：支持 DHCPv4，DHCPv6，BOOTP 和 PXE
- 路由器廣告子系統：它為 IPv6 主機提供基本的自動配置。 它可以獨立使用，也可以與 DHCPv6 結合使用。

在本教程中，我們將向你展示如何在 Ubuntu 20.04 伺服器上使用 Dnsmasq 設置本地 DNS 伺服器。

## 先決條件

- 運行 Ubuntu 20.04 的伺服器。
- 為伺服器配置了 root 密碼。

## 入門

首先，建議將系統軟體包更新為最新版本。 你可以通過運行以下命令來更新所有軟體包：

```bash
sudo apt-get update -y
```

更新所有軟體包後，你將需要在系統中停用 `systemd-resolved` 的服務。`systemd-resolved` 的服務用於本地應用程序的網路名稱解析。

檢查狀態:

```bash
sudo systemctl status systemd-resolved
```

結果:

```
● systemd-resolved.service - Network Name Resolution
     Loaded: loaded (/lib/systemd/system/systemd-resolved.service; enabled; vendor preset: enabled)
     Active: active (running) since Mon 2023-02-27 12:37:37 CST; 5h 56min ago
       Docs: man:systemd-resolved.service(8)
             man:org.freedesktop.resolve1(5)
             https://www.freedesktop.org/wiki/Software/systemd/writing-network-configuration-managers
             https://www.freedesktop.org/wiki/Software/systemd/writing-resolver-clients
   Main PID: 766 (systemd-resolve)
     Status: "Processing requests..."
      Tasks: 1 (limit: 38095)
     Memory: 9.2M
        CPU: 1.687s
     CGroup: /system.slice/systemd-resolved.service
             └─766 /lib/systemd/systemd-resolved
    ...
    ...
```

你可以通過運行以下命令停用它：

```bash
sudo systemctl disable --now systemd-resolved
```

停用該服務後，你將需要刪除默認的 `resolv.conf` 文件，並使用你的自定義 DNS 伺服器詳細信息創建一個新文件。

```title="resolv.conf"
# This is /run/systemd/resolve/stub-resolv.conf managed by man:systemd-resolved(8).
# Do not edit.
#
# This file might be symlinked as /etc/resolv.conf. If you're looking at
# /etc/resolv.conf and seeing this text, you have followed the symlink.
#
# This is a dynamic resolv.conf file for connecting local clients to the
# internal DNS stub resolver of systemd-resolved. This file lists all
# configured search domains.
#
# Run "resolvectl status" to see details about the uplink DNS servers
# currently in use.
#
# Third party programs should typically not access this file directly, but only
# through the symlink at /etc/resolv.conf. To manage man:resolv.conf(5) in a
# different way, replace this symlink by a static file or a different symlink.
#
# See man:systemd-resolved.service(8) for details about the supported modes of
# operation for /etc/resolv.conf.

nameserver 127.0.0.53
options edns0 trust-ad
search home
```

```bash
$ ls -l /etc/resolv.conf 
lrwxrwxrwx 1 root root 39 十一  2 11:01 /etc/resolv.conf -> ../run/systemd/resolve/stub-resolv.conf
```

你可以使用以下命令刪除默認的 `resolv.conf` 文件：

```bash
sudo rm -rf /etc/resolv.conf
```

接下來，使用以下命令將 Google DNS 伺服器添加到 resolv.conf 文件：

```bash
sudo echo "nameserver 8.8.8.8" > /etc/resolv.conf
```

完成後，你可以繼續下一步。

## 安裝 Dnsmasq

默認情況下，Ubuntu 20.04 默認存儲庫中提供 Dnsmasq。 你可以通過運行以下命令來安裝它：

```bash
sudo apt-get install dnsmasq dnsutils ldnsutils -y
```

安裝完成後，Dnsmasq 服務將自動啟動。 你可以使用以下命令檢查 Dnsmasq 的狀態：

```bash
sudo systemctl status dnsmasq
```

結果:

```
● dnsmasq.service - dnsmasq - A lightweight DHCP and caching DNS server
     Loaded: loaded (/lib/systemd/system/dnsmasq.service; enabled; vendor preset: enabled)
     Active: active (running) since Mon 2023-02-27 19:08:47 CST; 1min 11s ago
    Process: 53276 ExecStartPre=/etc/init.d/dnsmasq checkconfig (code=exited, status=0/SUCCESS)
    Process: 53284 ExecStart=/etc/init.d/dnsmasq systemd-exec (code=exited, status=0/SUCCESS)
    Process: 53293 ExecStartPost=/etc/init.d/dnsmasq systemd-start-resolvconf (code=exited, status=0/SUCCESS)
   Main PID: 53292 (dnsmasq)
      Tasks: 1 (limit: 38095)
     Memory: 608.0K
        CPU: 31ms
     CGroup: /system.slice/dnsmasq.service
             └─53292 /usr/sbin/dnsmasq -x /run/dnsmasq/dnsmasq.pid -u dnsmasq -7 /etc/dnsmasq.d,.dpkg-dist,.dpkg-old,.dpkg-new --loca>

...
...
```

完成後，你可以繼續下一步。

## 配置 Dnsmasq

接下來，你將需要將 Dnsmasq 配置為本地 DNS 伺服器。 你可以通過編輯 Dnsmasq 主配置文件來做到這一點：

```bash
sudo nano /etc/dnsmasq.conf
```

修改成:

```conf title="/etc/dnsmasq.conf"
port=53
domain-needed
bogus-priv
listen-address=127.0.0.1,your-server-ip
expand-hosts
domain=dns-example.com
cache-size=1000
```

完成後，保存並關閉文件。

接下來，你需要在 `resolv.conf` 文件中將伺服器的 IP 地址添加為主域名伺服器。 你可以使用以下命令添加它：

```bash
sudo nano /etc/resolv.conf
```

在「名稱伺服器8.8.8.8」行上方添加以下行：

```title="/etc/resolv.conf"
nameserver your-server-ip
nameserver 8.8.8.8
```

完成後，保存並關閉文件。修改完之後重新啟動 `dnsmasq` 服務:

```bash
sudo systemctl restart dnsmasq
```

檢查結果:

```bash
sudo systemctl status dnsmasq
```

接下來，使用以下命令驗證伺服器是否存在任何配置錯誤：

```bash
dnsmasq --test
```

如果一切正常，你應該獲得以下輸出：

```
dnsmasq: syntax check OK.
```

最後，重新啟動 Dnsmasq 服務以應用更改：

```bash
sudo systemctl restart dnsmasq
```

此時，Dnsmasq 已啟動並在埠 53 上偵聽。你可以使用以下命令進行驗證：

```bash
ss -alnp | grep -i :53
```

你應該獲得以下輸出：

```
udp     UNCONN   0        0                                             0.0.0.0:53                                                0.0.0.0:*                      users:(("dnsmasq",pid=41051,fd=4))                                             
udp     UNCONN   0        0                                                [::]:53                                                   [::]:*                      users:(("dnsmasq",pid=41051,fd=6))                                             
tcp     LISTEN   0        32                                            0.0.0.0:53                                                0.0.0.0:*                      users:(("dnsmasq",pid=41051,fd=5))                                             
tcp     LISTEN   0        32                                               [::]:53                                                   [::]:*                      users:(("dnsmasq",pid=41051,fd=7))                                             
```

## 將 DNS 記錄添加到 Dnsmasq 伺服器

接下來，你將需要編輯 `/etc/hosts` 文件並添加本地 DNS 伺服器條目。

```bash
sudo nano /etc/hosts
```

添加以下行：

```title="/etc/hosts"
...
your-server-ip host1.dns-example.com
...
```

完成後，保存並關閉文件。修改完之後重新啟動 `dnsmasq` 服務:

```bash
sudo systemctl restart dnsmasq
```

## 驗證 Dnsmasq 伺服器 DNS 解析

你可以使用 `dig` 命令檢查 DNS 解析，如下所示：

```bash
dig host1.dns-example.com +short
```

如果一切正常，你應該在以下輸出中看到伺服器ip：


```
your-server-ip
```

你還可以使用以下命令來驗證外部DNS解析：

```bash
dig howtoforge.com +short
```

你應該獲得以下輸出：

```
172.67.68.93
104.26.3.165
104.26.2.165
```

