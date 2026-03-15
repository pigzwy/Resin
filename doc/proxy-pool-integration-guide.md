# Resin 代理池对接指南

> **网关地址**：`https://resin.pigll.site`  
> **管理面板**：`https://resin.pigll.site/ui`  
> **管理密码**：`RESIN_ADMIN_TOKEN`（登录面板用）  
> **代理密码**：`RESIN_PROXY_TOKEN`（业务请求用，下文称 Token）

---

## 一、快速上手

### 1.1 最简调用（不过滤地区，每次换 IP）

```bash
curl "https://resin.pigll.site/{Token}/./https/api.ipify.org"
```

替换 `{Token}` 为你实际的 `RESIN_PROXY_TOKEN` 值即可。每次请求会自动选择最优节点，IP 随机轮换。

### 1.2 指定地区（需先创建 Platform）

```bash
# 走美国节点
curl "https://resin.pigll.site/{Token}/US/https/api.ipify.org"

# 走日本节点
curl "https://resin.pigll.site/{Token}/JP/https/api.ipify.org"
```

> ⚠️ 使用 `US`、`JP` 等名称前，**必须先在管理面板创建对应的 Platform**，见下方第三节。

---

## 二、核心概念

### 2.1 Token（代理密码）

`RESIN_PROXY_TOKEN` 的值，用于认证代理请求。**不是**登录管理面板的密码。

两个 Token 的区别：

| 环境变量 | 用途 | 在哪用 |
|:---|:---|:---|
| `RESIN_ADMIN_TOKEN` | 登录管理面板 | 浏览器打开面板时输入 |
| `RESIN_PROXY_TOKEN` | 代理请求认证 | 写在业务代码的 URL 里 |

### 2.2 Platform（代理池分组）

Platform 是一个**按规则过滤出来的节点子池**。每个 Platform 可以过滤：
- **地区**：只保留 `us`（美国）、`jp`（日本）等地区的节点
- **名称正则**：只保留名字匹配特定模式的节点
- **订阅来源**：只保留某个订阅里的节点

**不指定 Platform 时**，用 `.` 表示走全部节点的默认池。

### 2.3 Account（业务账号标识）

Account 是你**随便取的一个名字**，不需要提前注册。它的唯一作用是：

| | 不带 Account | 带 Account |
|:---|:---|:---|
| **IP 行为** | ✅ 每次请求随机换节点 | 🔒 绑定固定出口 IP |
| **绑定时长** | — | 默认 7 天（Platform 可调） |
| **适用场景** | 爬虫、数据采集 | 店铺管理、账号操作 |

**示例**：管理 3 个店铺，每个店铺需要固定 IP：

```
店铺1 → .../US.shop_01/https/目标网站   ← shop_01 绑定一个美国 IP
店铺2 → .../US.shop_02/https/目标网站   ← shop_02 绑定另一个美国 IP
店铺3 → .../JP.shop_03/https/目标网站   ← shop_03 绑定一个日本 IP
```

---

## 三、创建 Platform（分地区代理池）

打开 `https://resin.pigll.site/ui/platforms`：

1. 点击 **新建**
2. **名称**：填你想要的标识，如 `US`
3. **地区过滤**：填地区代码，如 `us`
4. 保存

常用地区代码参考：

| 代码 | 地区 | 代码 | 地区 |
|:---|:---|:---|:---|
| `us` | 美国 | `jp` | 日本 |
| `hk` | 香港 | `sg` | 新加坡 |
| `tw` | 台湾 | `kr` | 韩国 |
| `gb` | 英国 | `de` | 德国 |

> 一个 Platform 可以填多个地区，例如地区过滤填 `us` 和 `jp`，则同时包含美国和日本节点。

---

## 四、URL 格式详解

```
https://resin.pigll.site/{Token}/{Platform}.{Account}/{协议}/{目标地址及路径}
```

| 部分 | 必填 | 说明 | 示例 |
|:---|:---|:---|:---|
| Token | ✅ | `RESIN_PROXY_TOKEN` 的值 | `my-token` |
| Platform | ❌ | Platform 名称，不指定填 `.` | `US`、`JP`、`.` |
| Account | ❌ | 业务标识，不需要就不写 | `shop_01`、`tom` |
| 协议 | ✅ | 目标网站的协议 | `https`、`http` |
| 目标地址 | ✅ | 目标网站地址 + 路径 | `api.example.com/v1/data` |

### URL 示例速查

| URL | 行为 |
|:---|:---|
| `.../{Token}/./https/api.example.com` | 全部节点，轮换 IP |
| `.../{Token}/US/https/api.example.com` | 美国节点，轮换 IP |
| `.../{Token}/JP/https/api.example.com` | 日本节点，轮换 IP |
| `.../{Token}/US.tom/https/api.example.com` | 美国节点，tom 固定 IP |
| `.../{Token}/JP.shop_01/https/api.example.com` | 日本节点，shop_01 固定 IP |

---

## 五、各语言对接代码

### Python

