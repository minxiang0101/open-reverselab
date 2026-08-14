# ccus.ink 临时邮箱操作手册（创建 / 收信 / 接口）

基于当前线上部署 `dreamhunter2333/cloudflare_temp_email` **v1.10.0** 实机验证整理。  
密码、URL、域名以本手册为准（和本地 `wrangler.toml` 可能不完全一致，以 **live `/open_api/settings`** 为准）。

---

## 1. 入口与账号

| 项 | 值 |
|---|---|
| **Worker API** | `https://temp-email-api.ccus.ink` |
| **Web 前端** | `https://cloudflare-temp-email-6bh.pages.dev` |
| **自定义前端域名** | `https://temp.ccus.ink`（若 SSL 异常，用 pages.dev） |
| **站点访问密码 `PASSWORDS`** | 当前 live：`needAuth=false`（公开页可不带 `x-custom-auth`） |
| **管理员密码 `ADMIN_PASSWORDS`** | `tzd_7792526` |
| **当前 live 可用域名** | `aqrap.uk` / `ytouch.me` / `dsnice.com` / `jabhat.cn` |
| **随机二级子域名** | 上述 4 个域名均在 `randomSubdomainDomains` 中 |
| **旧入口** | `https://ccus.ink` 已不可用，不要再用 |

> 说明：本地仓库 `wrangler.toml` 里还写着 `ccus.ink` 等 7 域，但 **线上 settings 当前只暴露 4 域**。对接脚本请读 live settings，不要硬编码旧列表。

---

## 2. 鉴权三层（必读）

| 层级 | 作用路径 | Header | 校验来源 |
|---|---|---|---|
| 全局门 | 除 `/open_api/`、`/telegram/` 外 | `x-custom-auth` | `PASSWORDS`（未配置则跳过；live 目前可省略） |
| 管理门 | `/admin/*` | `x-admin-auth` | `ADMIN_PASSWORDS`（必填） |
| 邮箱 JWT | `/api/*` | `Authorization: Bearer <jwt>` | 创建地址时返回的 `jwt` |

**外部脚本推荐写法：两个 admin 头都带上**

```http
x-custom-auth: tzd_7792526
x-admin-auth: tzd_7792526
```

常见中文报错：

- `您需要提供管理员密码才能访问此页面` → 缺 `x-admin-auth`
- `无效的邮箱地址凭据` → 打了 `/api/*` 但没 JWT / JWT 错
- `无效的 offset 参数` → `/admin/mails` 或 `/api/mails` 缺 `offset`
- `无效的域名` → `domain` 不在 worker 的 `DOMAINS` 里

**禁止** 用 `curl_cffi` impersonate 打这个 API（CF WAF Error 1010）。用普通 `requests` / `curl`。

---

## 3. 网页操作（人用）

1. 打开 `https://cloudflare-temp-email-6bh.pages.dev`
2. 选域名 → 填名字（或随机）→ 创建地址
3. 复制完整邮箱，去注册站填这个邮箱
4. 页面收件箱刷新，点开邮件看验证码
5. 需要管理后台时，用管理员密码登录 admin（同 `tzd_7792526`）

人用足够；批量 / 自动化走第 4 节 API。

---

## 4. API 操作手册

### 4.1 探活 / 拉域名池（无需鉴权）

```bash
curl -sS "https://temp-email-api.ccus.ink/open_api/settings"
```

关注字段：

- `version`
- `domains` / `defaultDomains` / `randomSubdomainDomains`
- `needAuth`
- `enableUserCreateEmail`

### 4.2 创建邮箱（Admin，推荐）

`POST /admin/new_address`

```bash
curl -sS -X POST "https://temp-email-api.ccus.ink/admin/new_address" \
  -H "x-custom-auth: tzd_7792526" \
  -H "x-admin-auth: tzd_7792526" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "manualtest",
    "domain": "dsnice.com",
    "enablePrefix": true,
    "enableRandomSubdomain": true
  }'
```

**Body 字段**

| 字段 | 类型 | 说明 |
|---|---|---|
| `name` | string | 本地部分前缀 |
| `domain` | string | 必须在 live `domains` 内 |
| `enablePrefix` | bool | true 时会加随机后缀，减少撞名 |
| `enableRandomSubdomain` | bool | true → 二级子域，如 `name@xxxxxxxx.dsnice.com` |

**成功响应示例（实机）**

```json
{
  "jwt": "<HS256 JWT，保存！>",
  "address": "manualtest@e5i3cpn5.dsnice.com",
  "password": null,
  "address_id": 216
}
```

**地址形态**

| 参数 | 结果形态 | 含义 |
|---|---|---|
| `enableRandomSubdomain=false` | `prefix@dsnice.com` | 一级 |
| `enableRandomSubdomain=true` | `prefix@xxxxxxxx.dsnice.com` | 二级随机子域（推荐批量） |

> 若你期望二级，却拿到扁平 `prefix@domain`，且没 JWT —— 多半是创建失败后本地兜底生成的假地址，OTP 永远收不到。

### 4.3 拉邮件 — 方式 A：Admin（按地址）

