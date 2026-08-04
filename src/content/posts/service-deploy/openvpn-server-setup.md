---
title: "OpenVPN 服务端部署与配置指南"
published: 2026-08-04
description: "本文详细记录了在Linux环境下部署OpenVPN服务端的完整流程，涵盖环境配置、PKI证书生成、客户端配置及服务运维排查。"
category: "服务部署"
tags:
  - "OpenVPN"
  - "VPN"
  - "证书体系"
  - "防火墙配置"
  - "系统运维"
---

# OpenVPN 服务端部署与配置指南

## 1. 系统环境准备

### 1.1 关闭 SELinux

```bash
setenforce 0
sed -i 's/SELINUX=enforcing/SELINUX=disabled/' /etc/selinux/config
```

### 1.2 配置防火墙

```bash
systemctl enable firewalld
systemctl start firewalld
firewall-cmd --zone=public --add-port=1194/udp --permanent
firewall-cmd --zone=public --add-masquerade --permanent
firewall-cmd --reload
```

### 1.3 开启内核转发

```bash
echo 'net.ipv4.ip_forward = 1' >> /etc/sysctl.conf
sysctl -p
```

### 1.4 安装依赖

```bash
yum install epel-release -y
yum install openvpn easy-rsa -y
```

## 2. 构建 PKI 证书体系

### 2.1 初始化工作目录

```bash
mkdir -p /etc/openvpn/pki
cp -rf /usr/share/easy-rsa/3/* /etc/openvpn/pki/
cd /etc/openvpn/pki
```

### 2.2 生成证书和密钥

```bash
# 初始化 PKI
./easyrsa init-pki

# 生成 CA 证书
./easyrsa build-ca nopass

# 生成 Server 证书
./easyrsa build-server-full server nopass

# 生成 Client 证书
./easyrsa build-client-full client1 nopass

# 生成 DH 密钥
./easyrsa gen-dh

# 生成 TLS Auth 密钥
openvpn --genkey --secret pki/ta.key
```

## 3. 配置 OpenVPN 服务端

### 3.1 创建 server.conf

文件路径：`/etc/openvpn/server.conf`

```conf
port 1194
proto udp
dev tun

ca /etc/openvpn/pki/pki/ca.crt
cert /etc/openvpn/pki/pki/issued/server.crt
key /etc/openvpn/pki/pki/private/server.key
dh /etc/openvpn/pki/pki/dh.pem

tls-auth /etc/openvpn/pki/pki/ta.key 0

server 10.8.0.0 255.255.255.0
ifconfig-pool-persist ipp.txt

push "redirect-gateway def1"
push "dhcp-option DNS 8.8.8.8"
push "dhcp-option DNS 1.1.1.1"

keepalive 10 120

cipher AES-256-GCM
auth SHA256
compress lz4-v2

user nobody
group nobody
persist-key
persist-tun

status /var/log/openvpn-status.log
log-append /var/log/openvpn.log
verb 3

plugin /usr/lib64/openvpn/plugins/openvpn-plugin-auth-pam.so login
duplicate-cn
```

## 4. 创建 VPN 登录用户

注意：不要使用 root 用户登录 VPN，需创建普通系统用户。

```bash
# 创建用户
useradd -s /sbin/nologin vpnuser

# 设置密码
echo "vpnuser:YourPassword123" | chpasswd

# 验证用户
id vpnuser
```

## 5. 生成客户端配置文件

### 5.1 生成内嵌证书的 ovpn 文件

以下脚本将生成 `/root/client.ovpn` 文件。请确保将 `SERVER_IP` 替换为实际公网 IP。

```bash
cd /etc/openvpn/pki

# 设置服务器公网 IP
SERVER_IP="1.2.3.4"

cat > /root/client.ovpn <<EOF
client
dev tun
proto udp
remote ${SERVER_IP} 1194
resolv-retry infinite
nobind
user nobody
group nobody
persist-key
persist-tun

<ca>
$(cat pki/ca.crt)
</ca>

<cert>
$(cat pki/issued/client1.crt)
</cert>

<key>
$(cat pki/private/client1.key)
</key>

<tls-auth>
$(cat pki/ta.key)
</tls-auth>
key-direction 1

cipher AES-256-GCM
auth SHA256
compress lz4-v2

auth-user-pass
verb 3
EOF

chmod 600 /root/client.ovpn
echo "配置文件已生成：/root/client.ovpn"
```

## 6. 启动服务与验证

### 6.1 启动 OpenVPN

```bash
systemctl start openvpn@server
systemctl enable openvpn@server
systemctl status openvpn@server
```

### 6.2 查看日志

```bash
tail -f /var/log/openvpn.log
```

### 6.3 客户端连接

1. 将 `/root/client.ovpn` 下载到客户端设备。
2. 使用 OpenVPN Connect 导入文件。
3. 连接时输入用户名：`vpnuser`，密码：`YourPassword123`。
4. 连接成功后检查 IP 是否为 `10.8.0.x`。

## 7. 运维与维护

### 7.1 查看在线用户

```bash
cat /var/log/openvpn-status.log
```

### 7.2 重启服务

```bash
systemctl restart openvpn@server
```

### 7.3 证书撤销

若客户端证书泄露，需撤销证书。

```bash
cd /etc/openvpn/pki
./easyrsa revoke client1
./easyrsa gen-crl

# 在 server.conf 中添加以下行
# crl-verify /etc/openvpn/pki/pki/crl.pem

# 重启服务
systemctl restart openvpn@server
```

### 7.4 日志轮转

配置 `/etc/logrotate.d/openvpn` 防止日志过大。

```conf
/var/log/openvpn.log {
    daily
    rotate 7
    missingok
    notifempty
    compress
    sharedscripts
    postrotate
        /bin/kill -HUP `cat /var/run/openvpn.server.pid 2> /dev/null` 2> /dev/null || true
    endscript
}
```

## 8. 故障排查要点

1. TLS Error：检查服务端和客户端的 `tls-auth`、`cipher`、`auth`、`compress` 配置是否完全一致。
2. Authentication failed：检查用户是否存在，密码是否正确，是否使用了 root 用户。
3. 连接超时：检查防火墙是否放行 `1194/udp`，是否开启 IP 转发。
4. 无法上网：检查防火墙 `masquerade` 是否开启，客户端是否推送了 `redirect-gateway`。
