# 反检测策略指南

## 基本原则

1. **模拟真人行为**：随机延迟、合理的请求频率
2. **保持浏览器状态**：复用 session、维护 cookie
3. **首选 API**：API 调用比页面操作更不易被检测
4. **优雅降级**：被限制时主动退避，而非强行突破

## 请求频率控制

### Python requests 场景
```python
import random, time

# 请求间隔：基础延迟 + 随机抖动
def polite_sleep(base=1.0):
    time.sleep(base + random.uniform(0.2, 0.8))
```

### Playwright 场景
```bash
# 在操作间插入等待
"$PWCLI" click <ref>
sleep 1.5  # 等待页面响应
"$PWCLI" snapshot
```

### 推荐频率
| 场景 | 频率上限 |
|------|---------|
| API调用 | 每秒1次 |
| 页面操作 | 每次操作间隔1-3秒 |
| 文件下载 | 每个文件间隔2-5秒 |
| 翻页 | 每页间隔2-4秒 |

## User-Agent 管理

### 轮换策略
```python
USER_AGENTS = [
    "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 Chrome/122.0.0.0 Safari/537.36",
    "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 Chrome/121.0.0.0 Safari/537.36",
    "Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:123.0) Gecko/20100101 Firefox/123.0",
]

import random
headers["User-Agent"] = random.choice(USER_AGENTS)
```

### Playwright 场景
Playwright 使用真实浏览器，User-Agent 已是正常值，通常无需额外处理。

## Cookie / Session 管理

### Python requests
```python
session = requests.Session()

# 1. 先访问首页建立 session
session.get(base_url, headers=headers)

# 2. 后续请求复用同一 session
response = session.post(api_url, data=payload)
```

### Playwright
```bash
# 同一 session 内操作，cookie 自动维护
"$PWCLI" open <url>
# 后续操作共享浏览器状态
```

## 验证码处理策略

### 策略1：降低频率避免触发
大多数网站的验证码在请求频率过高时触发。降低到人类速度通常可以避免。

### 策略2：Playwright headed 模式人工介入
```bash
# 使用 headed 模式，遇到验证码时暂停等待用户手动处理
"$PWCLI" open <url> --headed
"$PWCLI" snapshot
# 如果发现验证码元素，提示用户手动处理
"$PWCLI" screenshot  # 截图让用户看到验证码
# 等待用户确认已处理
"$PWCLI" snapshot  # 重新获取页面状态
```

### 策略3：识别验证码类型
```
snapshot 中出现以下元素时判定为验证码：
- 包含 "验证码" / "captcha" / "verify" 的元素
- img 标签含 captcha 相关 URL
- iframe 含 captcha 域名
```

### 策略4：滑块验证
```
部分网站使用滑块验证（如巨潮资讯高频访问时）：
1. 使用 --headed 模式
2. 截图给用户查看
3. 等待用户手动完成验证
4. 继续操作
```

## IP 限制应对

### 策略1：请求间隔
```python
# 检测到 429/503 时，指数退避
def retry_with_backoff(func, max_retries=3):
    for attempt in range(max_retries):
        try:
            return func()
        except Exception:
            wait = (2 ** attempt) * 5 + random.uniform(1, 5)
            time.sleep(wait)
```

### 策略2：分时段请求
```
将大批量任务拆分为多段：
- 每段处理10-20条
- 段间等待60-120秒
- 避免在短时间内发送大量请求
```

### 策略3：任务拆分
```python
# 将大批量下载拆分为小批次
batch_size = 10
for i in range(0, len(tasks), batch_size):
    batch = tasks[i:i+batch_size]
    process_batch(batch)
    if i + batch_size < len(tasks):
        time.sleep(60)  # 批次间等待
```

## HTTP 错误码应对

| 状态码 | 含义 | 应对 |
|--------|------|------|
| 429 | 请求过多 | 等待30-60秒后重试 |
| 421 | 请求误导 | 检查URL和Host头 |
| 403 | 禁止访问 | 检查cookie/session是否过期 |
| 503 | 服务不可用 | 等待后重试 |
| 302 | 重定向 | 跟随重定向，更新session |

## 常见网站特定注意事项

### 巨潮资讯 (cninfo.com.cn)
- 高频API调用可能触发滑块验证
- 建议：每页请求间隔2秒以上

### 中国货币网 (chinamoney.com.cn)
- 必须先访问页面建立session
- 421错误常见，需退避重试
- 限制并发连接数

### 国家统计局 (stats.gov.cn)
- 数据加载较慢，操作间多等待
- 无明显反爬，但仍建议控制频率

### 政府采购网 (ccgp.gov.cn)
- 验证码可能出现（搜索频率过高时）
- 建议：搜索间隔3秒以上
