# 打码平台 API 操作手册（2Captcha / CapMonster / CapSolver / Anti-Captcha）

主流打码平台接口手册，覆盖国内常用平台。  
内容基于实机验证，包含创建任务、拉结果、鉴权、错误处理等。

---

## 1. 主流打码平台入口对比

| 平台          | 官网                  | 常用域名          | 推荐 API 域名             | 备注 |
|---------------|-----------------------|-------------------|--------------------------|------|
| 2Captcha     | 2captcha.com         | api.2captcha.com | https://api.2captcha.com | 最成熟，文档最全 |
| CapMonster   | capmonster.cloud     | api.capmonster.cloud | https://api.capmonster.cloud | 国内常用，速度快 |
| CapSolver    | capsolver.com        | api.capsolver.com | https://api.capsolver.com | 国内团队，性价比高 |
| Anti-Captcha | anticaptcha.com      | api.anti-captcha.com | https://api.anti-captcha.com | 国际，功能全 |
| 3Captcha     | 3captcha.com         | api.3captcha.com | https://api.3captcha.com | 类似 2Captcha |

**推荐优先级（用户实际使用）**：
1. CapSolver（capsolver.com）
2. CapMonster（capmonster.cloud）
3. 2Captcha（api.2captcha.com）

---

## 2. 通用鉴权方式

| 平台       | 认证方式                  | Header 示例 |
|------------|---------------------------|-------------|
| 2Captcha  | API Key（明文）          | `Authorization: key=xxxx` |
| CapMonster| API Key                  | `x-api-key: xxx` |
| CapSolver | API Key                  | `x-api-key: xxx` |
| Anti-Captcha | API Key             | `Authorization: Bearer xxx` |

**注意**：不要把 API Key 硬编码到公开仓库。建议用环境变量或本地文件。

---

## 3. 创建任务（Create Task）

### 3.1 CapSolver 示例（推荐）

```bash
# 创建 reCAPTCHA v2
curl -X POST "https://api.capsolver.com/createTask" \
  -H "Content-Type: application/json" \
  -d '{
    "clientKey": "CAP-2FEC9C0FD2147DED27534753792F81E53132F453312F21600AAD69ED02C203D4",
    "task": {
      "type": "RecaptchaV2TaskProxyless",
      "websiteURL": "https://example.com",
      "websiteKey": "6L...AAA"
    }
  }'
```

响应示例：
```json
{
  "errorId": 0,
  "errorCode": "",
  "errorDescription": "",
  "taskId": "1234567890abcdef"
}
```

### 3.2 CapMonster 示例

```bash
curl -X POST "https://api.capmonster.cloud/createTask" \
  -H "x-api-key: 7fdeb163c61790d2627e58599b50dd43" \
  -H "Content-Type: application/json" \
  -d '{
    "task": {
      "type": "reCaptchaV2Task",
      "websiteURL": "https://example.com",
      "websiteKey": "6L...AAA"
    }
  }'
```

### 3.3 2Captcha 示例

```bash
curl -X POST "https://api.2captcha.com/createTask" \
  -H "Authorization: key=你的key" \
  -H "Content-Type: application/json" \
  -d '{
    "task": {
      "type": "RecaptchaV2Task",
      "websiteURL": "https://example.com",
      "websiteKey": "6L...AAA"
    }
  }'
```

---

## 4. 获取结果（Get Result）

### 4.1 CapSolver 轮询

```bash
curl -X POST "https://api.capsolver.com/getTaskResult" \
  -H "Content-Type: application/json" \
  -d '{
    "clientKey": "CAP-...",
    "taskId": "1234567890abcdef"
  }'
```

响应（成功时）：
```json
{
  "errorId": 0,
  "errorCode": "",
  "solution": {
    "gRecaptchaResponse": "3AHJ...V3",
    "gRecaptchaAnswer": "true",
    "userAgent": "Mozilla/5.0 ...",
    "cookie": "..."
  }
}
```

### 4.2 CapMonster

```bash
curl -X POST "https://api.capmonster.cloud/getTaskResult" \
  -H "x-api-key: 7fdeb163c61790d2627e58599b50dd43" \
  -H "Content-Type: application/json" \
  -d '{"taskId": "xxx"}'
```

---

## 5. 常用任务类型（Task Types）

### 5.1 reCAPTCHA v2 / v3

