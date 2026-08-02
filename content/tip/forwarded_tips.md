---
title: "网络流量中转技巧荟萃"
layout: page
date: 2019-03-23 19:00
updated: 2026-08-02
---

[TOC]

本文整理 OpenSSH 的跳板、端口转发、反向隧道和远程抓包用法。示例中的主机名、用户名、端口和路径都是占位符，请按实际环境替换。

本文统一使用下面的角色：

~~~text
A / a.example.com    有公网 IP 的 SSH 中转机
B / b.example        内网主机，不能被公网直接访问
C                     外部客户端
~~~

## 先判断转发方向

SSH 的选项名称容易混淆。先判断“监听端口出现在哪里”，再选择命令：

| 选项 | 监听端口 | 连接目标 | 常见用途 |
| --- | --- | --- | --- |
| `-L` | SSH 客户端本机 | SSH 服务端能访问的地址 | 本机访问远程数据库、Redis、HTTP |
| `-R` | SSH 服务端本机 | SSH 客户端能访问的地址 | 内网主机主动连公网机，建立反向 SSH |
| `-D` | SSH 客户端本机 | 由 SOCKS 客户端动态指定 | 临时 SOCKS 代理 |
| `-J` | 不额外监听端口 | 通过跳板机连接最终主机 | 登录、SCP、SFTP、端口转发 |
| `-W` | 不额外监听端口 | 把标准输入输出连接到 `host:port` | `ProxyCommand` 的底层实现 |

图示如下：

~~~text
-L: C:13306  ->  SSH server -> serverC:3306
-R: A:2222   ->  B:127.0.0.1:22
-D: C:1080   ->  由 SOCKS 客户端指定的目标
~~~

建议在纯转发命令中使用下面这些选项：

~~~console
C> ssh -NT -o ExitOnForwardFailure=yes ...
~~~

- `-N`：不执行远程命令，只建立转发。
- `-T`：不分配伪终端。
- `ExitOnForwardFailure=yes`：转发监听失败时立即退出，避免脚本或 systemd 把“SSH 已连接但转发没建立”误判为成功。
- `ServerAliveInterval`、`ServerAliveCountMax`：检测失联并让自动重启机制接管。

`-A` 是 SSH agent forwarding（代理转发），不是端口转发。只有确实需要让跳板机代用本机 SSH agent 时才启用；跳板机上的管理员或被入侵的进程可能利用转发中的 agent 请求签名。

## 通过一台或多台跳板机登录

### `ProxyJump`：推荐方式

OpenSSH 7.3 及以上可以直接使用 `-J`：

~~~console
C> ssh -J userA@serverA userB@serverB
~~~

多台跳板机用逗号分隔：

~~~console
C> ssh -J user1@server1,user2@server2 user3@server3
~~~

也可以写入 `~/.ssh/config`：

~~~sshconfig
Host server-a
    HostName a.example.com
    User your_a_user

Host server-b
    HostName b.example.com
    User b_user
    ProxyJump server-a
~~~

以后直接运行：

~~~console
C> ssh server-b
~~~

这种方式中，SSH 会在本机建立到跳板机的连接，并通过跳板机的 `direct-tcpip` 通道连接最终主机；跳板机不需要保存最终主机的私钥。

### 旧版本的 `ProxyCommand`

不支持 `-J` 时可以使用 `-W`：

~~~console
C> ssh -o 'ProxyCommand=ssh -W %h:%p userA@serverA' userB@serverB
~~~

等价的配置写法：

~~~sshconfig
Host server-b
    HostName b.example.com
    User b_user
    ProxyCommand ssh -W %h:%p server-a
~~~

### 嵌套 SSH

也可以在 A 上再启动一次 SSH：

~~~console
C> ssh -t userA@serverA ssh -t userB@serverB
~~~

如果 B 的认证需要使用 C 本机的 agent，才加上 `-A`：

~~~console
C> ssh -A -t userA@serverA ssh -t userB@serverB
~~~

这会把 agent 暴露给 A，安全边界比 `ProxyJump` 更宽。更推荐把最终主机的 key 保存在 C，或者使用 `ProxyJump`。

## 通过跳板机传输文件

### SCP 和 SFTP

~~~console
C> scp -J userA@serverA ./thefile userB@serverB:/destination/
C> sftp -J userA@serverA userB@serverB
~~~

多台跳板机的写法与 `ssh` 相同：

~~~console
C> scp -J user1@server1,user2@server2 ./thefile user3@server3:/destination/
~~~

不支持 `-J` 的旧版本可以写成：

~~~console
C> scp -o 'ProxyCommand=ssh -W %h:%p userA@serverA' \
    ./thefile userB@serverB:/destination/
~~~

### Rsync

~~~console
C> rsync -av -e 'ssh -J userA@serverA' ./local-dir/ \
    userB@serverB:/destination/
~~~

大批量文件传输时，`rsync` 通常比反复执行 `scp` 更适合；先在 SSH 配置中定义跳板机别名，可以让 `rsync` 命令更短。