```python
import requests

RESIN = "https://resin.pigll.site"
TOKEN = "my-token"  # 替换为你的 RESIN_PROXY_TOKEN


def proxy_request(url, platform=".", account=None):
    """
    通过 Resin 代理池发送请求
    
    Args:
        url:      目标地址，如 "https://api.ipify.org"
        platform: Platform 名称，"." 表示全部节点
        account:  业务账号，None 表示轮换 IP
    """
    scheme = "https" if url.startswith("https") else "http"
    target = url.split("://", 1)[1]
    
    identity = f"{platform}.{account}" if account else platform
    proxy_url = f"{RESIN}/{TOKEN}/{identity}/{scheme}/{target}"
    
    return requests.get(proxy_url)


# ── 示例 1：走美国节点，每次换 IP ──
resp = proxy_request("https://api.ipify.org", platform="US")
print(f"美国随机 IP: {resp.text}")

# ── 示例 2：走日本节点，shop_01 固定 IP ──
resp = proxy_request("https://api.ipify.org", platform="JP", account="shop_01")
print(f"shop_01 日本固定 IP: {resp.text}")

# ── 示例 3：全部节点，轮换 IP ──
resp = proxy_request("https://api.ipify.org")
print(f"随机 IP: {resp.text}")
```

### Java (OkHttp)

```java
public class ResinProxy {
    private static final String RESIN = "https://resin.pigll.site";
    private static final String TOKEN = "my-token";
    private final OkHttpClient client = new OkHttpClient();

    /**
     * 通过 Resin 代理池发送 GET 请求
     *
     * @param targetUrl 目标地址，如 "https://api.ipify.org"
     * @param platform  Platform 名称，"." 表示全部节点
     * @param account   业务账号，null 表示轮换 IP
     */
    public String proxyGet(String targetUrl, String platform, String account) throws IOException {
        String scheme = targetUrl.startsWith("https") ? "https" : "http";
        String target = targetUrl.split("://", 2)[1];
        String identity = (account != null) ? platform + "." + account : platform;
        
        String url = RESIN + "/" + TOKEN + "/" + identity + "/" + scheme + "/" + target;
        Request request = new Request.Builder().url(url).build();
        
        try (Response response = client.newCall(request).execute()) {
            return response.body().string();
        }
    }
}

// 使用
ResinProxy proxy = new ResinProxy();

// 走美国节点，轮换 IP
String ip1 = proxy.proxyGet("https://api.ipify.org", "US", null);

// 走日本节点，shop_01 固定 IP
String ip2 = proxy.proxyGet("https://api.ipify.org", "JP", "shop_01");
```

### Node.js (axios)

```javascript
const axios = require('axios');

const RESIN = 'https://resin.pigll.site';
const TOKEN = 'my-token';

/**
 * 通过 Resin 代理池发送请求
 * @param {string} targetUrl  - 目标地址
 * @param {string} platform   - Platform 名称，"." 表示全部
 * @param {string} [account]  - 业务账号，不传则轮换 IP
 */
async function proxyRequest(targetUrl, platform = '.', account = null) {
  const scheme = targetUrl.startsWith('https') ? 'https' : 'http';
  const target = targetUrl.split('://')[1];
  const identity = account ? `${platform}.${account}` : platform;
  
  const url = `${RESIN}/${TOKEN}/${identity}/${scheme}/${target}`;
  const { data } = await axios.get(url);
  return data;
}

// 走美国节点，轮换 IP
const ip1 = await proxyRequest('https://api.ipify.org', 'US');

// 走日本节点，shop_01 固定 IP
const ip2 = await proxyRequest('https://api.ipify.org', 'JP', 'shop_01');
```

### cURL

```bash
# 美国节点，轮换 IP
curl "https://resin.pigll.site/my-token/US/https/api.ipify.org"

# 日本节点，固定 IP
curl "https://resin.pigll.site/my-token/JP.shop_01/https/api.ipify.org"

# 全部节点，轮换 IP
curl "https://resin.pigll.site/my-token/./https/api.ipify.org"
```

---

## 六、高级用法

### 6.1 通过 Header 传递 Account（推荐生产环境）

使用 `X-Resin-Account` 请求头，代码更清晰，便于中间件统一处理：

```python
resp = requests.get(
    f"{RESIN}/{TOKEN}/US/https/api.example.com/v1/orders",
    headers={"X-Resin-Account": "shop_01"}
)
```

优先级：`X-Resin-Account` Header > URL 中的 Account > Header 提取规则

### 6.2 零侵入对接（Header 提取规则）

如果你的业务请求已经带有 `Authorization` 等头部，Resin 可以自动提取作为 Account：

1. 在 Platform 设置中，将**反向代理空账号行为**设为`提取指定请求头作为 Account`
2. 填入要提取的头部名称，如 `Authorization`

这样即使 URL 不带 Account，Resin 也能自动从请求头识别身份并绑定固定 IP。

### 6.3 调整 Sticky TTL

默认绑定时长为 **7 天**，可在 **Platform 设置** 中修改 `sticky_ttl` 字段。例如设为 `30m`（30 分钟）、`1h`（1 小时）等。

---

## 七、常见问题

| 问题 | 答案 |
|:---|:---|
| Platform 名称写错会怎样？ | 返回错误，请确保名称和面板中创建的完全一致 |
| Token 写错会怎样？ | 返回 401 认证失败 |
| 节点全部故障了？ | Resin 返回错误，建议监控告警 |
| 能同时用多个 Platform 吗？ | 每个请求只能指定一个 Platform |
| Account 需要提前注册吗？ | 不需要，URL 里写什么就是什么 |
| 固定 IP 的节点挂了？ | 自动切换到同 IP 的其他节点，没有则切到新节点 |