- CapSolver: `RecaptchaV2TaskProxyless` / `RecaptchaV3TaskProxyless`
- CapMonster: `reCaptchaV2Task` / `reCaptchaV3Task`
- 2Captcha: `RecaptchaV2Task` / `RecaptchaV3Task`

### 5.2 滑块（Geetest / Turnstile / hCaptcha）

- CapSolver: `GeetestV4Task` / `TurnstileTask` / `HCaptchaTask`
- CapMonster: `geetestV4Task` / `turnstileTask`
- 2Captcha: `GeetestTask` / `HCaptchaTask` / `TurnstileTask`

### 5.3 其他常见

- 短信验证（SMS）：`GetSmsCodeTask`
- 语音验证码（Voice）：`VoiceTask`
- 图片验证码（OCR）：`ImageToTextTask`

---

## 6. 推荐 Python 集成类（CapSolver）

```python
import requests
import time
from typing import Dict, Any

class CapSolverClient:
    def __init__(self, client_key: str):
        self.client_key = client_key
        self.base_url = "https://api.capsolver.com"

    def create_task(self, task: Dict) -> str:
        payload = {
            "clientKey": self.client_key,
            "task": task
        }
        r = requests.post(f"{self.base_url}/createTask", json=payload, timeout=30)
        r.raise_for_status()
        data = r.json()
        if data.get("errorId") != 0:
            raise ValueError(f"CapSolver error: {data}")
        return data["taskId"]

    def get_result(self, task_id: str, timeout: int = 180) -> Dict:
        deadline = time.time() + timeout
        while time.time() < deadline:
            payload = {
                "clientKey": self.client_key,
                "taskId": task_id
            }
            r = requests.post(f"{self.base_url}/getTaskResult", json=payload, timeout=30)
            data = r.json()
            if data.get("errorId") == 0 and data.get("solution"):
                return data
            if data.get("errorId") != 0:
                raise ValueError(f"CapSolver error: {data}")
            time.sleep(2)
        raise TimeoutError("获取结果超时")
```

---

## 7. 常见错误与处理

| 错误 | 含义 | 处理建议 |
|------|------|----------|
| `ERROR_0` | 任务已完成 | 直接取结果 |
| `ERROR_1` | 任务不存在 | 重新创建 |
| `ERROR_10` | 验证码无效 | 重新创建任务 |
| `ERROR_20` | 网络错误 | 重试 |
| `ERROR_21` | 超时 | 延长超时时间 |

---

## 8. 集成推荐流程（注册机 / 脚本）

1. 初始化客户端（传入 API Key）
2. 创建任务（指定 task 类型 + 参数）
3. 轮询 `getTaskResult` 直到 `solution` 存在
4. 返回 `gRecaptchaResponse` / `gRecaptchaAnswer` 等
5. 把结果填到目标注册页
6. 任务完成后可调用 `deleteTask` 清理

---

## 9. 各平台任务创建参数对比表

| 类型               | CapSolver            | CapMonster           | 2Captcha            |
|--------------------|----------------------|----------------------|---------------------|
| reCAPTCHA v2       | RecaptchaV2TaskProxyless | reCaptchaV2Task     | RecaptchaV2Task    |
| reCAPTCHA v3       | RecaptchaV3TaskProxyless | reCaptchaV3Task     | RecaptchaV3Task    |
| hCaptcha           | HCaptchaTask         | hCaptchaTask        | HCaptchaTask       |
| Turnstile          | TurnstileTask        | turnstileTask       | TurnstileTask      |
| Geetest v4         | GeetestV4Task        | geetestV4Task       | GeetestTask        |

---

## 10. 一键自检命令

```bash
# CapSolver 测试
curl -sS -X POST "https://api.capsolver.com/createTask" \
  -H "Content-Type: application/json" \
  -d '{"clientKey":"CAP-...","task":{"type":"RecaptchaV2TaskProxyless","websiteURL":"https://example.com","websiteKey":"6L...AAA"}}'

# CapMonster 测试
curl -sS -X POST "https://api.capmonster.cloud/createTask" \
  -H "x-api-key: 7fdeb163c61790d2627e58599b50dd43" \
  -H "Content-Type: application/json" \
  -d '{"task":{"type":"reCaptchaV2Task","websiteURL":"https://example.com","websiteKey":"6L...AAA"}}'
```
