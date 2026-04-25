# 国家统计局 (stats.gov.cn) 操作指南

## 籽车概述

国家统计局数据平台提供宏观经济统计数据，包含月度、季度、年度等频度的各类指标。

**推荐策略：Playwright 为主**（数据以Echarts可视化+表格形式展示，需要浏览器交互）

## 关键URL

| 页面 | URL |
|------|-----|
| 月度数据 | `https://data.stats.gov.cn/dg/website/page.html#/pc/national/monthData` |
| 季度数据 | `https://data.stats.gov.cn/dg/website/page.html#/pc/national/quarterData` |
| 年度数据 | `https://data.stats.gov.cn/dg/website/page.html#/pc/national/yearData` |
| 数据API（可能） | `https://data.stats.gov.cn/easyquery.htm` |

## 界面结构（基于截图分析）

```
┌─────────────────────────────────────────┐
│ 顶部导航栏                               │
├──────────┬──────────────────────────────┤
│ 左侧     │ 右侧数据区域                  │
│ 分类     │ ┌──────────────────────────┐ │
│ 导航树   │ │ Echarts 图表             │ │
│          │ └──────────────────────────┘ │
│ 月度数据 │ ┌──────────────────────────┐ │
│  ├ CPI   │ │ 数据表格                 │ │
│  ├ PPI   │ │ 时间 | 指标1 | 指标2     │ │
│  ├ GDP   │ │ 2024-01 | xxx | xxx     │ │
│  └ ...   │ │ ...                      │ │
│          │ └──────────────────────────┘ │
├──────────┴──────────────────────────────┤
│ 时间范围选择器                            │
└─────────────────────────────────────────┘
```

## Playwright 操作流程

### Step 1: 打开页面

```bash
"$PWCLI" open "https://data.stats.gov.cn/dg/website/page.html#/pc/national/monthData" --headed
# 等待页面完全加载（数据量大，需要较长等待）
sleep 5
"$PWCLI" snapshot
```

### Step 2: 导航到目标指标

```bash
# 在左侧分类树中找到并点击目标分类
# 例如：点击"CPI"指标
"$PWCLI" click <cpi_treeitem_ref>
sleep 3  # 等待数据加载
"$PWCLI" snapshot
```

### Step 3: 提取表格数据

```bash
# 方法1：提取HTML表格
"$PWCLI" eval "document.querySelector('.table-container table').outerHTML"

# 方法2：提取Echarts数据（如果表格不可见）
"$PWCLI" eval "JSON.stringify(window.myChart?.getOption())"

# 方法3：提取页面中所有表格数据为JSON
"$PWCLI" eval "JSON.stringify(Array.from(document.querySelectorAll('table tbody tr')).map(row => Array.from(row.cells).map(cell => cell.textContent.trim())))"
```

### Step 4: 调整时间范围

```bash
# 如果需要调整时间范围，找到时间选择器
"$PWCLI" click <time_selector_ref>
"$PWCLI" snapshot
"$PWCLI" fill <start_date_ref> "2023-01"
"$PWCLI" fill <end_date_ref> "2024-12"
"$PWCLI" click <confirm_ref>
sleep 3
"$PWCLI" snapshot
```

## API 发现（可能存在）

统计局网站可能使用以下API格式：

```python
# 常见API端点模式
possible_api = "https://data.stats.gov.cn/easyquery.htm"

params = {
    "m": "QueryData",
    "dbcode": "fsnd",       # 数据库代码
    "rowcode": "zb",        # 行代码
    "colcode": "sj",        # 列代码
    "wds": "[]",            # 维度
    "dfwds": '[{"wdcode":"zb","valuecode":"A0801"}]',  # 指标筛选
}

# 用 network 命令监听以确认实际API格式
```

### 用 Playwright network 发现API

```bash
"$PWCLI" network
# 然后在页面上操作（点击指标等）
# 观察network输出中的请求
# 找到返回JSON数据的API端点
```

## 数据提取脚本示例

```python
# 从Echarts实例提取数据
import json

def extract_echarts_data(html_source):
    """从页面HTML中提取Echarts数据"""
    # 查找Echarts实例
    # 方法：eval获取 chart.getOption()
    pass

def extract_table_data(table_html):
    """从HTML表格提取数据"""
    from html.parser import HTMLParser

    class TableParser(HTMLParser):
        def __init__(self):
            super().__init__()
            self.rows = []
            self.current_row = []
            self.in_cell = False

        def handle_starttag(self, tag, attrs):
            if tag == 'tr':
                self.current_row = []
            elif tag in ('td', 'th'):
                self.in_cell = True

        def handle_endtag(self, tag):
            if tag == 'tr' and self.current_row:
                self.rows.append(self.current_row)
            elif tag in ('td', 'th'):
                self.in_cell = False

        def handle_data(self, data):
            if self.in_cell:
                self.current_row.append(data.strip())

    parser = TableParser()
    parser.feed(table_html)
    return parser.rows
```

## 已知注意事项

1. **页面加载慢**：统计数据量大，首次加载需要5-10秒
2. **Echarts数据**：图表数据可能存在JS变量中，需要通过eval提取
3. **时间格式**：月度数据格式为 YYYY-MM（如"2024-01"）
4. **指标编码**：每个指标有唯一编码（如A0801=CPI），需要通过操作或API发现
5. **导出按钮**：部分页面可能有"导出"按钮，优先使用
6. **数据更新**：月度数据通常在下月中旬更新
