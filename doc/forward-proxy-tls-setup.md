# Resin 正向代理 TLS (HTTPS Proxy) 部署指南

> 本文档记录了 Resin 正向代理在中国大陆网络环境下 CONNECT 隧道被阻断的问题分析与解决方案。

---

## 一、问题描述

使用 Resin 正向代理（HTTP CONNECT 隧道）访问被墙域名时，连接被立即重置：

```
curl -x http://proxy.pigll.site:2260 -U "us:token" https://chatgpt.com
> CONNECT chatgpt.com:443 HTTP/1.1
* Recv failure: Connection was reset  ❌
```

而相同配置访问未被墙的域名则正常：

```
curl -x http://proxy.pigll.site:2260 -U "us:token" https://api.ipify.org
< HTTP/1.1 200 Connection Established  ✅
```

---

## 二、问题排查过程

### 2.1 排除 DNS 污染

服务器位于新加坡，DNS 解析正常：

```bash
dig chatgpt.com +short
# 172.64.155.209  ← 正确的 Cloudflare IP
```

服务器直连 chatgpt.com 也没问题：

```bash
curl -sv https://chatgpt.com  # TLS 握手成功 ✅
```

### 2.2 排除 Resin 本身的问题

从服务器本地通过 Resin 正向代理 CONNECT chatgpt.com：

```bash
curl -x http://127.0.0.1:2260 -U "us:token" https://chatgpt.com
< HTTP/1.1 200 Connection Established  ✅  ← 本地完全正常
```

### 2.3 确认根因：GFW 对明文 CONNECT 请求的 DPI

Resin 正向代理端口（2260）使用 **明文 HTTP**。当客户端发送 `CONNECT chatgpt.com:443` 时，GFW 的深度包检测（DPI）能读取到目标域名，并发送 TCP RST 包阻断连接。

```
明文 HTTP 正向代理（被 GFW 拦截 ❌）：
客户端(中国) ──明文 HTTP── CONNECT chatgpt.com:443 ──→ GFW 检测到 ──→ RST

HTTPS 正向代理（不被拦截 ✅）：
客户端(中国) ──TLS 加密── CONNECT chatgpt.com:443 ──→ GFW 看不到 ──→ 放行
```

**关键证据**：
- 同一节点，本地走 SOCKS5（加密）可以访问 chatgpt.com ✅
- 同一节点，走 Resin 明文 CONNECT 不行 ❌
- 从服务器本地 `127.0.0.1` 走 Resin CONNECT 正常 ✅（不经过 GFW）
- 不在 GFW 封锁名单的域名（auth.openai.com、api.ipify.org）明文 CONNECT 也正常 ✅

---

## 三、解决方案：stunnel TLS 封装

在 Resin 正向代理端口（2260）前加一层 TLS，使用 stunnel 将 HTTPS:2261 → HTTP:2260。

### 3.1 架构

```
客户端 ──HTTPS:2261──→ stunnel（TLS 终止）──HTTP:2260──→ Resin 正向代理 ──节点──→ 目标站
                       ↑ GFW 只看到 TLS 流量                ↑ CONNECT 隧道透传
                       ↑ 看不到 CONNECT 目标域名             ↑ 客户端 TLS 指纹保留
```

### 3.2 前提条件

- Resin 服务已运行，正向代理监听 `127.0.0.1:2260`
- 域名 `proxy.pigll.site` 的 DNS 记录指向服务器 IP（**DNS Only，不开 Cloudflare 代理**）
- Caddy 已运行，用于自动管理 TLS 证书

### 3.3 部署步骤

**第 1 步：Caddy 管理 proxy.pigll.site 的证书**

在 Caddyfile 中添加：

```
proxy.pigll.site {
    respond "Resin Forward Proxy - connect to port 2261" 200
}
```

重载 Caddy：

```bash
systemctl reload caddy   # 或 caddy reload
```

Caddy 会自动从 Let's Encrypt 申请证书，存储在：

```
/var/lib/caddy/.local/share/caddy/certificates/acme-v02.api.letsencrypt.org-directory/proxy.pigll.site/
├── proxy.pigll.site.crt
├── proxy.pigll.site.key
└── proxy.pigll.site.json
```

**第 2 步：安装和配置 stunnel**

```bash
apt install -y stunnel4

cat > /etc/stunnel/resin.conf << 'EOF'
[resin-forward-proxy]
accept = 2261
connect = 127.0.0.1:2260
cert = /var/lib/caddy/.local/share/caddy/certificates/acme-v02.api.letsencrypt.org-directory/proxy.pigll.site/proxy.pigll.site.crt
key = /var/lib/caddy/.local/share/caddy/certificates/acme-v02.api.letsencrypt.org-directory/proxy.pigll.site/proxy.pigll.site.key
EOF

stunnel /etc/stunnel/resin.conf
```

**第 3 步：防火墙放行 2261 端口**

```bash
# 腾讯云安全组中添加入站规则：TCP 2261
```

### 3.4 验证

```bash
# 从客户端（中国大陆）测试
curl --proxy-insecure -x https://proxy.pigll.site:2261 -U "us:token" https://chatgpt.com
# HTTP/1.1 200 Connection Established ✅
```

---

## 四、客户端接入

### 正向代理地址

```
https://proxy.pigll.site:2261
```

### 认证格式

```
{Platform}.{Account}:{Token}
```

### 代码示例

**Python (curl_cffi，保留 TLS 指纹)**：

```python
import curl_cffi.requests as requests

resp = requests.get(
    "https://chatgpt.com/backend-api/checkout",
    proxy="https://us.my_account:{Token}@proxy.pigll.site:2261",
    impersonate="chrome",  # TLS 指纹完整保留 ✅
)
```

**Python (requests)**：

```python
import requests

proxies = {
    "http": "https://us.my_account:{Token}@proxy.pigll.site:2261",
    "https": "https://us.my_account:{Token}@proxy.pigll.site:2261",
}
resp = requests.get("https://api.ipify.org", proxies=proxies)
```

---

## 五、端口和域名总结

| 域名 | 端口 | 模式 | Cloudflare | 用途 |
|:---|:---|:---|:---|:---|
| `resin.pigll.site` | 443 | 反向代理（URL 改写） | ☁️ 已代理 | API 调用（不需 TLS 伪装） |
| `proxy.pigll.site` | 2260 | 正向代理（明文） | 仅 DNS | 服务器本地测试 |
| `proxy.pigll.site` | 2261 | 正向代理（TLS） | 仅 DNS | 生产使用（保留 TLS 指纹） |

---

## 六、注意事项

1. **proxy.pigll.site 不要开 Cloudflare 代理（小黄云）**：Cloudflare 不支持转发 CONNECT 请求，开了会导致正向代理不可用。

2. **证书续期**：Caddy 自动续期证书。但 stunnel 不会自动重载证书，需要定期重启 stunnel 或配置 cron 任务：

   ```bash
   # 每月重启 stunnel 以加载新证书
   echo "0 3 1 * * killall stunnel && stunnel /etc/stunnel/resin.conf" | crontab -
   ```

3. **stunnel 开机自启**：

   ```bash
   echo "stunnel /etc/stunnel/resin.conf" >> /etc/rc.local
   ```

4. **反向代理不丢 TLS 指纹的误区**：反向代理（URL 改写）会终止客户端 TLS 并重新建连，TLS 指纹会丢失。正向代理（CONNECT 隧道）是透传的，TLS 指纹保留。TLS 封装（stunnel）不影响隧道内的数据。
