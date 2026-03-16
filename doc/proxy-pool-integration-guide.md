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

## 三、创建 Platform（节点筛选器）

Platform 就是你的**节点筛选器**。通过不同的过滤规则，从所有节点中筛选出你想要的子集。

打开 `https://resin.pigll.site/ui/platforms` → 点击 **新建**。

### 3.1 按地区过滤（最常用）

只保留特定地区的节点。

| Platform 名称 | 地区过滤 | 效果 |
|:---|:---|:---|
| `US` | `us` | 只走美国节点 |
| `JP` | `jp` | 只走日本节点 |
| `US_JP` | `us`, `jp` | 美国和日本节点混用 |

常用地区代码：

| 代码 | 地区 | 代码 | 地区 |
|:---|:---|:---|:---|
| `us` | 美国 | `jp` | 日本 |
| `hk` | 香港 | `sg` | 新加坡 |
| `tw` | 台湾 | `kr` | 韩国 |
| `gb` | 英国 | `de` | 德国 |

### 3.2 按名称正则过滤（指定订阅/节点）

通过正则表达式匹配节点名称，可以精确控制使用哪些节点。

| Platform 名称 | 名称正则 | 效果 |
|:---|:---|:---|
| `fafa专线` | `^fafa/` | 只走 fafa 订阅的节点 |
| `家宽专线` | `webshare家宽` | 只走 webshare 家宽节点 |
| `GPT专线` | `socks-163.123.202.163` | **只走这一个节点** |

### 3.3 指定具体节点（固定节点）

如果你想让某个业务**只走某一个特定节点**，方法是：

1. 新建 Platform，名称填 `GPT专线`（或任何名字）
2. **名称正则**填该节点名称中的唯一关键词，如 `socks-163.123.202.163`
3. 保存。此时这个 Platform 里只有那一个节点

然后请求里使用这个 Platform：

```bash
# GPT 业务只走 163.123.202.163 这一个节点
curl "https://resin.pigll.site/{Token}/GPT专线/https/api.openai.com/v1/chat/completions"
```

> 💡 **核心思路**：用 Platform 的正则过滤缩小范围，范围缩到只剩一个节点 = 指定节点。

### 筛选方式总结

| 你想要 | 怎么建 Platform |
|:---|:---|
| 只走美国节点 | 地区过滤填 `us` |
| 只走某个订阅的节点 | 名称正则填订阅前缀，如 `^fafa/` |
| 只走某一个具体节点 | 名称正则填那个节点的关键词 |
| 美国 + 日本混用 | 地区过滤填 `us` 和 `jp` |

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

## 六、实战场景

### 6.1 GPT / OpenAI 固定代理

让所有 OpenAI API 请求走同一个美国 IP，避免频繁切换 IP 导致封号：

```python
from openai import OpenAI

# 将 base_url 指向 Resin，Account 设为 "gpt" 绑定固定 IP
client = OpenAI(
    api_key="sk-xxx",
    base_url="https://resin.pigll.site/my-token/US.gpt/https/api.openai.com/v1"
)

# 所有请求都会走同一个美国出口 IP（7 天内不变）
resp = client.chat.completions.create(
    model="gpt-4",
    messages=[{"role": "user", "content": "Hello"}]
)
```

> 💡 Account 名称随意取，`gpt`、`openai_main`、`chatbot_prod` 都可以。
> 不同 Account 会绑定不同的 IP，可以用来隔离不同业务。

### 6.2 GPT 指定固定节点

如果你不想让 Resin 自动选节点，而是**指定某个具体节点**给 GPT 用：

**第 1 步**：在面板创建一个 Platform，名称正则过滤填节点关键词（如 `socks-163.123.202.163`），使该 Platform 只包含那一个节点。

**第 2 步**：请求里使用该 Platform：

```python
from openai import OpenAI

# GPT专线 这个 Platform 只包含 163.123.202.163 这一个节点
client = OpenAI(
    api_key="sk-xxx",
    base_url="https://resin.pigll.site/my-token/GPT专线/https/api.openai.com/v1"
)
```

> 💡 详见第三节「3.3 指定具体节点」的创建步骤。

### 6.3 多店铺 / 多账号管理

每个店铺绑定独立的固定 IP：

```python
# 店铺 A → 美国固定 IP
resp_a = proxy_request("https://api.shop.com/orders", platform="US", account="shop_a")

# 店铺 B → 美国固定 IP（和 A 不同）
resp_b = proxy_request("https://api.shop.com/orders", platform="US", account="shop_b")

# 店铺 C → 日本固定 IP
resp_c = proxy_request("https://api.shop.com/orders", platform="JP", account="shop_c")
```

### 6.4 爬虫 / 数据采集（轮换 IP）

