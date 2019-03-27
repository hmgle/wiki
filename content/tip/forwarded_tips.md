---
title: "网络流量中转技巧荟萃"
layout: page
date: 2019-03-23 19:00
---

[TOC]

## 通过多台跳板机 SSH 登录

假如需要借助 SSH 跳板机 `A` 才能登录服务器 `B`，即：

```console
local> ssh userA@serverA
serverA> ssh userB@serverB
serverB>
```

那么可以在本机通过 ssh 的正向代理（forwarding）选项，运行下面命令直接登录服务器 B：

```console
local> ssh -A -t userA@serverA ssh -A -t userB@serverB
```

如果有多台跳板机也一样：

```console
local> ssh -A -t server1 ssh -A -t server2 ssh -A server3
```

来源：[https://stackoverflow.com/questions/20079646/is-it-possible-to-chain-from-one-terminal-to-another-via-ssh-in-one-series-of-co](https://stackoverflow.com/questions/20079646/is-it-possible-to-chain-from-one-terminal-to-another-via-ssh-in-one-series-of-co)


## 通过跳板机 SCP 传输文件

如果需要经过跳板机 `A` 才能 SSH 服务器 `B`，那么在本机可以只运行一次命令直接传输文件到 `B`：

```console
# OpenSSH 7.3 以上
local> scp -oProxyJump=A thefile B:destination
```

或：

```console
local> scp -o ProxyCommand="ssh -W %h:%p userA@serverA" userB@serverB:/<remotePath> <localpath>
```

来源：[https://superuser.com/questions/276533/scp-files-via-intermediate-host](https://superuser.com/questions/276533/scp-files-via-intermediate-host)

## 本机通过多个跳板机连接 MySQL

如果本机需要通过跳板机 `A` ssh 连接跳板机 `B` 后才能连上服务器 `C` 上的 MySQL（经过两台跳板机），
那么本机的 `Sequel Pro` 客户端可以这样连上这个 MySQL 服务：

```console
# 先用 ssh 转发到本机 1234 端口：
local> ssh userA@serverA -L 1234:serverB:22
```
再用 `Sequel Pro` 通过 ssh 连接 MySQL，`Sequel Pro` ssh 配置如下：

```
MySQL Host: serverC
Username: db_user_name
Password: db_password
Database: db_name
Port: 3306

SSH Host: localhost
SSH User: ssh-user-serverB
SSH Key: ssh-password-of-serverB
SSH Port: 1234
```

来源：[https://stackoverflow.com/questions/10023494/how-to-connect-mysql-database-through-two-ssh-hosts](https://stackoverflow.com/questions/10023494/how-to-connect-mysql-database-through-two-ssh-hosts)

## 本机通过跳板机 A 连接 Redis

```console
local> ssh -L 9999:redis-host:6379 -p 远程ssh端口 userA@serverA
# 或
local> ssh -N -L 9999:redis-host:6379 -p 远程ssh端口 userA@serverA
# -N 表示不执行远程命令，man ssh
```

然后：

```
local> redis-cli -p 9999
```

来源：[http://momolog.info/2011/12/02/connect-to-redis-via-ssh-tunneling/](http://momolog.info/2011/12/02/connect-to-redis-via-ssh-tunneling/)

可以编写脚本把它们组合起来，本机上运行该脚本即可借助跳板机连上 Redis：

```sh
#!/bin/sh

PORT=9999
REDIS_HOST=redis-host
REDIS_PORT=6379
SOCK_PATH=/tmp/${REDIS_HOST}.sock
SSH_PORT=22
SSH_USER=userA
SSH_HOST=serverA

ssh -N -M -S ${SOCK_PATH} -L ${PORT}:${REDIS_HOST}:${REDIS_PORT} -p ${SSH_PORT} ${SSH_USER}@${SSH_HOST} &

# 结束脚本时退出 ssh
trap "ssh -S ${SOCK_PATH} -O exit ${SSH_USER}@${SSH_HOST}" EXIT

# 等待 ssh 连接建立
sleep 1

redis-cli -p ${PORT}
```

## 远程主机网络抓包

在分析网络协议时，常常需要借助 wireshark 等抓包工具捕获、显示网络数据。但 wireshark 依赖图形界面系统，我们一般不具备直接在远程主机运行 wireshark 的条件，所以要捕获远程主机的网络包时，就需要在远程主机运行 tcpdump 这个命令行工具了。但 tcpdump 显示的结果没有 wireshark 直观，我们可以结合两个工具，实时地通过本机的 wireshark 显示远程主机 tcpdump 捕获的网络数据。

以捕获远程主机 serverA "eth0" 网卡的 TCP 2001 端口网络数据，并实时显示到本机的 wireshark 为例:

```
local> ssh userA@serverA tcpdump -U -s0 -w - -i eth0 'port 2001' | wireshark -k -i -
```

`-U` 表示禁用文件缓冲，`-s0` 表示不截断 tcpdump 的每行数据结果，`-w -` 表示将结果输出到终端，对应于 wireshark 的 `-i -` 选项表示从终端读入数据。

如果不需要在 wireshark 实时显示数据，也可以通过 tcpdump 输出到文件，再下载回本机，使用 wireshark 分析该文件。
