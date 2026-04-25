---
name: web-data-scraper
description: >
  通用网站数据查询与抓取技能。当用户需要从任意网站查询、搜索、抓取、下载数据时触发。
  典型场景：批量下载公告报告、提取网页表格数据、搜索网站信息、自动化网页查询。
  触发词：网站数据、抓取、爬取、批量下载、查询数据、数据采集、网页提取、公告下载、报告下载。
  支持 API 优先 + Playwright 兜底的混合策略，输出 Markdown 和 CSV 格式。
---

# 通用网站数据查询技能

从任意网站高效查询、提取、下载数据。支持 API 直取和 Playwright 浏览器自动化两种策略。

## 核心工作流

每次面对新网站，按以下3个阶段执行：

### 阶段A — 网站分析（必做）

```
1. 用 Playwright 打开目标页面（--headed 模式调试）
2. snapshot 获取页面结构
3. 识别关键交互元素：搜索框、筛选条件、查询按钮、结果列表、分页
4. 用 network 命令监听网络请求，发现隐藏 API
5. 判断数据加载方式：静态HTML / AJAX / 后端API
6. 输出分析结论 → 选择策略
```

判断策略的决策树：
```
network 捕获到 JSON API 请求？
  ├─ 是 → 策略1：API直取（最高效）
  └─ 否 → 页面有搜索/筛选交互？
       ├─ 是 → 策略2：Playwright表单操作
       └─ 否 → 策略3：Playwright页面直读
```

### 阶段B — 数据查询与提取

**策略1：API直取**
```bash
# 发现API后，用脚本直接调用
python scripts/api_discovery.py --url "<api_url>" --method POST \
  --data '{"key":"value"}' --output result.json
```

**策略2：Playwright表单操作**
```bash
export PWCLI="$HOME/.codex/skills/playwright/scripts/playwright_cli.sh"

"$PWCLI" open <url> --headed
"$PWCLI" snapshot                    # 获取元素引用
"$PWCLI" fill <search_ref> <keyword> # 填搜索条件
"$PWCLI" select <dropdown_ref> <val> # 选下拉项
"$PWCLI" click <button_ref>          # 点查询
"$PWCLI" snapshot                    # 重新获取结果页引用
"$PWCLI" eval "el => el.textContent" <item_ref>  # 提取数据
```

**策略3：Playwright页面直读**
```bash
"$PWCLI" open <url>
"$PWCLI" snapshot
# 直接从 snapshot 中读取数据表格
```

分页处理：
```bash
# 传统分页：循环 click 下一页按钮
"$PWCLI" click <next_page_ref>
"$PWCLI" snapshot

# API分页：修改 pageNum 参数循环请求
```

### 阶段C — 数据保存与下载

```
1. 列表/表格数据 → 保存为 CSV + Markdown
   python scripts/data_extractor.py --input result.json --format csv,md --output ./output/

2. 文件下载（PDF/Word等）→ 批量下载
   python scripts/batch_downloader.py --config download-tasks.json

3. 网站结构分析 → 生成分析报告
   python scripts/site_analyzer.py --url <url> --output ./analysis/
```

## Playwright 操作规则

- 每次操作前必须 `snapshot` 获取最新元素引用
- 页面跳转/弹窗/Tab切换后必须重新 `snapshot`
- 调试用 `--headed`，确认后可切 headless
- 用 `network` 发现隐藏 API（优先级最高）
- 用 `console` 排查 JS 错误
- 元素引用失效时立即重新 `snapshot`，不要猜测

## 脚本说明

| 脚本 | 用途 |
|------|------|
| `scripts/site_analyzer.py` | 打开URL，截图+snapshot+network，输出分析报告 |
| `scripts/api_discovery.py` | 建立session，调用API，输出JSON |
| `scripts/data_extractor.py` | 从JSON/HTML提取数据，输出CSV和Markdown |
| `scripts/batch_downloader.py` | 配置驱动的批量文件下载，支持重试和断点续传 |

## 参考文档

按需加载：
- **通用方法论**：`references/methodology.md` — 网站分类、元素识别、分页策略等
- **反检测策略**：`references/anti-detection.md` — 验证码、IP限制、请求频率控制
- **具体网站指南**：
  - `references/site-examples/cninfo.md` — 巨潮资讯网
  - `references/site-examples/chinamoney.md` — 中国货币网
  - `references/site-examples/statsgov.md` — 国家统计局
  - `references/site-examples/ccgp.md` — 政府采购网

## 网站配置模板

对新网站，使用 `assets/site-config-template.json` 快速配置，记录已发现的元素选择器和API端点。

## 快速示例

### 从巨潮资讯下载审计报告
```bash
# 1. 发现API（巨潮有公开API，无需Playwright）
python scripts/api_discovery.py \
  --url "http://www.cninfo.com.cn/new/hisAnnouncement/query" \
  --method POST \
  --data '{"searchkey":"万科","category":"category_ndbg_szsh","pageNum":"1","pageSize":"30"}' \
  --output ./output/cninfo_result.json

# 2. 提取数据
python scripts/data_extractor.py \
  --input ./output/cninfo_result.json \
  --format csv,md \
  --output ./output/cninfo/

# 3. 批量下载PDF
python scripts/batch_downloader.py --config ./output/cninfo/download-tasks.json
```

### 从统计局获取月度数据
```bash
# 1. Playwright打开页面
"$PWCLI" open "https://data.stats.gov.cn/dg/website/page.html#/pc/national/monthData" --headed
"$PWCLI" snapshot

# 2. 导航到目标指标
"$PWCLI" click <category_ref>
"$PWCLI" snapshot
"$PWCLI" click <indicator_ref>
"$PWCLI" snapshot

# 3. 提取表格数据
"$PWCLI" eval "document.querySelector('table').outerHTML"
```
