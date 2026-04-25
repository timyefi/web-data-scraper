# 中国政府采购网 (ccgp.gov.cn) 操作指南

## 网站概述

中国政府采购网发布中央及地方政府采购公告、中标结果等信息。

**推荐策略：Playwright 为主**（典型列表+详情结构，需要两步提取）

## 关键URL

| 页面 | URL |
|------|-----|
| 首页 | `https://www.ccgp.gov.cn/` |
| 信息公告列表 | `https://www.ccgp.gov.cn/xxgg/index.shtml` |
| 招标公告 | 按分类筛选 |
| 中标公告 | 按分类筛选 |

## 界面结构（基于截图分析）

```
┌──────────────────────────────────────────┐
│ 搜索框 [________________] [搜索]         │
├──────────────────────────────────────────┤
│ 分类Tab: 招标公告 | 中标公告 | 更正公告... │
├──────────────────────────────────────────┤
│ 信息列表:                                 │
│  1. [公告标题]         2024-01-15  地区   │
│  2. [公告标题]         2024-01-15  地区   │
│  3. [公告标题]         2024-01-15  地区   │
│  ...                                     │
├──────────────────────────────────────────┤
│ 分页: [1] [2] [3] ... [下一页]            │
└──────────────────────────────────────────┘
```

## Playwright 操作流程

### Step 1: 打开信息公告页面

```bash
"$PWCLI" open "https://www.ccgp.gov.cn/xxgg/index.shtml" --headed
"$PWCLI" snapshot
```

### Step 2: 搜索/筛选

```bash
# 方法1：使用搜索框
"$PWCLI" fill <search_ref> "信息化建设"
"$PWCLI" click <search_btn_ref>
sleep 2
"$PWCLI" snapshot

# 方法2：使用分类Tab
"$PWCLI" click <tab_zhaobiao_ref>  # 点击"招标公告"
sleep 2
"$PWCLI" snapshot
```

### Step 3: 提取列表数据

```bash
# 提取当前页所有公告信息
"$PWCLI" eval "JSON.stringify(Array.from(document.querySelectorAll('.vF_c_list li, .list-content li')).map(item => ({
  title: item.querySelector('a')?.textContent?.trim(),
  link: item.querySelector('a')?.href,
  date: item.querySelector('.date, span')?.textContent?.trim()
})))"
```

### Step 4: 进入详情页提取内容

```bash
# 点击第一条公告
"$PWCLI" click <first_item_ref>
"$PWCLI" snapshot

# 提取详情页内容
"$PWCLI" eval "document.querySelector('.vF_detail_content, .detail-content, .article-content')?.textContent"

# 返回列表页
"$PWCLI" go-back
"$PWCLI" snapshot
```

### Step 5: 处理分页

```bash
# 点击下一页
"$PWCLI" click <next_page_ref>
sleep 2
"$PWCLI" snapshot

# 重复 Step 3-4 直到所有页面处理完毕
```

## 批量提取完整流程

```bash
# 伪代码流程
all_items = []
page = 1

while True:
    snapshot → 提取列表 → 加入 all_items
    if 存在"下一页"按钮:
        click 下一页
        page += 1
    else:
        break

# 对每个 item 进入详情页提取完整内容
for item in all_items:
    open item.link
    snapshot → 提取详情内容
    go-back
```

## 已知注意事项

1. **搜索频率**：频繁搜索可能触发验证码，间隔3秒以上
2. **详情页结构**：不同类型的公告详情页结构可能略有不同
3. **附件下载**：部分公告包含附件（PDF/Word），需要额外处理
4. **地区筛选**：支持按地区筛选采购信息
5. **时间范围**：支持按发布日期筛选
6. **数据量**：政府采购信息量大，注意控制抓取范围
7. **列表→详情**：这是典型的两步提取模式，先获取列表元数据，再逐条进入详情
