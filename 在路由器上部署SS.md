# 在路由器上部署SS 

## 前言

一直想尝试通过路由器实现透明代理，最近终于入手 [ASUS RT-AC68U](https://www.asus.com.cn/Networking/RTAC68U/) 。经过不懈摸索终于成功，在此记录安装过程备忘。

## 准备工作

1. 刷入 [Asuswrt-Merlin](https://asuswrt.lostrealm.ca/) 
2. 准备 ext2/ext3 格式的 U 盘
3. 开启 SSH
4. 登录路由器，执行 `entware-setup.sh`

## 安装过程

### 安装必要软件

```shell
# 更新软件列表
opkg update
# 安装
opkg install shadowsocks-libev-ss-local chinadns haveged
```

### 配置 shadowsocks

1. 修改 `/opt/etc/ss-local.json` 和  `/opt/etc/ss-redir.json`  如下：

   ```json
   {
       "server": "SERVER-HOST",
       "server_port": SERVER-PORT,
       "local_address": "0.0.0.0",
       "local_port": PORT,
       "password": "PASSWORD",
       "method": "METHOD"
   }
   ```
   
   **NOTE:** 注意将 `SERVER-HOST`, `SERVER-PORT`, `PASSWORD` 和 `METHOD` 替换成实际信息。`ss-local` 的 `PORT` 设置为  `1080`, `ss-redir` 的 `PORT` 设置为  `1081`。
2. 修改 `/opt/etc/init.d/S21shadowsocks`

   ```shell
   #!/bin/sh
   
   ENABLED=yes
   PROCS=ss-local
   ARGS="-c /opt/etc/ss-.json -b 0.0.0.0"
   PREARGS=""
   DESC=$PROCS
   PATH=/opt/sbin:/opt/bin:/opt/usr/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
   
   [ -z "$(which $PROCS)" ] && exit 0
   
   . /opt/etc/init.d/rc.func
   ```
   `ss-local` 启动本地 SOCKS5 代理。

4. 修改 `/opt/etc/init.d/S23shadowsocks`

   ```shell
   #!/bin/sh
   
   ENABLED=yes
   PROCS=ss-redir
   ARGS="-c /opt/etc/ss-redir.json -b 0.0.0.0"
   PREARGS=""
   DESC=$PROCS
   PATH=/opt/sbin:/opt/bin:/opt/usr/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
   
   [ -z "$(which $PROCS)" ] && exit 0
   
   . /opt/etc/init.d/rc.func
   ```
   
4. 启动服务

   ```shell
   /opt/etc/init.d/S21shadowsocks start
   /opt/etc/init.d/S23shadowsocks start
   ```

### 配置 ChinaDNS

1. 修改 `/opt/etc/init.d/S22dns2socks`

   ```shell
   #!/bin/sh
   
   ENABLED=yes
   PROCS=dns2socks
   ARGS="127.0.0.1:1080 8.8.8.8:53 0.0.0.0:5300 -q"
   PREARGS=""
   DESC=$PROCS
   PATH=/opt/sbin:/opt/bin:/opt/usr/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
   
   . /opt/etc/init.d/rc.func
   ```

   把 `5300`端口到 DNS 请求代理到 `1080` 端口的 SOCKS5 代理。

2. 修改 `/opt/etc/init.d/S30chinadns`

   ```shell
   #!/bin/sh
   
   ENABLED=yes
   PROCS=chinadns
   ARGS="-l /opt/etc/chinadns_iplist.txt -c /opt/etc/chinadns_chnroute.txt -p 5335 -b 0.0.0.0 -s 114.114.114.114,127.0.0.1:5300"
   PREARGS=""
   DESC=$PROCS
   PATH=/opt/sbin:/opt/bin:/opt/usr/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
   
   . /opt/etc/init.d/rc.func
   ```

   `5353` 端口被路由器占用，因此使用 `5301` 作为 `chinadns` 的端口。

3. 启动服务

   ```shell
   /opt/etc/init.d/S22dns2socks start
   /opt/etc/init.d/S30chinadns start
   ```

4. 配置 `/jffs/configs/dnsmasq.conf`

   ```
   domain-needed
   no-resolv
   server=127.0.0.1#5335
   dhcp-range=192.168.50.2,192.168.50.254,24h
   dhcp-option=option:router,192.168.50.1
   ```
   `127.0.0.1:5335` 为 ChinaDNS 服务。

   此处需要同时配置 `dhcp` 服务，否则  `dhcp`  服务将被关闭。

5. 重启 `dnsmasq` 服务

   ```shell
   service restart_dnsmasq
   ```


### 配置自动转发流量

1. 新建 `/mnt/sda1/s.sh`

   ```shell
   #!/usr/bin/env sh
   
   IP_FILE=/opt/etc/chinadns_chnroute.txt
   
   create_all_route(){
       ipset -N china_routes hash:net maxelem 99999
       ( while read ip; do ipset add china_routes "$ip"; done ) < "$IP_FILE"
   
   	# Create new chain
       iptables -t nat -N SHADOWSOCKS
       iptables -t mangle -N SHADOWSOCKS
   
       # Ignore your shadowsocks server's addresses
       # It's very IMPORTANT, just be careful.
       iptables -t nat -A SHADOWSOCKS -d SERVER-HOST -j RETURN
   
       # Ignore LANs and any other addresses you'd like to bypass the proxy
       # See Wikipedia and RFC5735 for full list of reserved networks.
       # See ashi009/bestroutetb for a highly optimized CHN route list.
       iptables -t nat -A SHADOWSOCKS -d 0.0.0.0/8 -j RETURN
       iptables -t nat -A SHADOWSOCKS -d 10.0.0.0/8 -j RETURN
       iptables -t nat -A SHADOWSOCKS -d 127.0.0.0/8 -j RETURN
       iptables -t nat -A SHADOWSOCKS -d 169.254.0.0/16 -j RETURN
       iptables -t nat -A SHADOWSOCKS -d 172.16.0.0/12 -j RETURN
       iptables -t nat -A SHADOWSOCKS -d 192.168.0.0/16 -j RETURN
       iptables -t nat -A SHADOWSOCKS -d 224.0.0.0/4 -j RETURN
       iptables -t nat -A SHADOWSOCKS -d 240.0.0.0/4 -j RETURN
   
       iptables -t nat -A SHADOWSOCKS -p tcp -m set --match-set china_routes dst -j RETURN
       
       # Anything else should be redirected to shadowsocks's local port
       iptables -t nat -A SHADOWSOCKS -p tcp -j REDIRECT --to-port 1081
       
       iptables -t nat -A OUTPUT -p tcp -j SHADOWSOCKS
       
       # Apply the rules
       iptables -t nat -A PREROUTING -p tcp -j SHADOWSOCKS
       iptables -t mangle -A PREROUTING -j SHADOWSOCKS
   }
   
   clean_all_route(){
       iptables -t nat -D OUTPUT -p tcp -j SHADOWSOCKS
       iptables -t nat -F SHADOWSOCKS
       iptables -t nat -X SHADOWSOCKS
       iptables -t mangle -F SHADOWSOCKS
       iptables -t mangle -X SHADOWSOCKS
       ipset destroy china_routes
   }
   
   "$1"_all_route
   ```

2. 新建 `/opt/etc/init.d/S24create_route`

   ```shell
   #!/bin/sh
   
   /tmp/mnt/sda1/s.sh create
   ```

3. 执行 `/opt/etc/init.d/S24create_route`

## 结语

至此全部配置完成。 `/opt/etc/init.d/` 目录下的服务文件会在开机时被 `/usr/bin/opt-start.sh` 脚本启动，因此 `ss-local`, `dns2socks`, `chinadns` 和配置 `iptable` 的脚本都会被自动启动。

### 参考链接

1. https://zzz.buzz/zh/gfw/2016/02/16/deploy-shadowsocks-on-routers/
2. https://blog.bluerain.io/p/SS-Redir-For-Router.html
3. https://github.com/RMerl/asuswrt-merlin/wiki/Custom-domains-with-dnsmasq
4. https://v2mm.tech/topic/183/%E4%B8%BA%E4%BB%80%E4%B9%88%E8%A6%81%E7%94%A8-chinadns-%E6%9F%A5%E8%AF%A2-dns-%E5%9F%9F%E5%90%8D%E5%8A%AB%E6%8C%81-%E7%BC%93%E5%AD%98%E6%B1%A1%E6%9F%93%E5%8F%8A%E5%BA%94%E5%AF%B9%E5%8A%9E%E6%B3%95
5. https://blog.terrychan.me/2017/thoughts-on-shadowsocks-dns
6. https://blogging.dragon.org.uk/howto-setup-dnsmasq-as-dns-dhcp/