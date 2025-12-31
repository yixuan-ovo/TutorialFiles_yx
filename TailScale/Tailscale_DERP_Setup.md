# Tailscale + 私有 DERP 高性能组网方案全记录

本方案旨在通过(国内云服务器)建立高性能中转节点，并优化防火墙规则，实现电脑间的 8ms(同省同运营商) 极致延迟直连。

## 一、服务器端基础环境准备

### 1. 开启系统转发

在 `/etc/sysctl.conf` 中添加以下内容以允许流量转发：

```bash
echo "net.ipv4.ip_forward = 1" | sudo tee -a /etc/sysctl.conf
```

执行以下命令使其生效：

```bash
sudo sysctl -p
```

### 2. 安装基础工具

```bash
sudo apt update && sudo apt install -y wget curl git ufw
```

## 二、安装与配置 Tailscale

### 1. 安装 Tailscale 客户端

```bash
curl -fsSL https://tailscale.com/install.sh | sh
```

### 2. 登录并启动

```bash
sudo tailscale up
```

根据提示点击 URL 并在浏览器中完成授权。

## 三、部署私有 DERP 服务器 (Go 方案)

### 1. 安装 Go 环境

下载并解压：

```bash
wget https://go.dev/dl/go1.25.5.linux-amd64.tar.gz
sudo tar -C /usr/local -xzf go1.25.5.linux-amd64.tar.gz
```

配置环境变量：

```bash
echo 'export PATH=$PATH:/usr/local/go/bin' >> ~/.bashrc
source ~/.bashrc
```

### 2. 下载并编译 DERPER

```bash
go install tailscale.com/cmd/derper@latest
```

### 3. 将程序移动到系统目录

为了方便管理和长期运行，建议把编译好的程序移出来：

```bash
sudo cp ~/go/bin/derper /usr/local/bin/
```

### 4. 放行服务器安全组（关键）

**云服务器安全组配置：**

- **TCP 协议**：端口 `31226`
- **UDP 协议**：端口 `12263`（STUN 协议，不通的话无法打洞直连）
- **UDP 协议**：端口 `41641`（Tailscale 用端口）

**服务器本地防火墙配置：**

```bash
sudo ufw allow 31226/tcp
sudo ufw allow 12263/udp
sudo ufw allow 41641/udp
sudo ufw reload
```

### 5. 配置 Systemd 守护进程（后台常驻）

创建一个服务文件，让它开机自启并持续运行：

```bash
sudo nano /etc/systemd/system/derp.service
```

复制并修改以下内容（注意替换 `你的公网IP`）：

```ini
[Unit]
Description=Tailscale DERP Server
After=network.target

[Service]
# -a :443 监听端口
# -stun 开启打洞辅助
# -http-port -1 禁用 HTTP 模式
# 因为保留了服务器上的 tailscale 客户端，所以不需要写 -verify-clients=false，默认更安全
# -a :31226 对应原来的 443
# -stun-port 12263 对应原来的 3478
ExecStart=/usr/local/bin/derper -hostname 你的公网IP -a :31226 -http-port -1 -stun -stun-port 12263 -certmode manual
Restart=always
User=root

[Install]
WantedBy=multi-user.target
```

### 6. 启动服务

```bash
sudo systemctl daemon-reload
sudo systemctl enable derp
sudo systemctl start derp
```

检查状态：

```bash
sudo systemctl status derp
```

如果看到绿色 "active (running)"，说明服务器端已就绪。

### 7. 修改 Tailscale 官网 ACL（最关键的一步）

如果不做这一步，设备还是会优先去连香港。

1. 登录 Tailscale Admin Console - Access Control
2. 在 JSON 配置文件中找到（或添加）`derpMap` 字段：

```json
"derpMap": {
    "OmitDefaultRegions": true, // 强制停用官方香港、日本等所有海外节点
    "Regions": {
        "901": {
            "RegionID": 901,
            "RegionCode": "ytq-server",
            "Nodes": [
                {
                    "Name": "1",
                    "RegionID": 901,
                    "HostName": "服务器公网IP",
                    "DERPPort": 31226,  // 必须对应 -a 参数
                    "STUNPort": 12263,  // 必须对应 -stun-port 参数
                    "IPv4": "服务器公网IP",
                    "InsecureForTests": true
                }
            ]
        }
    }
}
```

## 四、客户端验证与压力测试

### 1. 检查中转状态

在电脑终端执行：

```bash
tailscale netcheck
```

确保能看到你的自定义 DERP 节点（如 `ytq-server`）且延迟正常。

### 2. 验证直连 (P2P)

与另一台 Tailscale 设备握手后执行：

```powershell
tailscale status
```

期望结果：看到 `active; direct;` 字样。

### 3. 延迟测试

```powershell
ping <目标设备Tailscale-IP>
```

本地物理测速参考：由于优化了打洞与防火墙，同城延迟应在 5ms-10ms 之间。

## 五、常见维护方案

- **每次启动前,建议执行一遍ping对方电脑提前握手**
- **IP 变动**：若云服务器 IP 更改，需同步更新后台 JSON 配置。
- **直连失效**：优先检查本地路由器是否开启了 UPnP 或 Full Cone NAT。