## 本机访问远程 TCP 服务：`-L`

### 通用形式

~~~console
C> ssh -NT -o ExitOnForwardFailure=yes \
    -L 127.0.0.1:local_port:remote_host:remote_port \
    userA@serverA
~~~

含义是：在 C 的 `127.0.0.1:local_port` 监听，连接经过 SSH 后，由 A 访问 `remote_host:remote_port`。

`remote_host` 是从 SSH 服务端所在主机的网络视角解析和连接的，不是从 C 解析。例如 `redis-host` 是 A 能访问的内网名称时，C 无需解析它。

### 经过多台跳板机连接 MySQL

假设数据库主机 `serverC` 对 C 不可达，但 B 可以访问 `serverC:3306`，C 需要先经 A 再到 B：

~~~console
C> ssh -NT -o ExitOnForwardFailure=yes \
    -J userA@serverA \
    -L 127.0.0.1:13306:serverC:3306 \
    userB@serverB
~~~

然后让本机客户端连接：

~~~console
C> mysql -h 127.0.0.1 -P 13306 -u db_user_name -p db_name
~~~

图形化客户端的关键配置是：

~~~text
MySQL Host: 127.0.0.1
MySQL Port: 13306
MySQL User: db_user_name
~~~

SSH 隧道可以单独在终端运行，GUI 不需要再建立第二条 SSH 隧道。

如果 GUI 只支持“通过 SSH 登录数据库”，不能使用 `ProxyJump`，可以先把 B 的 SSH 端口转到 C：

~~~console
C> ssh -NT -L 127.0.0.1:1234:serverB:22 userA@serverA
~~~

然后在 GUI 中把 SSH Host 设置为 `127.0.0.1`、SSH Port 设置为 `1234`，数据库目标仍填写 `serverC:3306`。此时 GUI 建立的第二条 SSH 连接会落到 B，再由 B 访问 C。

### 通过跳板机连接 Redis

~~~console
C> ssh -NT -o ExitOnForwardFailure=yes \
    -L 127.0.0.1:9999:redis-host:6379 \
    -p 22 userA@serverA
~~~

另开终端使用本机端口：

~~~console
C> redis-cli -h 127.0.0.1 -p 9999
~~~

也可以把隧道和客户端组合成脚本。下面的写法使用 SSH control socket 检查隧道是否真的建立，避免固定 `sleep 1` 带来的竞态：

~~~sh
#!/bin/sh

set -eu

PORT=9999
REDIS_HOST=redis-host
REDIS_PORT=6379
SSH_PORT=22
SSH_USER=userA
SSH_HOST=serverA
SSH_TARGET=$SSH_USER@$SSH_HOST
SOCK_PATH=/tmp/redis-ssh-tunnel.sock
tunnel_pid=

cleanup() {
    if ! ssh -S "$SOCK_PATH" -O exit -p "$SSH_PORT" "$SSH_TARGET" \
        >/dev/null 2>&1; then
        if [ -n "$tunnel_pid" ]; then
            kill "$tunnel_pid" 2>/dev/null || true
        fi
    fi
}
trap cleanup EXIT

ssh -NT -M -S "$SOCK_PATH" \
    -o ExitOnForwardFailure=yes \
    -o ServerAliveInterval=30 \
    -o ServerAliveCountMax=3 \
    -L "127.0.0.1:$PORT:$REDIS_HOST:$REDIS_PORT" \
    -p "$SSH_PORT" "$SSH_TARGET" &
tunnel_pid=$!

i=0
while ! ssh -S "$SOCK_PATH" -O check -p "$SSH_PORT" "$SSH_TARGET" \
    >/dev/null 2>&1; do
    i=$((i + 1))
    [ "$i" -lt 20 ] || exit 1
    kill -0 "$tunnel_pid" 2>/dev/null || exit 1
    sleep 1
done

redis-cli -h 127.0.0.1 -p "$PORT"
~~~

长时间运行的服务应优先使用 systemd、launchd 或专门的 supervisor；上面的脚本适合本机临时使用。

## 临时建立 SOCKS 代理：`-D`

~~~console
C> ssh -NT -D 127.0.0.1:1080 userA@serverA
~~~

把浏览器或支持 SOCKS5 的客户端配置为 `127.0.0.1:1080`。命令行测试可以使用：

~~~console
C> curl --proxy socks5h://127.0.0.1:1080 https://example.com/
~~~

`socks5h` 中的 `h` 表示让代理端解析域名，避免 DNS 请求仍从 C 发出。SSH 的动态转发主要承载 TCP，不是完整 VPN，不能直接转发所有 UDP、广播或 ICMP 流量。

## 远程主机网络抓包

远程主机通常没有图形界面，可以在远程运行 `tcpdump`，把 pcap 流实时交给本机 Wireshark。

以捕获 A 的 `eth0` 网卡上 TCP 2001 端口的数据为例：

~~~console
C> ssh -T userA@serverA \
    "sudo tcpdump -U -s 0 -w - -i eth0 'tcp port 2001'" \
    | wireshark -k -i -