`GET /admin/mails?address=<完整邮箱>&limit=25&offset=0`

```bash
ADDR='manualtest@e5i3cpn5.dsnice.com'

curl -sS "https://temp-email-api.ccus.ink/admin/mails?address=${ADDR}&limit=25&offset=0" \
  -H "x-custom-auth: tzd_7792526" \
  -H "x-admin-auth: tzd_7792526"
```

- `address` 必填  
- `offset` 必填（没有会报「无效的 offset 参数」）  
- `limit` 建议 1–100  

空箱：`{"results":[],"count":0}`

每条大致字段：`id` / `message_id` / `address` 或 `to_address` / `from` 或 `from_address` / `subject`（可能 RFC2047） / `raw_text` / `created_at` / `source`

### 4.4 拉邮件 — 方式 B：JWT（推荐给单邮箱自动化）

创建时返回的 `jwt` 已绑定该邮箱，无需再传 address。

`GET /api/mails?limit=25&offset=0`

```bash
JWT='<创建时返回的 jwt>'

curl -sS "https://temp-email-api.ccus.ink/api/mails?limit=25&offset=0" \
  -H "Authorization: Bearer ${JWT}"
```

注意：

- `/api/mails` 条目常见只有 `raw` + `address` + `created_at`，`subject`/`from` 可能为空 → 从 `raw` RFC822 解析
- 无 JWT：`401 无效的邮箱地址凭据`

### 4.5 其它常用 Admin 接口

| 方法 | 路径 | 用途 |
|---|---|---|
| GET | `/admin/address?limit=N&offset=0` | 列出地址 |
| DELETE | `/admin/delete_address/:id` | 删地址 |
| DELETE | `/admin/clear_inbox/:id` | 清空收件箱 |
| DELETE | `/admin/mails/:id` | 删单封 |
| GET | `/admin/mails_unknow` | 未归属邮件 |
| GET | `/admin/statistics` | 统计 |
| POST | `/admin/send_mail` | 管理端发信（需开启发信） |

都要带 `x-admin-auth`（建议同时带 `x-custom-auth`）。

### 4.6 错误路径（不要用）

以下在本 worker **不存在 / 会失败**，第三方旧脚本常踩：

- `/api/mailboxes`、`/api/emails`
- `/admin/all`、`/admin/emails`
- 只带 `Authorization: Bearer` 却打 `/admin/*`
- 只带 `X-Admin-Token` 不带 `x-admin-auth`

---

## 5. Python 最小可运行示例

```python
import re
import time
import requests
from email import policy
from email.parser import BytesParser
from email.header import decode_header, make_header

API = "https://temp-email-api.ccus.ink"
ADMIN = "tzd_7792526"
HEADERS = {
    "x-custom-auth": ADMIN,
    "x-admin-auth": ADMIN,
    "Content-Type": "application/json",
}

def create_mailbox(name="bot", domain="dsnice.com", random_sub=True):
    r = requests.post(
        f"{API}/admin/new_address",
        headers=HEADERS,
        json={
            "name": name,
            "domain": domain,
            "enablePrefix": True,
            "enableRandomSubdomain": random_sub,
        },
        timeout=30,
    )
    r.raise_for_status()
    data = r.json()
    assert data.get("address") and data.get("jwt"), data
    return data["address"], data["jwt"], data.get("address_id")

def fetch_by_admin(address, limit=25):
    r = requests.get(
        f"{API}/admin/mails",
        headers=HEADERS,
        params={"address": address, "limit": limit, "offset": 0},
        timeout=30,
    )
    r.raise_for_status()
    return r.json().get("results") or []

def fetch_by_jwt(jwt, limit=25):
    r = requests.get(
        f"{API}/api/mails",
        headers={"Authorization": f"Bearer {jwt}"},
        params={"limit": limit, "offset": 0},
        timeout=30,
    )
    r.raise_for_status()
    return r.json().get("results") or []

def decode_rfc2047(s: str) -> str:
    if not s:
        return ""
    try:
        return str(make_header(decode_header(s)))
    except Exception:
        return s

def parse_raw(raw: str):
    raw = raw or ""
    msg = BytesParser(policy=policy.default).parsebytes(raw.encode("utf-8", "replace"))
    subject = decode_rfc2047(msg.get("Subject", ""))
    from_ = msg.get("From", "")
    # 纯文本优先
    body = ""
    if msg.is_multipart():
        for part in msg.walk():
            if part.get_content_type() == "text/plain":
                body = part.get_content()
                break
        if not body:
            for part in msg.walk():
                if part.get_content_type() == "text/html":
                    body = part.get_content()
                    break
    else:
        body = msg.get_content()
    return subject, from_, body or raw

OTP_RE = re.compile(r"(?<!\d)(\d{6})(?!\d)")

def extract_otp(text: str):
    m = OTP_RE.search(text or "")
    return m.group(1) if m else None

def poll_otp(address, jwt, timeout=180, interval=3):
    deadline = time.time() + timeout
    while time.time() < deadline:
        # 优先 JWT；失败再 admin
        try:
            items = fetch_by_jwt(jwt)
        except Exception:
            items = fetch_by_admin(address)
        for it in items:
            raw = it.get("raw") or it.get("raw_text") or it.get("body") or ""
            subject = it.get("subject") or ""
            if not subject and raw:
                subject, _, body = parse_raw(raw)
            else:
                _, _, body = parse_raw(raw) if raw else ("", "", "")
            blob = f"{subject}\n{body}\n{raw}"
            otp = extract_otp(blob)
            if otp:
                return otp, it
        time.sleep(interval)
    raise TimeoutError("OTP timeout")

if __name__ == "__main__":
    addr, jwt, aid = create_mailbox(name="demo", domain="dsnice.com", random_sub=True)
    print("address:", addr)
    print("address_id:", aid)
    print("jwt saved length:", len(jwt))
    # 把 addr 填到目标站 → 再：
    # otp, mail = poll_otp(addr, jwt)
    # print("otp:", otp)
```

