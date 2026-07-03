# nanobot + 智谱 GLM Coding Plan 完整使用指南

> 基于 nanobot v0.2.2 在本机（macOS + 公司代理环境）从零跑通 GLM Coding Plan 的实战排查记录。
> 已端到端验证：CLI 对话 ✅、WebUI 浏览器对话 ✅（2026-07-03 实测 43K tokens / 4 次请求）。

---

## 一、速查（TL;DR）

让 GLM Coding Plan 在 nanobot 跑起来，只需把握 **三件事**：

1. **端点要对**：config 里 `providers.zhipu.apiBase` 必须指向 `https://open.bigmodel.cn/api/coding/paas/v4`（Coding Plan 专用，有订阅额度），**不是**默认的 `/api/paas/v4`（标准端点，会报"余额不足 429"）。
2. **必须走代理**：本机直连外网被封（`Connection reset by peer`），智谱要走公司代理 `http://10.200.86.85:8080`。
3. **要信任公司 CA**：公司代理做 HTTPS MITM，Python `certifi` 不信任公司 CA 会 `CERTIFICATE_VERIFY_FAILED`。导出 macOS 系统钥匙串（含公司 CA），用 `SSL_CERT_FILE` 指定。

一键启动（见第四节完整脚本）：
```bash
export SSL_CERT_FILE=/tmp/nanobot-cacert.pem
export REQUESTS_CA_BUNDLE=/tmp/nanobot-cacert.pem
export NO_PROXY=127.0.0.1,localhost          # 本地不走代理；智谱要走代理，故不在列表里
.venv/bin/nanobot gateway
```

### 非公司网络环境（无代理 / 无 MITM）—— 只需三步

如果你在家 / 普通直连网络（没有公司代理、没有 HTTPS MITM），**跳过所有代理与证书环境变量**，只需「config 配对 + 装好 + 启动」。本节是本指南其余部分（公司网络场景）的"精简版"。

**1. config**（`~/.nanobot/config.json`，和环境无关、永远要配）：
```json
{
  "agents": { "defaults": { "model": "glm-4.5-air", "provider": "auto" } },
  "providers": {
    "zhipu": {
      "apiKey": "你的智谱-key",
      "apiBase": "https://open.bigmodel.cn/api/coding/paas/v4"
    }
  },
  "channels": {
    "websocket": { "enabled": true, "port": 8765, "tokenIssueSecret": "nanobot", "websocketRequiresToken": true }
  }
}
```

**2. 安装**：
```bash
python3 -m venv .venv && .venv/bin/pip install -e .
cd webui && npm install && cd ..
```

**3. 启动**（干净的，无需 `SSL_CERT_FILE` / `NO_PROXY` / `HTTPS_PROXY`）：
```bash
.venv/bin/nanobot gateway &
cd webui && NANOBOT_API_URL=http://127.0.0.1:8765 npm run dev
# 浏览器开 http://127.0.0.1:5173 ，密码 nanobot
```

**环境对照（哪些是公司环境才需要的负担）**：

| 条件 | 普通环境 | 公司环境（本机） |
|---|---|---|
| 端点 `coding/paas/v4` | ✅ 必需 | ✅ 必需 |
| config apiKey/model | ✅ 必需 | ✅ 必需 |
| websocket channel（WebUI） | ✅ 必需 | ✅ 必需 |
| `HTTPS_PROXY` | ❌ 不需要（直连可达） | ✅ 必须走代理 |
| `NO_PROXY` | ❌ 不需要 | ✅ 只放 127.0.0.1 |
| `SSL_CERT_FILE`（系统证书） | ❌ 不需要（certifi 能验官方证书） | ✅ 必须（代理 MITM） |
| 导出 `/tmp/nanobot-cacert.pem` | ❌ 不需要 | ✅ 必须 |