~~~

- `-U`：让 pcap 尽快刷新，适合实时管道。
- `-s 0`：使用完整 snaplen，避免截断数据包。
- `-w -`：把二进制 pcap 写到标准输出。
- Wireshark 的 `-i -`：从标准输入读取 pcap。

如果不需要实时显示，可以保存后再分析：

~~~console
C> ssh -T userA@serverA \
    "sudo tcpdump -U -s 0 -w - -i eth0 'tcp port 2001'" \
    > serverA-2001.pcap
C> wireshark serverA-2001.pcap
~~~

远程 `sudo` 必须已经具备执行权限，或在一个单独的交互式会话中完成认证；不要让 sudo 的密码提示混入 pcap 数据流。抓包通常需要 root 或 `CAP_NET_RAW`、`CAP_NET_ADMIN` 等权限。

## 通过公网主机反向访问内网主机：`-R`

### 原理和拓扑

当 B 没有公网 IP，但 B 能主动访问 A 的 SSH 端口时，让 B 主动建立到 A 的 SSH 连接，并在 A 上请求一个远程监听端口：

~~~text
B  --主动 SSH 连接-->  A

A:127.0.0.1:2222  --反向隧道-->  B:127.0.0.1:22
~~~

`-R` 的语法是：

~~~text
-R [listen_address:]listen_port:target_host:target_port
~~~

左侧监听端口出现在 SSH 服务端 A，右侧目标从 SSH 客户端 B 的网络视角访问。这个方案不是“让 A 主动穿过 NAT”，而是利用 B 已经建立的出站连接；B 必须能访问 A，A 的 SSH 端口也必须允许该连接。

### 前置条件

A 需要：

- 有公网 IP 或公网域名。
- 对 B 开放 SSH 服务端口。
- 安装并运行 `openssh-server`。
- 允许远程端口转发，但最好只允许专用用户和专用端口。

B 需要：

- 能主动连接 A 的 SSH 端口。
- 安装 `openssh-client`。
- 最终要被登录时，必须运行自己的 `sshd`。

C 需要：

- 能访问 A。
- 安装 SSH 客户端。

### B 开启 SSH 服务

Debian 或 Ubuntu 示例：

~~~console
B> sudo apt update
B> sudo apt install openssh-server
B> sudo systemctl enable --now ssh
B> systemctl status ssh --no-pager
~~~

如果 B 完全只通过反向隧道访问，可以让 SSH 只监听回环地址，减少局域网暴露面。优先使用发行版已经 Include 的 drop-in 目录：

~~~console
B> sudoedit /etc/ssh/sshd_config.d/99-listen-localonly.conf
~~~

内容：

~~~text
ListenAddress 127.0.0.1
~~~

确认主配置中没有与之冲突的 `ListenAddress`，然后检查并重载：

~~~console
B> sudo sshd -t
B> sudo systemctl reload ssh
~~~

如果 B 还需要被同一局域网直接 SSH，不要只监听 `127.0.0.1`；应监听明确的局域网地址，并配合关闭密码登录、限制用户和主机防火墙。

### A 创建隧道专用用户和 key

不要让 B 使用 A 的 root 或日常管理账号建立常驻隧道。A 上创建专用用户：

~~~console
A> sudo adduser --disabled-password --gecos "" tunnel
~~~

B 上单独生成连接 A 的 key，不要复用日常登录 key：

~~~console
B> ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519_to_host_a
B> chmod 600 ~/.ssh/id_ed25519_to_host_a
~~~

如果 A 的 `tunnel` 用户禁用了密码登录，`ssh-copy-id` 不能完成安装。可以使用 A 的管理账号登录后手工安装公钥：

~~~console
B> cat ~/.ssh/id_ed25519_to_host_a.pub
A> sudo install -d -m 700 -o tunnel -g tunnel /home/tunnel/.ssh
A> sudoedit /home/tunnel/.ssh/authorized_keys
~~~

写入公钥后修正权限：

~~~console
A> sudo chown -R tunnel:tunnel /home/tunnel/.ssh
A> sudo chmod 700 /home/tunnel/.ssh
A> sudo chmod 600 /home/tunnel/.ssh/authorized_keys
~~~

### 限制隧道 key 和 A 的 sshd

在 A 的 `authorized_keys` 中，给 B 的 key 加上限制：

~~~text
restrict,port-forwarding,permitlisten="127.0.0.1:2222" ssh-ed25519 AAAA... reverse-ssh-to-b
~~~

这里的重点是：

- `restrict` 默认关闭伪终端、agent forwarding、X11 forwarding、用户 rc 和端口转发等能力。
- `port-forwarding` 只重新开启端口转发能力。
- `permitlisten` 只允许该 key 在 A 的 `127.0.0.1:2222` 监听。
- `permitlisten` 限制的是 `-R` 的监听端；不要用只限制 `-L` 的 `permitopen` 替代它。

A 的 `/etc/ssh/sshd_config` 可以为 `tunnel` 用户增加单独的 `Match`：

