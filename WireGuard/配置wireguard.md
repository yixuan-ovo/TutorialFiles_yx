# WireGuard 配置教程

## 前置要求

- 部署在带公网 IP 的服务器
- 安全组放行 UDP 51820 端口

## 服务端配置

### 1. 安装 WireGuard

```bash
sudo apt update
sudo apt install -y wireguard iptables
```

> **注意**：上面命令顺便安装了 iptables，建议安装之前先检查一下服务器有没有自带 iptables：
> 
> ```bash
> iptables -v
> ```

确认安装成功：

```bash
wg -v
```

### 2. 开启 IP 转发

编辑系统配置文件：

```bash
sudo nano /etc/sysctl.conf
```

添加以下内容：

```
net.ipv4.ip_forward = 1
```

应用配置：

```bash
sudo sysctl -p
```

### 3. 生成服务器密钥

```bash
cd /etc/wireguard
sudo bash -c 'umask 077; wg genkey | tee server.key | wg pubkey > server.pub'
```

查看生成的密钥：

```bash
cat server.key  # 私钥
cat server.pub  # 公钥
```

> **重要**：记住这两个私钥和公钥

### 4. 配置服务端

编辑配置文件：

```bash
sudo nano /etc/wireguard/wg0.conf
```

填写如下内容：

```ini
[Interface]
# 网段,可自定义
Address = 192.168.126.1/24
# 监听端口
ListenPort = 51820
# 服务器私钥
PrivateKey = server.key
# 不做 NAT，只转发
PostUp   = iptables -A FORWARD -i wg0 -o wg0 -j ACCEPT
PostDown = iptables -D FORWARD -i wg0 -o wg0 -j ACCEPT
```

### 5. 启动服务

启动服务，检查是否报错：

```bash
sudo wg-quick up wg0
sudo systemctl enable wg-quick@wg0
```

## 客户端配置

### 1. 创建新隧道

如果服务端成功启动，下载一个 WireGuard 客户端。

点击左下角**新建空隧道**：

![](./img/1.png)

名称随便写，记住**公钥**和 **PrivateKey**，点击保存：

![](./img/2.png)

### 2. 配置客户端

接下来点击右下角**编辑**，输入如下配置，点击保存：

```ini
[Interface]
PrivateKey = (新增隧道自动生成的私钥,不需要动)
# 自己设置的ip,后面必须写/24
Address = 192.168.126.77/24
# 服务器端设置网段的网关
DNS = 192.168.126.1
MTU = 1380

[Peer]
PublicKey = (服务器端的公钥)
# 服务器端设置的网段
AllowedIPs = 192.168.126.0/24
Endpoint = 服务器的公网IP:开放的UDP端口号
# 链接保活间隔
PersistentKeepalive = 25
```

![](./img/3.png)

### 3. 配置服务端 Peer

还需要设置服务器端的配置文件：

```bash
sudo nano /etc/wireguard/wg0.conf
```

在配置文件中添加：

```ini
[Peer]
# 添加的第一位用户
PublicKey = 隧道的公钥
# 自己设定的本机ip,与客户端一致
AllowedIPs = 192.168.126.77/32
PersistentKeepalive = 25
```

保存后重载服务：

```bash
sudo systemctl restart wg-quick@wg0
```

### 4. 测试连接

客户端点击连接，如果一切正常，下方节点的流量应该是接收与发送都在涨，并且可以 ping 通 `192.168.xxx.1`：

![](./img/4.png)

## 添加其他用户

如果需要添加其他用户，复制并修改公钥和 IP 即可，服务器端也需要添加对应的 IP。

## 防火墙配置

如果都能 ping 通 `.1`，但是互相 ping 不通，需要在电脑防火墙进行配置：

1. 打开**防火墙** → **高级设置** → **入站规则** → **新建规则**
2. 依次选择：
   - **自定义**
   - **所有程序**
   - **协议类型**：任何
   - **添加远程 IP**（自己设定的网段的 IP/24）
   - **允许连接**
   - **三个全选**
   - **名称**建议填写：WireGuard

配置完成后即可互相 IP 之间 ping 通了。