> 三条环境变量（`SSL_CERT_FILE` / `HTTPS_PROXY` / `NO_PROXY`）全是公司代理 MITM 惹出来的额外负担，换个网络就全消失。**唯一不分环境都要注意**：① `coding/paas/v4` 端点必须用（否则 429 余额不足）；② GLM 默认带 reasoning 思考链，首条回复十几秒属正常。

---

## 二、Coding Plan 接口知识

智谱（open.bigmodel.cn）有两套 OpenAI 兼容端点：

| 端点 | URL | 计费 | 本机实测 |
|---|---|---|---|
| **标准 Paas** | `https://open.bigmodel.cn/api/paas/v4` | 按量计费 | **HTTP 429「余额不足」** |
| **Coding Plan** | `https://open.bigmodel.cn/api/coding/paas/v4` | 订阅制（有额度） | **HTTP 200 成功** ✅ |

- 认证方式相同：`Authorization: Bearer <api_key>`
- 请求/响应格式：OpenAI Chat Completions 兼容（`/chat/completions`）
- 模型名不变：`glm-4.5-air`（智谱会路由到 `glm-4.7` 等实际模型，正常）
- **同一把 api_key 两个端点都能用**，区别只在额度池。Coding Plan 订阅有效期内调 `/coding/paas/v4` 不扣余额。

nanobot 的 zhipu provider 默认用标准 `paas/v4`（见 `nanobot/providers/registry.py` 的 `default_api_base`），所以**必须在 config 显式覆盖为 coding/paas/v4**。

---

## 三、config 配置（`~/.nanobot/config.json`）

关键三段：默认模型、zhipu 凭据、（可选）WebUI 通道。

```json
{
  "agents": {
    "defaults": {
      "model": "glm-4.5-air",
      "provider": "auto",
      "maxTokens": 8192,
      "temperature": 0.7
    }
  },
  "providers": {
    "zhipu": {
      "apiKey": "你的智谱-api-key",
      "apiBase": "https://open.bigmodel.cn/api/coding/paas/v4"
    }
  },
  "channels": {
    "websocket": {
      "enabled": true,
      "host": "127.0.0.1",
      "port": 8765,
      "tokenIssueSecret": "nanobot",
      "websocketRequiresToken": true
    }
  }
}
```

要点：
- `provider: "auto"` + 模型名含 `glm` → nanobot 自动路由到 `zhipu` provider（关键字 zhipu/glm/zai）。
- `providers.zhipu.apiBase` 是**决定性的一行**。nanobot `Config.get_api_base()`（`nanobot/config/schema.py:572`）优先读它，只有为空才回退到默认 `paas/v4`。
- `channels.websocket` 是 WebUI 必需（见第六节），纯 CLI 可不加。

验证 nanobot 实际解析出的 provider 和 apiBase：
```bash
.venv/bin/python -c "
from nanobot.config.loader import load_config
c = load_config()
m = c.agents.defaults.model
print('provider:', c.get_provider_name(m))
print('api_base:', c.get_api_base(m))   # 应输出 .../coding/paas/v4
"
```

---

## 四、网络：代理 + 公司 CA（最大坑）

### 4.1 现象与根因
- 本机环境有 `HTTPS_PROXY=http://10.200.86.85:8080`（公司代理）。
- 该代理对 HTTPS 做**中间人（MITM）**，用公司自签 CA 替换目标网站证书。
- `curl` 用 macOS 系统钥匙串（含公司 CA）→ 验证通过 → 成功。
- Python `httpx`/`openai` 默认用 `certifi` 证书库（**不含公司 CA**）→ `SSL: CERTIFICATE_VERIFY_FAILED`。
- 若绕过代理直连 → `Connection reset by peer`（直连外网被封）。

### 4.2 修复：导出系统证书，让 Python 信任公司 CA
一次性导出 macOS 系统钥匙串（含公司 MITM CA）到一个 PEM 文件：