~~~text
Match User tunnel
    AllowTcpForwarding remote
    GatewayPorts no
    PermitListen 127.0.0.1:2222
    X11Forwarding no
    AllowAgentForwarding no
    PermitTTY no
    ClientAliveInterval 30
    ClientAliveCountMax 3
~~~

`GatewayPorts no` 保证远程转发端口默认只对 A 本机可见；`ClientAlive*` 用于在 B 断网、睡眠或代理异常后清理 A 上失效的 SSH 会话，避免旧会话长期占住 `2222`。

检查实际生效配置：

~~~console
A> sudo sshd -t
A> sudo systemctl reload ssh
A> sudo sshd -T -C user=tunnel,host=localhost,addr=127.0.0.1 \
    | rg 'allowtcpforwarding|gatewayports|permitlisten|clientalive|permittty'
~~~

`restrict`、`permitlisten` 和 `PermitListen` 需要较新的 OpenSSH。如果旧版本不认识这些选项，应先升级或查阅对应版本的 `sshd(8)`，不要为了绕过错误而去掉所有限制。

### B 手动建立和验证隧道

B 上先用前台命令测试：

~~~console
B> ssh -NT \
    -i ~/.ssh/id_ed25519_to_host_a \
    -o BatchMode=yes \
    -o ExitOnForwardFailure=yes \
    -o ServerAliveInterval=30 \
    -o ServerAliveCountMax=3 \
    -o TCPKeepAlive=no \
    -o StrictHostKeyChecking=yes \
    -R 127.0.0.1:2222:127.0.0.1:22 \
    tunnel@a.example.com
~~~

第一次连接 A 时应通过可信渠道核对 A 的主机 key 指纹，再保存到 B 的 `known_hosts`；生产配置不要使用 `StrictHostKeyChecking=no` 或把 `known_hosts` 指向 `/dev/null`。

A 上检查监听并直接测试 B：

~~~console
A> ss -lntp | rg ':2222'
A> ssh -p 2222 b_user@127.0.0.1
~~~

C 上通过 A 的普通 SSH 登录作为跳板，访问 A 本机的 `2222`：

~~~console
C> ssh -J your_a_user@a.example.com -p 2222 b_user@127.0.0.1
~~~

这条命令的关键是 `-J`：C 先连接 A，再让 A 连接自己的 `127.0.0.1:2222`。因为 `2222` 只监听 A 的回环地址，C 不应直接执行 `ssh -p 2222 a.example.com`。

反向 SSH 不只可以转发 B 的 SSH，也可以转发 B 上的 HTTP、数据库等 TCP 服务。例如把 B 的 `8080` 暴露为 A 本机的 `18080`：

~~~console
B> ssh -NT ... -R 127.0.0.1:18080:127.0.0.1:8080 tunnel@a.example.com
~~~

此时 A 本机的程序可以访问 `127.0.0.1:18080`。如果 C 需要访问这个非 SSH 服务，通常再从 C 开一条到 A 的本地转发：

~~~console
C> ssh -NT -L 127.0.0.1:18080:127.0.0.1:18080 your_a_user@a.example.com
~~~

每增加一个反向入口，都应分配独占端口，并同步更新 `PermitListen` 和对应 key 的 `permitlisten`。

### 使用 systemd 自动重连

手动前台命令适合验证，不适合长期运行。B 上创建 `/etc/systemd/system/reverse-ssh-to-host-a.service`：

~~~ini
[Unit]
Description=Reverse SSH tunnel to Host A
After=network-online.target
Wants=network-online.target
StartLimitIntervalSec=0

[Service]
Type=simple
User=b_user
Environment=HOME=/home/b_user
ExecStart=/usr/bin/ssh -NT \
  -i /home/b_user/.ssh/id_ed25519_to_host_a \
  -o BatchMode=yes \
  -o ConnectTimeout=15 \
  -o ExitOnForwardFailure=yes \
  -o ServerAliveInterval=30 \
  -o ServerAliveCountMax=3 \
  -o TCPKeepAlive=no \
  -o StrictHostKeyChecking=yes \
  -o LogLevel=VERBOSE \
  -R 127.0.0.1:2222:127.0.0.1:22 \
  tunnel@a.example.com
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
~~~

确认 key、`.ssh` 和 `known_hosts` 都属于 `b_user`，再启用服务：

~~~console
B> sudo systemctl daemon-reload
B> sudo systemctl enable --now reverse-ssh-to-host-a.service
B> systemctl status reverse-ssh-to-host-a.service
B> journalctl -u reverse-ssh-to-host-a.service -f
~~~

如果之前手动测试命令还在运行，先停止它，否则 A 的 `2222` 已经被占用，systemd 会反复报告 `remote port forwarding failed`。

`ServerAlive*` 让 B 在到 A 的连接失效时主动退出；A 侧的 `ClientAlive*` 负责清理 A 上已经失联的旧会话。两边一起配置，自动重连才可靠。

## macOS 没有管理员权限时使用 launchd

