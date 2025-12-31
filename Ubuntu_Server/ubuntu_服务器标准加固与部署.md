# Ubuntu 服务器标准加固与业务部署手册

---

## 一、基础环境初始化

### 1. 更新系统软件包

```bash
sudo apt update && sudo apt upgrade -y
```
---

## 二、创建普通用户

避免长期使用 root 账号。

```bash
sudo adduser ubuntu
sudo usermod -aG sudo ubuntu
```

验证：

```bash
su - ubuntu
sudo whoami
```

---

## 三、SSH 安全加固（核心步骤）

### 1. 本地生成密钥

```bash
ssh-keygen -t ed25519
```
生成的密钥文件位置：

- **Windows**: `C:\Users\你的用户名\.ssh\`
- **文件**: `id_ed25519`（私钥）和 `id_ed25519.pub`（公钥）

### 2. 上传公钥到服务器

```bash
mkdir -p ~/.ssh
echo "这里粘贴你刚才复制的公钥内容" >> ~/.ssh/authorized_keys
cat ~/.ssh/authorized_keys
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

### 3. 修改 SSH 配置

```bash
sudo nano /etc/ssh/sshd_config
```

**推荐最小安全配置：**

```conf
Port 38617

#指定 SSH 使用的协议版本
Protocol 2

#禁止 root 用户通过 SSH 直接登录
PermitRootLogin no

#允许使用 SSH 公钥 / 私钥 方式认证
PubkeyAuthentication yes

#彻底关闭密码登录
PasswordAuthentication no

AllowUsers ubuntu

#限制一次连接中允许的认证失败次数
MaxAuthTries 3

#连接建立后，允许用户完成认证的最长时间
LoginGraceTime 20

#服务器主动检测客户端是否存活
ClientAliveInterval 300

#连续多少次心跳无响应后断开连接
ClientAliveCountMax 2
```

重启 SSH：

```bash
sudo systemctl restart ssh
```

⚠️ **务必在断开当前连接前测试新端口登录是否成功。**

```bash
ssh -p 38617 -i ~/.ssh/id_ed25519 用户名@服务器IP
```

---

## 四、内核级安全加固（sysctl）

创建安全配置文件：

```bash
sudo nano /etc/sysctl.d/99-hardening.conf
```

推荐内容：

```conf
# 防止 IP 欺骗
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.default.rp_filter = 1

# 禁止 ICMP 重定向
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.default.accept_redirects = 0

# SYN Flood 防护
net.ipv4.tcp_syncookies = 1

# 忽略广播 ping
net.ipv4.icmp_echo_ignore_broadcasts = 1
```

应用配置：

```bash
sudo sysctl --system
```

---

## 五、iptables 防火墙策略（精准放行）

### 1. 清理并初始化规则

```bash
sudo iptables -F
sudo iptables -X
sudo iptables -Z
```

### 2. 基础允许规则

```bash
sudo iptables -A INPUT -i lo -j ACCEPT
sudo iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
sudo iptables -A INPUT -m conntrack --ctstate INVALID -j DROP
```

### 3. SSH 防暴力破解

```bash
sudo iptables -A INPUT -p tcp --dport 38617 -m conntrack --ctstate NEW -m recent --set
sudo iptables -A INPUT -p tcp --dport 38617 -m conntrack --ctstate NEW -m recent --update --seconds 60 --hitcount 4 -j DROP
sudo iptables -A INPUT -p tcp --dport 38617 -j ACCEPT
```

### 4. 业务端口放行

```bash
sudo iptables -A INPUT -p tcp --dport 端口 -j ACCEPT
sudo iptables -A INPUT -p udp --dport 端口 -j ACCEPT
```

如非公网必要，建议限制来源 IP。

### 5. 默认拒绝

```bash
sudo iptables -P INPUT DROP
```

### 6. 保存规则

```bash
sudo apt install -y iptables-persistent
sudo netfilter-persistent save
```

---

## 六、Fail2Ban 入侵防护

### 1. 安装

```bash
sudo apt install -y fail2ban
```

### 2. 配置 jail

```bash
sudo nano /etc/fail2ban/jail.local
```

```ini
[sshd]
enabled = true
port = 38617
maxretry = 3
findtime = 10m
bantime = 1h
```

重启并验证：

```bash
sudo systemctl restart fail2ban
sudo fail2ban-client status sshd
```

---

## 七、账户与权限强化

### 1. 锁定 root 密码

```bash
sudo passwd -l root
```

### 2. sudo 审计（可选）

```bash
sudo visudo
```

```conf
Defaults logfile="/var/log/sudo.log"
Defaults timestamp_timeout=5
```

---

## 八、日志与运维基础

确认日志系统运行：

```bash
sudo systemctl status rsyslog
```

建议启用 logrotate（通常已安装）：

```bash
sudo apt install -y logrotate
```

---

## 九、业务部署前检查清单

- [ ] SSH 密钥登录正常
- [ ] root 无法登录
- [ ] 非必要端口未暴露
- [ ] 防火墙策略已保存
- [ ] Fail2Ban 正常工作
- [ ] 系统时间 / 时区正确

---

## 十、安全维护

- 每月执行一次系统更新
- 定期检查 `/var/log/auth.log`

---

**适用于个人服务器**