```bash
OUT=/tmp/nanobot-cacert.pem
: > "$OUT"
security find-certificate -a -p /System/Library/Keychains/SystemRootCertificates.keychain >> "$OUT"
security find-certificate -a -p /Library/Keychains/System.keychain >> "$OUT"
security find-certificate -a -p "$HOME/Library/Keychains/login.keychain-db" >> "$OUT" 2>/dev/null
# 检查：应导出 150+ 张证书
grep -c "BEGIN CERTIFICATE" "$OUT"
```

> 该文件含公司内网 CA，请勿外发。公司 CA 更新后需重新导出。

### 4.3 代理规则
- **必须走代理**：`open.bigmodel.cn`（智谱）。
- **不走代理**：`127.0.0.1`、`localhost`（vite ↔ gateway 本地通信）。
- 所以 `NO_PROXY=127.0.0.1,localhost`，**不要**把 `open.bigmodel.cn` 放进 NO_PROXY（否则直连被封）。

---

## 五、完整启动流程（从源码运行 nanobot v0.2.2）

前置：Python 3.11+、node（bun 非必需，npm 即可）。

```bash
cd "/Users/yjcao/Documents/开发项目/yjcao/nanobot-main 2"

# 1. 装源码（venv 隔离，不污染全局）
python3 -m venv .venv
.venv/bin/python -m pip install --upgrade pip
.venv/bin/pip install -e .
.venv/bin/nanobot --version    # 应为 v0.2.2

# 2. 装前端依赖（npm 代替 bun；~/.npm 若有 root 权限残留，用独立 cache 绕过）
cd webui && npm install --cache /tmp/npm-cache-nanobot && cd ..

# 3. 导出系统证书（见 4.2，若 /tmp/nanobot-cacert.pem 已存在可跳过）

# 4. 启动 gateway（带正确的代理 + 证书环境）
export SSL_CERT_FILE=/tmp/nanobot-cacert.pem
export REQUESTS_CA_BUNDLE=/tmp/nanobot-cacert.pem
export NO_PROXY=127.0.0.1,localhost
export no_proxy=127.0.0.1,localhost
.venv/bin/nanobot gateway        # 后台常驻；日志出现 ws://127.0.0.1:8765/ 即成功
```

纯 CLI 单条对话（不启 WebUI 时验证 LLM 最快）：
```bash
export SSL_CERT_FILE=/tmp/nanobot-cacert.pem NO_PROXY=127.0.0.1,localhost
.venv/bin/nanobot agent -m "用一句话介绍你自己" --no-markdown </dev/null
```

---

## 六、WebUI（v0.2.2 新增，老版本 v0.1.4 没有）

### 6.1 启用 websocket channel
config 的 `channels.websocket`（见第三节）必须有。否则 gateway 日志报 `No channels enabled`，且只挂 `/health`，所有 `/webui/*` 返回 404。

### 6.2 端口模型
- `18790`：gateway 主端口（health、对外标识）。
- `8765`：websocket channel 端口（**WebUI 的 HTTP bootstrap + WS 连接都在这里**）。
- `5173`：vite dev server（开发态，浏览器访问入口）。

### 6.3 开发态启动（热更新）
```bash
# gateway 先起好（见第五节）
cd webui
NANOBOT_API_URL="http://127.0.0.1:8765" npm run dev   # proxy 要指 8765，不是 18790
# 浏览器开 http://127.0.0.1:5173 ，登录密码 = config 里的 tokenIssueSecret（本例 "nanobot"）
```

### 6.4 生产态（单进程，无需 vite）
```bash
cd webui && npm run build          # 产物输出到 nanobot/web/dist
.venv/bin/nanobot gateway          # gateway 直接服务静态 dist
# 浏览器开 http://127.0.0.1:8765
```

### 6.5 认证流程（理解 401）
浏览器 → `GET /webui/bootstrap`（带 `Authorization: Bearer <tokenIssueSecret>` 或 `X-Nanobot-Auth`）→ gateway 校验 secret → 返回临时 `{token, ws_url, expires_in:300}` → 浏览器用 token 连 `ws://127.0.0.1:8765/`。所以 `tokenIssueSecret` 就是 WebUI 登录密码。

---