如果 macOS 没有权限开启系统级 Remote Login，可以在用户目录运行一个只监听高位端口的独立 `sshd`，再用用户态 `launchd` 保持反向隧道。这个方案与 Linux 的 systemd 二选一。

### 用户态 sshd

以下示例使用 Homebrew OpenSSH；路径以 Apple Silicon 为例，Intel macOS 或系统 OpenSSH 请先用 `command -v sshd` 确认实际路径。

~~~console
macOS> brew install openssh
macOS> mkdir -p ~/.config/reverse-ssh-tunnel-macos/logs
macOS> ssh-keygen -t ed25519 \
    -f ~/.config/reverse-ssh-tunnel-macos/ssh_host_ed25519_key -N ''
macOS> ssh-keygen -t ed25519 \
    -f ~/.ssh/id_ed25519_to_host_a_macos_tunnel
macOS> cat ~/.ssh/id_ed25519_to_host_a_macos_tunnel.pub
~~~

创建 `/Users/your_user/.config/reverse-ssh-tunnel-macos/sshd_config`：

~~~text
Port 22022
ListenAddress 127.0.0.1
AddressFamily inet
HostKey /Users/your_user/.config/reverse-ssh-tunnel-macos/ssh_host_ed25519_key
AuthorizedKeysFile /Users/your_user/.ssh/authorized_keys_reverse_tunnel_macos
PubkeyAuthentication yes
PasswordAuthentication no
KbdInteractiveAuthentication no
ChallengeResponseAuthentication no
UsePAM no
AllowUsers your_user
AllowTcpForwarding no
X11Forwarding no
SetEnv LANG=en_US.UTF-8 LC_CTYPE=en_US.UTF-8
~~~

如果还需要同一局域网客户端直连，把 `ListenAddress` 改成 `0.0.0.0`，但必须保持公钥认证、关闭密码登录、限制 `AllowUsers`，并用 macOS 防火墙或可信网络边界限制 `22022`。完全不需要局域网直连时，保持 `127.0.0.1`。

把允许登录该用户态 sshd 的客户端公钥写入单独文件，并检查配置：

~~~console
macOS> touch ~/.ssh/authorized_keys_reverse_tunnel_macos
macOS> chmod 600 ~/.ssh/authorized_keys_reverse_tunnel_macos
macOS> /opt/homebrew/sbin/sshd -t \
    -f ~/.config/reverse-ssh-tunnel-macos/sshd_config
~~~

### 两个 launchd agent

`launchd` 的 `ProgramArguments` 使用绝对路径，不会替你展开 `~`。创建 `~/Library/LaunchAgents/com.example.reverse-ssh.local-sshd.plist`：

~~~xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>Label</key>
  <string>com.example.reverse-ssh.local-sshd</string>
  <key>ProgramArguments</key>
  <array>
    <string>/opt/homebrew/sbin/sshd</string>
    <string>-D</string>
    <string>-e</string>
    <string>-f</string>
    <string>/Users/your_user/.config/reverse-ssh-tunnel-macos/sshd_config</string>
  </array>
  <key>RunAtLoad</key>
  <true/>
  <key>KeepAlive</key>
  <true/>
  <key>StandardErrorPath</key>
  <string>/Users/your_user/.config/reverse-ssh-tunnel-macos/logs/local-sshd.stderr.log</string>
</dict>
</plist>
~~~

创建 `~/Library/LaunchAgents/com.example.reverse-ssh.to-host-a.plist`：

~~~xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>Label</key>
  <string>com.example.reverse-ssh.to-host-a</string>
  <key>ProgramArguments</key>
  <array>
    <string>/usr/bin/ssh</string>
    <string>-NT</string>
    <string>-i</string>
    <string>/Users/your_user/.ssh/id_ed25519_to_host_a_macos_tunnel</string>
    <string>-o</string>
    <string>BatchMode=yes</string>
    <string>-o</string>
    <string>ExitOnForwardFailure=yes</string>
    <string>-o</string>
    <string>ServerAliveInterval=30</string>
    <string>-o</string>
    <string>ServerAliveCountMax=3</string>
    <string>-o</string>
    <string>TCPKeepAlive=no</string>
    <string>-o</string>
    <string>StrictHostKeyChecking=yes</string>
    <string>-R</string>
    <string>127.0.0.1:2223:127.0.0.1:22022</string>
    <string>tunnel@a.example.com</string>
  </array>
  <key>RunAtLoad</key>
  <true/>
  <key>KeepAlive</key>
  <true/>
  <key>ThrottleInterval</key>
  <integer>10</integer>
  <key>StandardErrorPath</key>
  <string>/Users/your_user/.config/reverse-ssh-tunnel-macos/logs/reverse-tunnel.stderr.log</string>
</dict>
</plist>
~~~

A 上应为 macOS 的 key 单独增加：

~~~text
restrict,port-forwarding,permitlisten="127.0.0.1:2223" ssh-ed25519 AAAA... reverse-ssh-from-macos
~~~

加载并查看状态：