---

## 6. 推荐集成流程（注册机 / 脚本）

1. `GET /open_api/settings` 确认域名池  
2. 从 `domains` 随机选一个（或固定 `dsnice.com`）  
3. `POST /admin/new_address`（`enableRandomSubdomain=true`）  
4. **立刻保存** `address` + `jwt` + `address_id`  
5. 把 `address` 交给目标站  
6. 轮询：  
   - 首选 `GET /api/mails` + Bearer JWT  
   - 兜底 `GET /admin/mails?address=...&offset=0`  
7. 从 `raw` / `raw_text` 解 RFC822 + RFC2047，提 6 位 OTP  
8. 用完可 `DELETE /admin/delete_address/:id` 或清收件箱  

GPT-Register-Tool 已对接路径：

- 创建：`/admin/new_address`  
- 拉信：`/api/mails`（JWT）+ `/admin/mails` 兜底  
- 配置：`cfworker_url=https://temp-email-api.ccus.ink`，`cfworker_admin_token=tzd_7792526`  
- 域名池示例：`dsnice.com,aqrap.uk,ytouch.me,jabhat.cn`  
- `cfworker_random_subdomain=true`  
- HTTP 库必须是 `requests`，不要 `curl_cffi`

---

## 7. 收信排障清单

| 现象 | 原因 | 处理 |
|---|---|---|
| 创建返回扁平 `a@b.com` 且无 jwt | API 失败 + 本地假地址 | 看 HTTP 状态/中文错误；换 `requests`；检查域名 |
| 有真实地址但永远无 OTP | 域名收信挂了 / 过滤过严 | 换 `dsnice.com` 试一封；`issued_after` 时区按 UTC |
| 403 / Error 1010 | 指纹库被 WAF 拦 | 禁用 curl_cffi impersonate |
| 401 无效凭据 | 打 `/api/*` 没 jwt | 用创建返回的 jwt |
| 管理接口中文「需要管理员密码」 | 缺 `x-admin-auth` | 补头 |
| 域名在 settings 里但收不到 | Email Routing / 该域异常 | 换池内其它域；ytouch.me 曾出过只建不出信 |

时区坑：`created_at` 形如 `2026-08-10 08:40:18`（无时区）按 **UTC** 解析，不要当本地 +8。

---

## 8. 一键自检命令

```bash
# 1) settings
curl -sS "https://temp-email-api.ccus.ink/open_api/settings"

# 2) 创建
curl -sS -X POST "https://temp-email-api.ccus.ink/admin/new_address" \
  -H "x-custom-auth: tzd_7792526" \
  -H "x-admin-auth: tzd_7792526" \
  -H "Content-Type: application/json" \
  -d '{"name":"ping","domain":"dsnice.com","enablePrefix":true,"enableRandomSubdomain":true}'

# 3) 用返回的 address / jwt 拉空箱（应 count=0）
# admin:
# curl -sS "https://temp-email-api.ccus.ink/admin/mails?address=ADDR&limit=5&offset=0" \
#   -H "x-custom-auth: tzd_7792526" -H "x-admin-auth: tzd_7792526"
# jwt:
# curl -sS "https://temp-email-api.ccus.ink/api/mails?limit=5&offset=0" \
#   -H "Authorization: Bearer JWT"
```

实机已验证：创建成功 → `manualtest@e5i3cpn5.dsnice.com`；空箱 admin/jwt 均返回 `{"results":[],"count":0}`。

---

## 9. 安全注意

- 管理员密码等同最高权限（可列全站地址、读任意邮箱）  
- 不要把 admin 密码写进前端公开仓库  
- 批量脚本优先用 **单邮箱 JWT** 读信，少用全局 admin 扫  
- 用完及时删地址，降低 D1 体积与误用面  

---

**交付结论**：创建走 `POST /admin/new_address`（双 admin 头），读信优先 `GET /api/mails` + JWT，管理兜底 `GET /admin/mails?address=&offset=0`。API 根：`https://temp-email-api.ccus.ink`，密码：`tzd_7792526`，当前域名池：`aqrap.uk / ytouch.me / dsnice.com / jabhat.cn`。