## 七、验证与排查命令

```bash
# A. curl 直连（应失败：reset）—— 证明必须走代理
curl --noproxy '*' -X POST https://open.bigmodel.cn/api/coding/paas/v4/chat/completions \
  -H "Authorization: Bearer $KEY" -H "Content-Type: application/json" \
  -d '{"model":"glm-4.5-air","messages":[{"role":"user","content":"hi"}],"max_tokens":8}'

# B. curl 走代理（应 200）—— 证明代理 + Coding Plan 端点可用
curl -x http://10.200.86.85:8080 -X POST https://open.bigmodel.cn/api/coding/paas/v4/chat/completions \
  -H "Authorization: Bearer $KEY" -H "Content-Type: application/json" \
  -d '{"model":"glm-4.5-air","messages":[{"role":"user","content":"hi"}],"max_tokens":8}'

# C. Python httpx 走代理 + 系统证书（应 200）—— 复现 nanobot 真实调用路径
.venv/bin/python -c "
import httpx
r = httpx.post('https://open.bigmodel.cn/api/coding/paas/v4/chat/completions',
  headers={'Authorization':'Bearer $KEY','Content-Type':'application/json'},
  json={'model':'glm-4.5-air','messages':[{'role':'user','content':'hi'}],'max_tokens':8},
  verify='/tmp/nanobot-cacert.pem', timeout=120)
print(r.status_code, r.text[:120])
"

# D. gateway 健康检查
curl http://127.0.0.1:18790/health          # {"status":"ok"}

# E. WebUI bootstrap（带 secret 应返回 token）
curl -H "X-Nanobot-Auth: nanobot" http://127.0.0.1:8765/webui/bootstrap
```

---

## 八、踩坑对照表

| 现象 | 根因 | 解决 |
|---|---|---|
| `HTTP 429 余额不足` | 用了标准 `paas/v4` 端点 | apiBase 改 `coding/paas/v4` |
| `Connection error` / `Connection reset by peer` | 直连外网被封 | 走代理，别把智谱放进 NO_PROXY |
| `SSL: CERTIFICATE_VERIFY_FAILED` | 公司代理 MITM，certifi 不信公司 CA | `SSL_CERT_FILE=/tmp/nanobot-cacert.pem`（导出的系统证书） |
| `ReadTimeout` | GLM 默认带 reasoning 思考链，响应慢 | timeout 加大到 90~120s |
| WebUI `bootstrap failed: HTTP 404` | websocket channel 未启用 | config 加 `channels.websocket.enabled=true` |
| WebUI `HTTP 401` | bootstrap 需要 secret | 带 `tokenIssueSecret`（= WebUI 登录密码） |
| `No channels enabled` | config 无 websocket 段 | 同上 |
| vite `EACCES ~/.npm` | 旧 sudo 跑过 npm，缓存有 root 文件 | `npm install --cache /tmp/xxx` 绕过，或 `sudo chown` |
| 老项目 v0.1.4「能跑」但新项目不行 | 老版本无内置 WebUI（v0.2.0 才内嵌），所谓"能跑"是 CLI 对话 | 新项目 WebUI 需额外启用 websocket channel |

---

## 九、关键文件位置（本机）

- 项目源码：`/Users/yjcao/Documents/开发项目/yjcao/nanobot-main 2`（v0.2.2）
- venv：`<项目>/.venv`
- 前端：`<项目>/webui`（vite dev → 5173）
- nanobot 配置：`~/.nanobot/config.json`（已加 websocket 段；原备份 `config.json.bak.1783070636`）
- 系统证书导出：`/tmp/nanobot-cacert.pem`（重启后若丢失需重新导出）
- gateway 日志：后台任务输出（`nanobot gateway` 的 stdout）
- provider 端点解析：`nanobot/config/schema.py` 的 `get_api_base` / `_match_provider`
- zhipu provider 注册：`nanobot/providers/registry.py`（`default_api_base` = 标准 paas/v4，需 config 覆盖）