~~~console
macOS> launchctl bootstrap gui/$(id -u) \
    "$HOME/Library/LaunchAgents/com.example.reverse-ssh.local-sshd.plist"
macOS> launchctl bootstrap gui/$(id -u) \
    "$HOME/Library/LaunchAgents/com.example.reverse-ssh.to-host-a.plist"
macOS> launchctl kickstart -k gui/$(id -u)/com.example.reverse-ssh.local-sshd
macOS> launchctl kickstart -k gui/$(id -u)/com.example.reverse-ssh.to-host-a
macOS> launchctl print gui/$(id -u)/com.example.reverse-ssh.to-host-a
macOS> lsof -nP -iTCP:22022 -sTCP:LISTEN
~~~

检查日志：

~~~console
macOS> sed -n '1,120p' \
    ~/.config/reverse-ssh-tunnel-macos/logs/local-sshd.stderr.log
macOS> sed -n '1,120p' \
    ~/.config/reverse-ssh-tunnel-macos/logs/reverse-tunnel.stderr.log
~~~

独立的用户态 `sshd` 不一定继承系统 `/etc/ssh/sshd_config` 的 `AcceptEnv`。因此配置中的 `SetEnv LANG=... LC_CTYPE=...` 不只是显示优化，也能避免 Linux 客户端传入 macOS 不支持的 locale 导致中文乱码。

## 局域网优先、反向隧道兜底

客户端和 B 在同一局域网时，始终绕公网 A 会增加延迟，也会让文件传输带宽受 A 限制。建议保留三个入口：

~~~text
host-b-lan    强制局域网直连，用于测速和排障
host-b-via-a  强制通过 A，用于外网访问和排障
host-b        自动判断，作为日常入口
~~~

### SSH 配置和 HostKeyAlias

~~~sshconfig
Host host-a
    HostName a.example.com
    User your_a_user

Host host-b-via-a
    HostName 127.0.0.1
    User b_user
    Port 2222
    ProxyJump host-a
    IdentityFile ~/.ssh/id_ed25519_to_host_b
    IdentitiesOnly yes
    HostKeyAlias host-b-sshd

Host host-b-lan
    HostName b.local
    User b_user
    Port 22
    IdentityFile ~/.ssh/id_ed25519_to_host_b
    IdentitiesOnly yes
    HostKeyAlias host-b-sshd
    ConnectTimeout 2
    ConnectionAttempts 1
~~~

`HostKeyAlias` 让两条路径共用同一个逻辑主机 key 记录，但只有在局域网路径和反向路径确实连接到同一台机器、并已核对 ED25519 指纹后才可以这样设置。不同内网机器必须使用不同的 alias。

### 使用 mDNS 应对局域网 IP 变化

macOS 可以使用 Bonjour/mDNS 名称，而不是把 DHCP 地址写死：

~~~console
macOS> sudo scutil --set LocalHostName b
Linux> getent hosts b.local
Linux> avahi-resolve-host-name b.local
~~~

`.local` 通常只在同一二层网络可用。访客 Wi-Fi 的客户端隔离、不同 VLAN、禁用组播，都可能导致解析失败。跨 VLAN 时应使用路由器的 DHCP 地址保留、本地 DNS，或 Tailscale/WireGuard 等组网方案。

不要让内网主机定时把 `192.168.x.x` 等地址上报到 A：这些地址只在对应局域网有意义，还会引入多网卡选择、地址过期和上报鉴权等状态。

### mihomo TUN/Fake-IP 下按路由选路

Fake-IP 代理可能为域名返回 `198.18.0.0/16` 地址，并由 TUN 接口接管连接。只执行 `nc -z b.local 22` 可能把“已被代理接管”误判成“局域网直连”。Linux 客户端可以先解析 IPv4，再用 `ip route get` 判断是否经过网关：

~~~sh
#!/bin/sh

set -u

host=$1
port=$2
jump_host=$3
relay_target=$4

relay() {
    exec ssh -q "$jump_host" -W "$relay_target"
}

address=$(getent ahostsv4 "$host" 2>/dev/null \
    | awk 'NR == 1 { print $1; exit }')
[ -n "$address" ] || relay

route=$(ip -4 route get "$address" 2>/dev/null) || relay
case " $route " in
    *" via "*) relay ;;
esac

exec nc "$address" "$port"
~~~

保存为 `~/.local/bin/ssh-lan-or-relay` 后检查：

~~~console
C> chmod 700 ~/.local/bin/ssh-lan-or-relay
C> sh -n ~/.local/bin/ssh-lan-or-relay
~~~

在自动入口中调用：

~~~sshconfig
Host host-b
    HostName b.local
    User b_user
    Port 22
    IdentityFile ~/.ssh/id_ed25519_to_host_b
    IdentitiesOnly yes
    HostKeyAlias host-b-sshd
    ProxyCommand ~/.local/bin/ssh-lan-or-relay %h %p host-a 127.0.0.1:2222
~~~

局域网地址没有 `via` 时直接使用 `nc`；解析失败或路由经过网关时，使用 `ssh host-a -W 127.0.0.1:2222` 进入 A 的反向入口。脚本是 Linux/IPv4 示例；macOS 可按 `route -n get` 和系统 `nc` 的输出调整。

