# 中国货币网 (chinamoney.com.cn) 操作指南

## 网站概述

中国货币网是银行间市场信息披露平台，提供债券评级报告、发行人财报、公告等信息。

**推荐策略：API + Playwright 混合**（需先建立session，再调用API）

## 关键URL

| 页面 | URL |
|------|-----|
| 信息披露首页 | `https://www.chinamoney.com.cn/chinese/ywts/` |
| 财报查询 | `https://www.chinamoney.com.cn/chinese/cqcwbglm/` |
| 评级公告 | `https://www.chinamoney.com.cn/chinese/pjgg/` |
| 债券评级报告 | `https://www.chinamoney.com.cn/chinese/zxpjbgh/` |
| 财报API | `https://www.chinamoney.com.cn/ags/ms/cm-u-notice-issue/financeRepo` |
| 年份类型API | `https://www.chinamoney.com.cn/ags/ms/cm-u-notice-an/staYearAndType` |
| 文件下载 | `https://www.chinamoney.com.cn/dqs/cm-s-notice-query/fileDownLoad.do?contentId={ID}&priority=0&mode=save` |

## API调用方法

### Step 1: 建立 Session（必须）

```python
import requests

session = requests.Session()
headers = {
    "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36",
    "Accept-Language": "zh-CN,zh;q=0.9",
}

# 先访问页面建立 session cookie
session.get("https://www.chinamoney.com.cn/chinese/zqcwbgcwgd/", headers=headers)
```

### Step 2: 查询财报列表

```python
api_url = "https://www.chinamoney.com.cn/ags/ms/cm-u-notice-issue/financeRepo"
api_headers = {
    "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36",
    "X-Requested-With": "XMLHttpRequest",
    "Referer": "https://www.chinamoney.com.cn/chinese/zqcwbgcwgd/",
    "Origin": "https://www.chinamoney.com.cn",
}

data = {
    "year": "2024",           # 年度
    "type": "4",              # 报告类型
    "orgName": "",             # 机构名称，空=全市场
    "pageSize": "30",
    "pageNo": "1",
    "inextp": "3,5",
    "limit": "1",
}

response = session.post(api_url, headers=api_headers, data=data)
result = response.json()
```

### 报告类型代码

| 代码 | 含义 |
|------|------|
| 4 | 年度报告 |
| 2 | 半年度报告 |
| 1 | 一季度报告 |
| 3 | 三季度报告 |

### Step 3: 构造下载链接

```python
content_id = record["contentId"]
download_url = f"https://www.chinamoney.com.cn/dqs/cm-s-notice-query/fileDownLoad.do?contentId={content_id}&priority=0&mode=save"
```

## Playwright 方式

```bash
"$PWCLI" open "https://www.chinamoney.com.cn/chinese/cqcwbglm/" --browser=firefox
"$PWCLI" snapshot

# 填机构名称
"$PWCLI" fill <search_ref> "万科"

# 选年份
"$PWCLI" select <year_ref> "2024"

# 选报告类型
"$PWCLI" select <type_ref> "年度报告"

# 点查询
"$PWCLI" click <query_ref>

# 结果可能在新标签页
"$PWCLI" tab-list
"$PWCLI" tab-select 1
"$PWCLI" snapshot
```

## 已有技能参考

已有专门的 `chinamoney` 技能提供更完整的功能：
- `discover_reports.py` — API数据发现
- `download_support.py` — 下载核心（含CNInfo镜像回退）
- `batch-download.py` — 批量下载

**建议**：如果只需操作中国货币网，直接使用 `chinamoney` 技能。本指南仅作为通用技能的示例参考。

## 已知注意事项

1. **必须先建立session**：直接调用API会返回403
2. **421错误**：并发连接过多时会触发，需要退避重试
3. **下载链接时效性**：contentId长期有效，但高频下载可能被限制
4. **搜索结果在新标签页**：Playwright操作时需要切换tab
5. **中文编码**：Windows环境需注意UTF-8编码设置