不带 Account，每次请求自动换 IP，降低被封风险：

```python
for page in range(100):
    # 每次请求走不同的 IP
    resp = proxy_request(f"https://target-site.com/page/{page}", platform="US")
```

---

## 七、高级用法

### 7.1 通过 Header 传递 Account（推荐生产环境）

使用 `X-Resin-Account` 请求头，代码更清晰，便于中间件统一处理：

```python
resp = requests.get(
    f"{RESIN}/{TOKEN}/US/https/api.example.com/v1/orders",
    headers={"X-Resin-Account": "shop_01"}
)
```

优先级：`X-Resin-Account` Header > URL 中的 Account > Header 提取规则

### 7.2 零侵入对接（Header 提取规则）

如果你的业务请求已经带有 `Authorization` 等头部，Resin 可以自动提取作为 Account：

1. 在 Platform 设置中，将**反向代理空账号行为**设为`提取指定请求头作为 Account`
2. 填入要提取的头部名称，如 `Authorization`

这样即使 URL 不带 Account，Resin 也能自动从请求头识别身份并绑定固定 IP。

### 7.3 调整 Sticky TTL

默认绑定时长为 **7 天**，可在 **Platform 设置** 中修改 `sticky_ttl` 字段。例如设为 `30m`（30 分钟）、`1h`（1 小时）等。

---

## 八、快速验证

部署完成后，可通过以下命令快速验证 Resin 是否正常工作。将 `{Token}` 替换为你的 `RESIN_PROXY_TOKEN`。

### 8.1 验证基本连通性

```bash
# 通过 Default 平台请求，返回出口 IP 即为成功
curl "https://resin.pigll.site/{Token}/Default/https/api.ipify.org"
# 预期输出: 103.197.71.113（一个代理节点的出口 IP）
```

### 8.2 验证 IP 轮换（不带 Account）

连续请求两次，观察 IP 是否变化：

```bash
curl "https://resin.pigll.site/{Token}/Default/https/api.ipify.org"
# 输出: 103.197.71.113

curl "https://resin.pigll.site/{Token}/Default/https/api.ipify.org"
# 输出: 103.62.49.154（IP 发生变化 ✅）
```

### 8.3 验证粘性代理（带 Account）

连续请求两次，使用相同的 Account，观察 IP 是否保持一致：

```bash
curl "https://resin.pigll.site/{Token}/Default.test_user/https/api.ipify.org"
# 输出: 103.62.49.178

curl "https://resin.pigll.site/{Token}/Default.test_user/https/api.ipify.org"
# 输出: 103.62.49.178（IP 保持不变 ✅）
```

### 8.4 验证指定 Platform（以 us 为例）

```bash
# 轮换 IP
curl "https://resin.pigll.site/{Token}/us/https/api.ipify.org"
# 输出: 23.184.88.83（美国 IP ✅）

curl "https://resin.pigll.site/{Token}/us/https/api.ipify.org"
# 输出: 47.147.26.75（IP 变化 ✅）

# 粘性代理
curl "https://resin.pigll.site/{Token}/us.test_user/https/api.ipify.org"
# 输出: 209.141.45.134

curl "https://resin.pigll.site/{Token}/us.test_user/https/api.ipify.org"
# 输出: 209.141.45.134（IP 保持不变 ✅）
```

### 8.5 验证结果速查

| 测试场景 | 预期行为 | 判断标准 |
|:---|:---|:---|
| Default 平台，无 Account | IP 轮换 | 多次请求返回不同 IP |
| Default 平台，带 Account | IP 固定 | 多次请求返回相同 IP |
| us 平台，无 Account | 美国 IP 轮换 | 返回美国 IP，多次请求 IP 不同 |
| us 平台，带 Account | 美国 IP 固定 | 返回美国 IP，多次请求 IP 相同 |
| Platform 名称不存在 | 返回错误 | 返回 `Platform not found` |
| Token 错误 | 认证失败 | 返回 401 |

> ⚠️ **注意**：Platform 名称**区分大小写**。例如面板中创建的是 `us`，请求中就必须写 `us`，写 `US` 会返回 `Platform not found`。

---

## 九、常见问题

| 问题 | 答案 |
|:---|:---|
| Platform 名称写错会怎样？ | 返回错误，请确保名称和面板中创建的完全一致 |
| Token 写错会怎样？ | 返回 401 认证失败 |
| 节点全部故障了？ | Resin 返回错误，建议监控告警 |
| 能同时用多个 Platform 吗？ | 每个请求只能指定一个 Platform |
| Account 需要提前注册吗？ | 不需要，URL 里写什么就是什么 |
| 固定 IP 的节点挂了？ | 自动切换到同 IP 的其他节点，没有则切到新节点 |