不要在真正连接前频繁执行一次 `nc -z` 探测：在某些 SSH 防护或代理环境中，只有 TCP 建立却没有 SSH 握手的连接会产生额外日志、惩罚或延迟。

### 扩展到多台内网主机

每台机器都应拥有独立的局域网名称、A 上的回环端口、隧道 key 和 `HostKeyAlias`：

~~~text
机器       局域网名称    B SSH 端口    A 入口
host-b     b.local       22            127.0.0.1:2222
host-c     c.local       22            127.0.0.1:2224
host-d     d.local       22            127.0.0.1:2225
~~~

接入新机器时同步完成：

1. 在 A 的 `PermitListen` 和对应 key 的 `permitlisten` 中加入独占端口。
2. 建立 `-R 127.0.0.1:<A端口>:127.0.0.1:<B端口>`。
3. 配置强制局域网、强制 A 和自动选路三个 SSH 别名。
4. 在可信路径上核对主机 key 指纹，并设置唯一 `HostKeyAlias`。

## 安全和暴露面

- A 上使用专用隧道用户，不使用 root 或日常管理账号。
- 隧道 key 单独生成，权限设为 `600`，不要复用个人登录 key。
- `authorized_keys` 使用 `restrict`、`permitlisten`，并在 sshd 的 `Match User` 中再次限制。
- A 的反向端口使用 `127.0.0.1`，保持 `GatewayPorts no`；不需要时不要设置 `0.0.0.0`、`::` 或 `-g`。
- A 的公网防火墙只开放 SSH 服务端口，不要为了访问 `2222` 而把它暴露到公网。
- B 只通过隧道访问时，可让 `sshd` 只监听 `127.0.0.1`。
- 对外提供局域网 SSH 时，关闭密码认证、限制登录用户并设置主机防火墙。
- 生产环境核对并固定 A、B 的主机 key，不要使用 `StrictHostKeyChecking=no`。
- 只有在信任跳板机且确实需要时才使用 agent forwarding (`-A`)。
- 保持 OpenSSH、操作系统和反向隧道依赖的 supervisor 更新。

确认 A 的反向端口没有暴露到公网：

~~~console
A> ss -lntp | rg ':2222'
~~~

期望看到类似：

~~~text
LISTEN 0 128 127.0.0.1:2222 0.0.0.0:*
~~~

如果看到 `0.0.0.0:2222` 或 `:::2222`，检查 `-R` 的监听地址、A 的 `GatewayPorts`、`PermitListen` 和主机防火墙。

## 常见问题排查

### `remote port forwarding failed`

优先检查：

- A 上的端口已被旧隧道或其他程序占用。
- `AllowTcpForwarding` 没有包含 `remote`。
- `PermitListen` 或 key 的 `permitlisten` 与实际 `-R` 地址、端口不一致。
- A 的 OpenSSH 版本不认识限制选项。
- systemd/launchd 和手动测试命令同时运行，争抢同一个端口。

~~~console
A> ss -lntp | rg ':2222'
A> ps -o pid,ppid,lstart,etime,user,cmd -C sshd | rg 'tunnel|PID'
A> sudo sshd -T -C user=tunnel,host=localhost,addr=127.0.0.1 \
    | rg 'allowtcpforwarding|gatewayports|permitlisten|clientalive'
B> journalctl -u reverse-ssh-to-host-a.service -n 100 --no-pager
~~~

如果 A 上 `sshd: tunnel` 的启动时间明显早于 B 上 systemd 服务，可能是旧会话占用了端口。先核对 PID 和启动时间，只结束对应的旧 tunnel 子进程，不要杀掉 A 的主 `sshd` listener。旧会话释放后，systemd 会按 `RestartSec` 自动重试。

### 外部连接 `connection refused`

这通常表示 C 已经通过 A，但 A 的 `127.0.0.1:2222` 当前没有监听：

~~~console
A> ss -lntp | rg ':2222'
B> systemctl status reverse-ssh-to-host-a.service
B> journalctl -u reverse-ssh-to-host-a.service -n 100 --no-pager
~~~

### 外部连接卡住

说明隧道可能存在，但 B 侧的目标 SSH 服务没有正确响应。B 上检查：

~~~console
B> systemctl status ssh
B> ss -lntp | rg ':22'
B> ssh -p 22 b_user@127.0.0.1
~~~

如果 B 的 `sshd` 没有监听 `127.0.0.1:22`，而 `-R` 却写的是这个地址，隧道会建立但登录会卡住或被关闭。

### 网络恢复后没有自动重连

检查：

~~~console
B> systemctl status reverse-ssh-to-host-a.service
B> journalctl -u reverse-ssh-to-host-a.service -n 100 --no-pager
~~~

服务文件应至少有：

~~~ini
Restart=always
RestartSec=10
~~~

SSH 命令应包含：

