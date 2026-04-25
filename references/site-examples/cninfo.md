# 巨潮资讯网 (cninfo.com.cn) 操作指南

## 网站概述

巨潮资讯网是证监会指定的上市公司信息披露网站，提供公告查询、下载功能。

**推荐策略：API直取**（巨潮有完善的公开JSON API，无需Playwright）

## 关键URL

| 页面 | URL |
|------|-----|
| 信息披露搜索 | `https://www.cninfo.com.cn/new/commonUrl/pageOfSearch?url=disclosure/list/search&checkedCategory=category_ndbg_szsh` |
| 公告搜索API | `http://www.cninfo.com.cn/new/hisAnnouncement/query` |
| 全文搜索API | `http://www.cninfo.com.cn/new/information/topSearch/query` |
| PDF静态下载 | `https://static.cninfo.com.cn/` + adjunctUrl |

## API调用方法

### 搜索公告（POST）

```python
import requests

url = "http://www.cninfo.com.cn/new/hisAnnouncement/query"
headers = {
    "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36",
    "X-Requested-With": "XMLHttpRequest",
    "Referer": "https://www.cninfo.com.cn/new/commonUrl/pageOfSearch?url=disclosure/list/search",
    "Accept": "application/json, text/javascript, */*; q=0.01",
    "Origin": "https://www.cninfo.com.cn",
}

# 查询参数
data = {
    "pageNum": "1",          # 页码
    "pageSize": "30",        # 每页条数
    "column": "szse",        # 板块：szse=深市, sse=沪市
    "tabName": "fulltext",   # 搜索类型
    "plate": "sz",           # plate: sz=深市, sh=沪市
    "searchkey": "",          # 搜索关键词
    "secid": "",              # 证券代码
    "category": "category_ndbg_szsh;",  # 公告类型
    "trade": "",              # 行业
    "seDate": "2024-01-01~2024-12-31",  # 时间范围
    "sortName": "",
    "sortType": "",
    "isHLtitle": "true",
}

response = requests.post(url, headers=headers, data=data)
result = response.json()
```

### 公告类型代码

| 类型 | category值 |
|------|-----------|
| 年度报告 | `category_ndbg_szsh` |
| 半年度报告 | `category_bndbg_szsh` |
| 一季度报告 | `category_yjdbg_szsh` |
| 三季度报告 | `category_sjdbg_szsh` |
| 审计报告 | `category_sj_bg_szsh` |
| 调研记录 | 需在全文搜索中使用关键词 |
| 利润分配 | `category_lr_gs_szsh` |

### 响应结构

```json
{
  "announcements": [
    {
      "announcementTitle": "标题",
      "announcementId": "ID",
      "secCode": "证券代码",
      "secName": "证券简称",
      "adjunctUrl": "相对下载路径",
      "announcementTime": 时间戳,
      "columnId": "板块",
      "pageColumn": "column"
    }
  ],
  "totalAnnouncement": 总数,
  "totalRecordNum": 总记录数,
  "hasMore": true/false
}
```

### 下载PDF

```python
# 拼接下载URL
pdf_url = "https://static.cninfo.com.cn/" + adjunct_url.lstrip("/")
response = requests.get(pdf_url, headers={"User-Agent": "..."})
```

## Playwright 方式（备用）

如果 API 不可用，使用 Playwright：

```bash
"$PWCLI" open "https://www.cninfo.com.cn/new/commonUrl/pageOfSearch?url=disclosure/list/search&checkedCategory=category_ndbg_szsh" --headed
"$PWCLI" snapshot

# 填搜索关键词
"$PWCLI" fill <search_ref> "万科"

# 选择公告类型（如果有下拉）
"$PWCLI" select <type_ref> "年度报告"

# 点查询
"$PWCLI" click <query_ref>
"$PWCLI" snapshot

# 提取结果列表
"$PWCLI" eval "JSON.stringify(Array.from(document.querySelectorAll('.result-item')).map(el => ({title: el.querySelector('.title')?.textContent, date: el.querySelector('.date')?.textContent, link: el.querySelector('a')?.href})))"
```

## 已知注意事项

1. **高频API调用触发滑块验证**：每页请求间隔2秒以上
2. **板块选择**：深市用 `szse/sz`，沪市用 `sse/sh`，可分别查询后合并
3. **时间格式**：`seDate` 参数格式为 `YYYY-MM-DD~YYYY-MM-DD`
4. **adjunctUrl**：以 `/` 开头，需要拼接 `https://static.cninfo.com.cn` 前缀
5. **跨板块搜索**：同一个公司可能在深市和沪市都有记录
