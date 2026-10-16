---
title: "3x-ui挖矿木马清理案例"
description: "3x-ui 面板服务器被挖矿木马入侵的完整排查与清理教程，包含 CPU 高占用排查、进程识别、门罗币挖矿木马删除、crontab 与 systemd 启动项清理、SSH 安全加固等详细步骤。"
bookCollapseSection: true
---

3x-ui 项目地址：https://github.com/MHSanaei/3x-ui

基于 x-ui 后续魔改版本，与其余 x-ui 魔改项目类似，后续也更新了内核，并支持了新协议和多用户。3x-ui 后续更新的实用功能是可以对 xray 核心进行修改。在美观程度和动画流畅程度上，3x-ui 与其余代理面板拉开了一个维度。

![3x-ui--10001.jpg](https://zhiblog.xyz/img/3x-ui--10001.jpg)

因为前一段时间爆出了 3x-ui 的高危漏洞，很多服务器被入侵。我的这台服务器被入侵后安装了挖矿服务。首先使用终端软件 [FinalShell](https://www.hostbuf.com/)，在登录 3x-ui 时会看到 CPU 占用情况，发现 CPU 一直是 100%。

![3x-ui--10002.jpg](https://zhiblog.xyz/img/3x-ui--10002.jpg)

然后使用命令行 `top` 命令查看进程情况，发现有一个进程 CPU 占用高达 96.4%，进程名称是 `system`。一般 Debian 服务器系统进程的名字是 `systemd`，因此引起了我的怀疑。

![3x-ui--10003.jpg](https://zhiblog.xyz/img/3x-ui--10003.jpg)

于是带着问题去咨询 ChatGPT。

![3x-ui--10004.jpg](https://zhiblog.xyz/img/3x-ui--10004.jpg)

| VPS 商家 | 最低价 | 特点 | 支付方式 |
| :--- | :--- | :--- | :--- |
| [BWG 版瓦工](https://bandwagonhost.com/vps-hosting.php?aff=73801) | $49.99/年 | **速度最快、最稳定** | 微信、支付宝、PayPal、信用卡 |
| [DMIT](https://www.dmit.io/cart.php?aff=8412) | $36.9/年 | CN2 GIA 线路、香港 CMI 线路 | **支付宝**、PayPal、信用卡 |
| [hostdare](https://bill.hostdare.com/aff.php?aff=3766&pid=60) | $16/年 | 大流量、速度快 CMI 线路 | **支付宝**、PayPal、信用卡 |
| [BuyVM](https://my.frantech.ca/aff.php?aff=7175) | ¥18 | 大硬盘、无限流量、流媒体解锁 | **支付宝**、PayPal、信用卡、数字货币 |
| [RackNerd](https://my.racknerd.com/aff.php?aff=6665) | $8.89/年 | **便宜**、大流量、高配置 | **支付宝**、PayPal、信用卡、数字货币 |
| [CloudCone](https://app.cloudcone.com.cn/vps/265/create?token=mcs-24-ssd-vps-1&ref=11035) | $13.99/月 | **便宜**、大流量、大硬盘 | **支付宝**、PayPal、信用卡 |
| [Vultr](https://www.vultr.com/?ref=9560397) | $5/月 | 全球几十机房任选，注册送 $100 | **支付宝**、PayPal、数字货币 |

这是 ChatGPT 给出的大概处理流程：

---

你这个进程信息里：

```
PID: 511620  
USER: root  
CPU: 96.7%  
TIME+: 7d+9h  
COMMAND: system
```

有几点需要注意：

1. **进程名 `system` 并不是正常的 Linux 系统守护进程名**
   - Linux 系统核心进程一般是 `systemd`（不是 `system`），所以这个名字很可疑。
   - 很多挖矿木马、恶意脚本会用 `system`、`kworker`、`bash` 这样的名字伪装。

2. **运行了 7 天 9 小时，高 CPU 占用**
   - 长时间高占用，很可能是恶意挖矿进程或者某个死循环任务。

3. **用户是 root**
   - 如果是恶意程序，说明它可能通过某种方式获得了 root 权限，危害更大。

### 先确认它是什么

运行：

```bash
ps -fp 511620
```

看完整路径和启动命令。

再用：

```bash
ls -l /proc/511620/exe
```

看它的可执行文件在哪个路径。

如果想查它的网络连接：

```bash
netstat -pantu | grep 511620
```

或：

```bash
ss -pantu | grep 511620
```

### 如果确认是恶意进程

1. **立即结束它**

```bash
kill -9 511620
```

2. **删除它的文件**  
   如果 `/proc/511620/exe` 指向的文件路径还存在，删除：

```bash
rm -f /路径/文件名
```

3. **清理启动项**  
   查一下定时任务和自启动：

```bash
crontab -l
ls /etc/cron* -R
cat /etc/rc.local
systemctl list-unit-files | grep enabled
```

4. **检查可疑用户**

```bash
cat /etc/passwd | grep -v '/nologin'
```

### 防止再次中招

- 更新系统：

```bash
apt update && apt upgrade -y
```

- 更改 root 密码：

```bash
passwd
```

- 禁用 root 远程 SSH 登录（编辑 `/etc/ssh/sshd_config`，设置 `PermitRootLogin no`，然后 `systemctl restart ssh`）。
- 安装 fail2ban 或其他防爆破工具。

---

我怀疑你这个 `system` 进程是个 **挖矿木马**。根据 ChatGPT 的提示，找到了挖矿木马，是一个门罗币挖矿程序，伪装成了系统进程。

![3x-ui--10005.jpg](https://zhiblog.xyz/img/3x-ui--10005.jpg)

以下是进一步清理步骤：

### 证据

- **`./Download/system`**：程序放在服务器的 `Download` 目录下，用了伪装名字 `system`。
- **`-o pool.hashvault.pro:443`**：`hashvault.pro` 是一个门罗币矿池的域名。
- **`-u 8AMoGmzK...`**：这是攻击者的钱包地址，服务器正在帮别人挖矿。
- **`-k --tls`**：这是 `xmrig` 挖矿程序常用参数（TLS 加密传输）。

### 现在要做的事

#### 1. 立刻杀掉进程

```bash
kill -9 511620
```

#### 2. 删除挖矿程序

```bash
rm -f /root/Download/system
```

如果不确定目录：

```bash
ls -l /root/Download
```

#### 3. 查找同类文件（防止它在别的地方留了备份）

```bash
find / -type f -name "system" -o -name "xmrig" 2>/dev/null
```

#### 4. 查启动项（防止重启后又运行）

```bash
crontab -l
ls -l /etc/cron* /var/spool/cron* 2>/dev/null
grep -r "Download/system" /etc/ 2>/dev/null
```

#### 5. 查可疑服务

```bash
systemctl list-units --type=service | grep system
```

#### 6. 更新密码，防止再次入侵

```bash
passwd
```

同时建议禁用 root 远程 SSH 登录，编辑：

```bash
nano /etc/ssh/sshd_config
```

修改：

```
PermitRootLogin no
```

保存后：

```bash
systemctl restart ssh
```

---

### 木马隐藏较深时的进一步排查

很多情况下木马放在用户自己的 crontab 或 systemd 启动项里。

#### 1. 查看 root 用户的 crontab

```bash
crontab -l -u root
```

如果里面有执行 `Download/system` 或 `xmrig` 的命令，全部删除。编辑：

```bash
crontab -e -u root
```

#### 2. 搜索所有定时任务文件中是否有挖矿命令

```bash
grep -r "Download/system" /etc/cron* /var/spool/cron* 2>/dev/null
```

#### 3. 检查 systemd 自启动服务

很多挖矿程序会伪装成 `system-update.service`、`network.service` 之类：

```bash
systemctl list-unit-files --type=service | grep enabled
systemctl status system
systemctl status system-update
```

如果发现有指向 `Download/system` 或陌生目录的，直接禁用并删除：

```bash
systemctl disable --now 服务名
rm -f /etc/systemd/system/服务名
systemctl daemon-reload
```

#### 4. 检查 `rc.local` 和开机脚本

```bash
cat /etc/rc.local
ls -l /etc/init.d/
grep -r "Download/system" /etc/init.d/
```

#### 5. 防止再次入侵

- **改密码**：`passwd`
- **禁止 root SSH 登录**（编辑 `/etc/ssh/sshd_config`，`PermitRootLogin no`）
- **仅允许密钥登录**（`PasswordAuthentication no`）
- 安装 Fail2ban 进行 SSH 防爆破保护：

```bash
apt install fail2ban -y
```

---

### 实际发现的 crontab 木马

crontab 第一行就是木马的定时任务：

```bash
* * * * * pgrep -x ibus || exec /root/.cache/ibus/ibus &
```

意思是：

- 每分钟检查一次是否有名为 `ibus` 的进程
- 如果没有，就运行 `/root/.cache/ibus/ibus` 这个程序（其实是挖矿木马，伪装成输入法 ibus）
- 这样即使杀掉进程，它一分钟后又会重启

第二行的 `~/.acme.sh/acme.sh --cron` 是正常的证书续签任务，不是木马。

### 清理方法

1. **删除木马定时任务**

```bash
crontab -e -u root
```

删除这行：

```bash
* * * * * pgrep -x ibus || exec /root/.cache/ibus/ibus &
```

保存退出。

2. **杀掉正在运行的木马进程**

```bash
pkill -f "/root/.cache/ibus/ibus"
```

3. **删除木马文件**

```bash
rm -rf /root/.cache/ibus
```

4. **检查 systemd 是否有木马自启**

```bash
systemctl list-unit-files --type=service | grep ibus
systemctl status ibus
```

如果发现异常的 ibus 服务：

```bash
systemctl disable --now ibus
rm -f /etc/systemd/system/ibus.service
systemctl daemon-reload
```

最后再删除守护进程、管理任务、修改管理员密码，重启服务器。