~~~text
-o ExitOnForwardFailure=yes
-o ServerAliveInterval=30
-o ServerAliveCountMax=3
~~~

同时确认 B 到 A 的 DNS、路由、防火墙和代理规则没有阻断 SSH；可以临时运行 `ssh -vvv tunnel@a.example.com` 查看失败发生在解析、TCP、认证还是端口转发阶段。

### A 的反向端口意外暴露

依次确认：

1. `-R` 使用 `127.0.0.1:2222:...`，不是 `0.0.0.0:2222:...`。
2. A 的 `GatewayPorts no` 实际生效。
3. A 的 `authorized_keys` 使用 `permitlisten="127.0.0.1:2222"`。
4. `ss -lntp` 显示监听地址是 `127.0.0.1`，而不是 `0.0.0.0` 或 `::`。

### `Connection closed by UNKNOWN port 65535`

如果 C 连接一段时间后出现这个提示，同时 B 的日志反复出现 `remote port forwarding failed`，常见原因是 A 上旧的 `sshd: tunnel` 会话仍占用 `2222`。C 连接到的是旧监听，旧监听后端已经不健康，最终才被关闭。

先检查 A 上的监听者和启动时间，确认后只结束旧的 tunnel 子进程。长期解决方法是保留 B 侧 `ServerAlive*`，并在 A 的 `Match User tunnel` 中配置：

~~~text
ClientAliveInterval 30
ClientAliveCountMax 3
~~~

两类 keepalive 的职责不同：B 侧主动发现连接失效并重启，A 侧主动清理失联会话。

### macOS 登录后中文乱码

先判断是 locale 问题，而不是隧道修改了字节流：

~~~console
C> ssh host-b-via-a 'locale; printf "%s\n" "中文测试"'
~~~

如果 `LANG` 为空、`LC_CTYPE` 是 `C` 或非 UTF-8，检查用户态 `sshd_config` 是否包含：

~~~text
SetEnv LANG=en_US.UTF-8 LC_CTYPE=en_US.UTF-8
~~~

修改后重新检查配置并重启对应的 launchd agent。独立 `sshd` 不一定会继承系统 sshd 的 `AcceptEnv`；Linux 客户端发送的 `C.UTF-8` 也不一定是 macOS 支持的 locale。

### 打开调试日志

客户端逐级提高日志级别：

~~~console
C> ssh -v ...
C> ssh -vvv ...
~~~

查看实际生效的客户端配置：

~~~console
C> ssh -G host-b-via-a
~~~

服务端检查配置语法和条件配置：

~~~console
A> sudo sshd -t
A> sudo sshd -T -C user=tunnel,host=localhost,addr=127.0.0.1
~~~

## 停止和清理

临时停止 Linux 隧道：

~~~console
B> sudo systemctl stop reverse-ssh-to-host-a.service
~~~

禁止开机启动但保留服务文件：

~~~console
B> sudo systemctl disable reverse-ssh-to-host-a.service
~~~

确定不再使用时，先核对路径，再删除 B 的服务、专用 key 和 B 上的本地监听限制：

~~~console
B> sudo systemctl disable --now reverse-ssh-to-host-a.service
B> sudo rm -f /etc/systemd/system/reverse-ssh-to-host-a.service
B> sudo systemctl daemon-reload
B> rm -f ~/.ssh/id_ed25519_to_host_a ~/.ssh/id_ed25519_to_host_a.pub
B> ssh-keygen -R a.example.com
~~~

如果不再需要“B 的 SSH 只监听回环地址”，再删除对应的 drop-in 并重载 SSH；如果仍希望保留这个收紧策略，不要删除它。A 上还应从 `/home/tunnel/.ssh/authorized_keys` 删除 B 对应的公钥。

停止 macOS 的用户态 agent：

~~~console
macOS> launchctl bootout gui/$(id -u) \
    "$HOME/Library/LaunchAgents/com.example.reverse-ssh.to-host-a.plist"
macOS> launchctl bootout gui/$(id -u) \
    "$HOME/Library/LaunchAgents/com.example.reverse-ssh.local-sshd.plist"
~~~

确认不再需要后，再删除对应 plist、用户态 `sshd` 配置、日志、key 和授权公钥。A 上也要删除 macOS 对应的 `authorized_keys` 行，并收回不再使用的 `PermitListen` 端口。

## 参考

- `man ssh`、`man ssh_config`、`man sshd_config`、`man authorized_keys`
- [OpenSSH manual pages](https://man.openbsd.org/ssh)
- [Chain from one SSH terminal to another](https://stackoverflow.com/questions/20079646/is-it-possible-to-chain-from-one-terminal-to-another-via-ssh-in-one-series-of-co)
- [SCP files via an intermediate host](https://superuser.com/questions/276533/scp-files-via-intermediate-host)
- [Connecting to MySQL through two SSH hosts](https://stackoverflow.com/questions/10023494/how-to-connect-mysql-database-through-two-ssh-hosts)
- [Connect to Redis via SSH tunneling](http://momolog.info/2011/12/02/connect-to-redis-via-ssh-tunneling/)
