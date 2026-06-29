---
name: citation-check
description: 当用户要求查论文引用、搜索引用文献或进行引用分析时使用。触发词包括"查引"、"引用"、"被引用"、"citation search"、"cited by"、"citation analysis"等。
---

# 学术查引（Citation Check）工作流

## 概述

本 Skill 提供完整的端到端论文学术引用分析流程。通过 Google Scholar（panda985.com 代理 / Semantic Scholar API）搜索目标论文，找出所有引用该论文的文献，下载 PDF 全文，提取引用上下文，查证作者专业职称（IEEE Fellow、中国工程院院士等），最终生成包含完整引用论述的 Excel 分析报告。

---

## 环境配置与安装

### 1. 安装 Playwright（核心依赖）

Playwright 是本流程的浏览器自动化基础，用于打开 IEEE 登录页面让用户认证、提取 Cookie 下载 PDF。

有两种使用方式，**推荐同时安装**：

```bash
# 方式 A：安装为本地 Node.js 项目依赖（推荐——最稳定）
npm install playwright

# 方式 B：安装 Playwright MCP 服务器（MCP 工具名可能因环境而异）
claude mcp add playwright -s user -- npx -y @playwright/mcp@latest

# 验证安装
claude mcp list
# 应显示：playwright: npx -y @playwright/mcp@latest - ✓ Connected
```

### 2. 安装 Python 依赖

需要 Python 3.12+（Windows 推荐使用 `py -3` 启动器）：

```bash
# PyMuPDF — 用于从 PDF 中提取文本
py -3 -m pip install pymupdf --trusted-host pypi.org --trusted-host files.pythonhosted.org

# openpyxl — 用于生成 Excel 报告
py -3 -m pip install openpyxl --trusted-host pypi.org --trusted-host files.pythonhosted.org

# 验证安装
py -3 -c "import fitz; print('PyMuPDF OK'); import openpyxl; print('openpyxl OK')"
```

> **⚠️ 国内高校/企业网络注意事项**：`pip install` 时如报 SSL 证书验证错误，必须添加 `--trusted-host` 参数。同样，Python `urllib` 下载 PDF 时如报 SSL 错误，在代码中添加：`ssl._create_default_https_context = ssl._create_unverified_context`。

### 3. 安装本 Skill

```powershell
# 创建 skill 目录
New-Item -ItemType Directory -Force -Path "$env:USERPROFILE\.claude\skills\citation-check"

# 复制 skill 文件
Copy-Item "SKILL.md" "$env:USERPROFILE\.claude\skills\citation-check\SKILL.md"
```

重启 Claude Code 后生效。

### 4. 完整环境检查清单

| 依赖 | 验证命令 | 安装方式 |
|------|---------|---------|
| Claude Code CLI | `claude --version` | https://code.claude.com |
| Playwright MCP | `claude mcp list` | `claude mcp add playwright ...` |
| Playwright (npm) | `node -e "require('playwright')"` | `npm install playwright` |
| Python 3.12+ | `py -3 --version` | https://python.org |
| PyMuPDF | `py -3 -c "import fitz"` | `pip install pymupdf` |
| openpyxl | `py -3 -c "import openpyxl"` | `pip install openpyxl` |
| PowerShell | Windows 自带 | 系统预装 |
| IEEE Xplore 访问 | 浏览器打开 ieeexplore.ieee.org | 需学校/机构订阅 |

---

## Playwright 的两种用法（重要）

### 情况 A：Playwright MCP 工具可用（自动注册成功时）

如果运行查引时，对话环境中出现了 `browser_navigate`、`browser_snapshot` 等工具，说明 MCP 已自动注册了 Playwright 工具，直接使用即可。

### 情况 B：MCP 工具未注册（经常发生！）

这是实际测试中遇到的情况——Playwright MCP 显示 ✓ Connected，但工具名并未出现在可用工具列表中。

**此时用以下替代方案**：

**步骤 1：打开浏览器让用户交互**
```bash
npx playwright open https://ieeexplore.ieee.org
```
这将在后台打开一个 Chrome 浏览器窗口。告诉用户：
- 浏览器已打开，请查看并登录 IEEE（机构登录）
- 登录成功后告诉"已登录"

**步骤 2：用户登录后，用 Node.js 脚本提取 Cookie**
编写 `get_cookies.js`：
```javascript
const { chromium } = require('playwright');
(async () => {
  const browser = await chromium.launch({ headless: true });
  const context = await browser.newContext({
    userAgent: 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36'
  });
  const page = await context.newPage();
  await page.goto('https://ieeexplore.ieee.org', { waitUntil: 'domcontentloaded', timeout: 30000 });
  await page.waitForTimeout(3000);
  
  // 检查登录状态
  const content = await page.content();
  const isLoggedIn = content.includes('Access provided by') || content.includes('My Settings') || content.includes('Sign Out');
  console.log('Logged in:', isLoggedIn);
  
  // 提取 Cookie
  const cookies = await context.cookies();
  const keyCookies = ['xpluserinfo', 'ERIGHTS', 'JSESSIONID'];
  const cookieStr = cookies
    .filter(c => keyCookies.includes(c.name) || c.name === 'JSESSIONID')
    .map(c => `${c.name}=${c.value}`)
    .join('; ');
  
  console.log('COOKIE:' + cookieStr);
  await browser.close();
})();
```
运行：`node get_cookies.js`

**步骤 3：用 PowerShell 批量下载**
```powershell
$cookie = "xpluserinfo=...; ERIGHTS=..."
$headers = @{"Cookie" = $cookie; "User-Agent" = "Mozilla/5.0"}
$outDir = "$env:TEMP\ieee_papers"; New-Item -Force -Type Directory $outDir | Out-Null

$papers = @(
    @{arnumber="9453784"}
    # ... 每篇论文的 arnumber
)
foreach ($p in $papers) {
    $url = "https://ieeexplore.ieee.org/stampPDF/getPDF.jsp?tp=&arnumber=$($p.arnumber)"
    $outFile = "$outDir\paper_$($p.arnumber).pdf"
    Invoke-WebRequest -Uri $url -Headers $headers -OutFile $outFile -UseBasicParsing -TimeoutSec 60
}
```

---

## 完整工作流步骤

### 第一阶段：搜索目标论文（全自动，无需用户交互）

**1.1 使用 Semantic Scholar API 或 Web Search 初步定位**

首选 Semantic Scholar API（速度快、无验证码）：
```python
import urllib.request, json
url = 'https://api.semanticscholar.org/graph/v1/paper/search?query=论文标题&limit=3&fields=title,year,authors,externalIds,citationCount'
req = urllib.request.Request(url, headers={'User-Agent': 'Mozilla/5.0'})
resp = urllib.request.urlopen(req, timeout=15)
data = json.loads(resp.read())
# 获取 paperId, citationCount, arXiv ID 等
```

**1.2 获取完整引用列表**
```python
url = 'https://api.semanticscholar.org/graph/v1/paper/{paperId}/citations?limit=200&fields=title,year,paperId,externalIds'
req = urllib.request.Request(url)
resp = urllib.request.urlopen(req, timeout=30)
data = json.loads(resp.read())
# 每篇引用论文：citingPaper.paperId, title, year
```

如果需要更全面的引用（Google Scholar 有而 Semantic Scholar 没有的），再用 Playwright 浏览器访问：
- 打开 panda985.com 代理 → 搜索论文 → 点"被引用次数"链接
- 逐页遍历引用列表
- 注意 panda985 可能有图形验证码，处理方式见原版第一阶段

**1.3 识别目标论文信息**

记录：
- 论文标题、作者、期刊、年份
- 引用总数
- DOI / arXiv ID / 引用链接 URL

### 第二阶段：用户筛选引用论文（可选）

如果用户已经有筛选好的引用列表（如从 Excel 读取），跳过通用列表收集，直接进入 PDF 下载阶段。

否则从引用列表中收集所有论文，记录：
- 论文标题、作者、来源
- IEEE arnumber（从 URL 提取）/ arXiv ID / DOI
- 链接地址

按可访问性分类：

| 类型 | 可访问性 | 下载方式 |
|------|---------|---------|
| `ieee` | 需 IEEE Xplore 订阅 | PowerShell + cookies（用户先登录浏览器） |
| `arxiv` | 开放获取 | Python `urllib`（SSL 绕过处理） |
| `open` | 开放获取 | Python `urllib` 或 PowerShell |
| `other` | 其他 | 先尝试浏览器访问 |

### 第三阶段：获取 IEEE Cookies 并下载 PDF（需要用户交互）

**这是整个流程中唯一需要用户操作的步骤。**

**3.1 打开浏览器让用户登录**

先尝试 Playwright MCP 的 `browser_navigate`：
```
browser_navigate → https://ieeexplore.ieee.org
```

如果 MCP 工具名不可用，改用：
```bash
npx playwright open https://ieeexplore.ieee.org
```

然后告诉用户：**"浏览器已打开，请检查是否已登录 IEEE（显示'Access provided by [机构名称]'），如需登录请完成机构认证，然后告诉我'已登录'"**

**3.2 等待用户确认已登录**

用户说"已登录"后，提取 Cookie。

**如果 Playwright MCP 可用：**  
用 `browser_run_code_unsafe` 提取：
```javascript
async (page) => {
  const cookies = await page.context().cookies();
  const keyCookies = ['xpluserinfo', 'ERIGHTS', 'JSESSIONID', 'CloudFront-Key-Pair-Id', 'CloudFront-Policy', 'CloudFront-Signature'];
  const pairs = [];
  for (const c of cookies) {
    if (keyCookies.includes(c.name) || c.name === 'JSESSIONID') {
      pairs.push(`${c.name}=${c.value}`);
    }
  }
  return pairs.join('; ');
}
```

**如果 Playwright MCP 不可用：**  
写 Node.js 脚本 `get_cookies.js`（见上方"情况 B"），用 `node get_cookies.js` 运行。

关键 Cookie：
- **`xpluserinfo`** — 包含机构认证信息的 base64 编码 JSON
- **`ERIGHTS`** — 授权令牌，用于 PDF 下载验证
- **`JSESSIONID`** — 会话标识

**3.3 通过 PowerShell 批量下载 IEEE PDF**

```powershell
$cookies = "xpluserinfo=...; ERIGHTS=...; JSESSIONID=..."
$headers = @{"Cookie" = $cookies; "User-Agent" = "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36"}
$outDir = "$env:TEMP\ieee_papers"; New-Item -Force -Type Directory $outDir | Out-Null

$papers = @(
    @{seq=1; arnumber="XXXXXXXXX"; name="01_Author_Keyword"}
    # ... 列表
)
foreach ($p in $papers) {
    $url = "https://ieeexplore.ieee.org/stampPDF/getPDF.jsp?tp=&arnumber=$($p.arnumber)"
    $outFile = "$outDir\$($p.name).pdf"
    Write-Host "Downloading $($p.seq): $($p.name)..."
    try {
        Invoke-WebRequest -Uri $url -Headers $headers -OutFile $outFile -UseBasicParsing -TimeoutSec 60
        $fileSize = (Get-Item $outFile).Length
        if ($fileSize -gt 1000) { Write-Host "  OK: $fileSize bytes" }
        else { Write-Host "  WARNING: file too small ($fileSize bytes)" }
    } catch { Write-Host "  FAILED: $_" }
    Start-Sleep -Seconds 2  # 避免触发反爬
}
```

`arnumber` 从 IEEE URL 中提取：`https://ieeexplore.ieee.org/abstract/document/XXXXXXXXX/`

> **⚠️ 重要**：IEEE cookies 会过期（约 1-2 小时）。如果下载返回"Error Code: 418"，说明 cookie 已失效，需要重新从浏览器提取。但是注意——第二次提取时不需再让用户登录，因为 `headless: true` 的 Playwright 新会话会自动继承之前的浏览器认证状态？实际上新的 headless 会话不会继承之前已经打开的浏览器窗口的 cookie。所以要么让 MCP 的浏览器保持打开，要么如果 token 已过期需要再次让用户操作。经验：下载要分批次快速进行，尽量不要让 session 过期。

### 第四阶段：下载非 IEEE 论文

**4.1 arXiv 论文**

使用 Python + SSL 绕过：
```python
import urllib.request, ssl
ssl._create_default_https_context = ssl._create_unverified_context
req = urllib.request.Request(f'https://arxiv.org/pdf/{arxiv_id}', headers={'User-Agent': 'Mozilla/5.0'})
with urllib.request.urlopen(req, timeout=60) as f:
    data = f.read()
```

**4.2 其他开放获取论文**

先尝试 PowerShell `Invoke-WebRequest`，如果失败（403/空文件），改为通过浏览器方式下载。

### 第五阶段：提取引用上下文（全自动）

使用 PyMuPDF 从 PDF 中提取文本，搜索目标论文的引用。

**关键提取脚本**（完整版，可直接保存为 `.py` 文件）：

```python
# -*- coding: utf-8 -*-
import fitz, re, os, json, sys, io
sys.stdout = io.TextIOWrapper(sys.stdout.buffer, encoding='utf-8')

def find_citation(pdf_path, target_patterns):
    doc = fitz.open(pdf_path)
    full_text = ''
    for page in doc:
        full_text += page.get_text()
    doc.close()
    lines = full_text.split('\n')
    
    # 定位 References 章节起点
    refs_start = None
    for i, line in enumerate(lines):
        if re.match(r'^\s*(References|REFERENCES|Bibliography)\s*$', line.strip()):
            refs_start = i
            break
    
    # 1. 在参考文献列表中搜索目标论文
    ref_num = None
    ref_entry = ''
    target_line_idx = None
    
    for i, line in enumerate(lines):
        for pat in target_patterns:
            if re.search(pat, line, re.IGNORECASE):
                target_line_idx = i
                break
        if target_line_idx:
            break
    
    if target_line_idx is not None:
        # 提取参考文献编号
        m = re.search(r'\[(\d+)\]', lines[target_line_idx])
        if m:
            ref_num = m.group(1)
        
        # 收集完整的参考文献条目（跨行）
        entry_start = target_line_idx
        while entry_start > 0 and not re.match(r'^\s*\[\d+\]', lines[entry_start]) and len(lines[entry_start].strip()) > 0:
            entry_start -= 1
        if entry_start > 0 and re.match(r'^\s*\[\d+\]', lines[entry_start]):
            entry_lines = []
            j = entry_start
            while j < len(lines) and (j < entry_start + 6 or len(lines[j].strip()) > 0):
                if re.match(r'^\s*\[\d+\]', lines[j]) and j != entry_start:
                    break
                cleaned = re.sub(r'^\s*\d+\s*$', '', lines[j])
                if cleaned.strip():
                    entry_lines.append(cleaned.strip())
                j += 1
            ref_entry = ' '.join(entry_lines).strip()
    
    # 2. 定位正文中的引用语句
    in_text = []
    if ref_num and refs_start:
        for i in range(0, refs_start):
            line = lines[i]
            if re.search(rf'\b{ref_num}\b', line):
                start = max(0, i-2)
                end = min(refs_start, i+4)
                ctx = ' '.join(lines[j].strip() for j in range(start, end))
                in_text.append(ctx)
    
    return {
        'ref_num': f'[{ref_num}]' if ref_num else 'Not found',
        'ref_entry': ref_entry[:300] or 'Not found',
        'in_text': in_text[:3] if in_text else ['Not found in text (only in References)'],
    }
```

**目标论文搜索模式（根据实际情况选择）**：
```python
target_patterns = [
    r'computation.rate.maximi',          # 论文标题关键词
    r'Zhou.*2018.*JSAC',                  # 作者+年份+期刊
    r'F\.\s*Zhou\b',                      # 首字母缩写
    r'JSAC.*2864426',                     # DOI 关键词
    r'UAV.*enabled.*wireless.*powered.*MEC',  # 主题描述
]
```

**常见问题**：
- 部分 PDF 使用加密流 — `fitz.open()` 能处理大多数情况
- 参考文献编号格式多样：有的用 `[1]`，有的用 `(1)` 或上标
- 目标论文可能被引用的是 arXiv 预印本号而非正式期刊版本
- 中英文作者名在参考文献中可能有拼写错误

### 第六阶段：查证作者专业职称

**6.1 从 PDF 作者简介中提取**

许多 IEEE 论文末尾有作者 Biography 段落。搜索模式：
```python
patterns = [
    r'(IEEE\s+(Fellow|Senior\s+Member|Member|Life\s+Fellow|Life\s+Member))',
    r'(Fellow,\s*IEEE)',
    r'(IET\s+Fellow)', r'(AAAS\s+Fellow)', r'(ACM\s+Fellow)',
    r'Academy\s+of\s+Engineering', r'(Optica|OSA|SPIE)\s+Fellow',
]
```

**6.2 关键作者网页搜索**

对主要作者使用 WebSearch 查询：
```
WebSearch → "[作者名] IEEE Fellow 教授"
WebSearch → "[作者名] 中国工程院 院士"
```

### 第七阶段：生成 Excel 报告

**7.1 Excel 列结构**：

| 列 | 内容 |
|----|------|
| 序号 | 顺序编号 |
| 被引文献 | 目标论文完整标题 |
| 被引文献作者 | 作者名 + 职称全称 |
| 被引文献期刊/发表时间 | 期刊名、卷号、页码、年份 + JCR 分区 + IF |
| 引用文献 | 引用论文完整标题 |
| 引用文献作者 | 作者名 + 职称（如 "John Smith (IEEE Fellow, IET Fellow)"） |
| 引用文献来源 | 期刊/会议名称及发表信息 |
| 引用文献发表时间 | 发表日期 |
| 引用文献网站地址 | 论文直接 URL |
| 引用文献中参考文献编号 | 如 "[41]" |
| 引用时的描述/论述 | 中文论述摘要 + 原文引用语句 + 参考文献条目 |

**7.2 Excel 格式规范**（`openpyxl`）：
- 表头行：蓝色背景（`#2F5496`），白色粗体字
- 单元格：自动换行，垂直顶端对齐，细线边框
- 冻结首行，自动筛选
- 字体：微软雅黑，正文 10 号，表头 11 号粗体

**7.3 双 Sheet 结构**：

- **Sheet 1「引用详情」**：主数据表（含中文论述摘要、英文原文上下文、参考文献条目）

  **论述列格式示例**：
  ```
  【引用方式】
  参考文献编号: [28]

  【中文论述摘要】
  该文引用 Zhou 等的 UAV 无线供能 MEC 系统作为关键相关工作...

  【原文引用语句】
  "An UAV enabled mobile edge computing wireless powered system was studied in [28] to maximize the achievable computation rate..."

  【参考文献条目】
  [28] F. Zhou, Y. Wu, R. Q. Hu, and Y. Qian, "Computation rate maximization in UAV-enabled wireless-powered mobile-edge computing systems," IEEE J. Sel. Areas Commun., vol. 36, no. 9, pp. 1927-1941, Sep. 2018.
  ```

- **Sheet 2「汇总统计」**：
  - 目标论文信息
  - 总引用次数、年份范围
  - 含 IEEE Fellow/院士论文数
  - 来源期刊分布

**7.4 保存位置**：
```
E:\平台文件\杰青文件查引\{第一作者}_{论文简称}_查引\
```

### 第八阶段：整理项目文件

标准文件夹结构：

```
{作者}_{论文简称}_查引/
├── {作者}_{论文简称}_引用分析报告.xlsx        ← 最终 Excel 报告
├── 引用文献PDF/                                ← 已下载引用论文 PDF
│   ├── 01_第一作者_论文关键词_来源.pdf
│   └── ...
└── 中间文件/                                   ← 脚本和中间数据
    ├── generate_excel.py
    ├── citation_results.json
    └── get_cookies.js
```

完成后清理临时文件：
```powershell
Remove-Item "$env:TEMP\ieee_papers" -Recurse -Force -ErrorAction SilentlyContinue
```

---

## 异常处理与故障排除

| 问题 | 解决方案 |
|------|---------|
| **Playwright MCP 已连接但工具不可用** | 改用 `npx playwright open URL` + Node.js 脚本方案 |
| **IEEE Cookie 过期 (Error 418)** | 重新提取 Cookie（用 Node.js headless 脚本，不需要再次手动打开浏览器） |
| **Python SSL 错误** | `ssl._create_default_https_context = ssl._create_unverified_context` |
| **PDF 下载为 0 字节** | 网站屏蔽非浏览器请求，检查 Cookie 是否过期或增加 headers（User-Agent、Referer） |
| **PDF 文本提取为乱码** | PDF 可能使用 Type 3 字体或为扫描图片，跳过并标注 |
| **参考文献编号在正文中找不到** | 尝试备选搜索模式（arXiv ID、作者名、部分标题） |
| **PowerShell `$pid` 变量冲突** | PS 5.1 中 `$pid` 为只读保留变量，改用 `$paperId` 等 |
| **Python GBK 编码报错** | 包裹 stdout：`sys.stdout = io.TextIOWrapper(sys.stdout.buffer, encoding='utf-8')` |
| **pip SSL 证书错误** | 添加 `--trusted-host pypi.org --trusted-host files.pythonhosted.org` |
| **Playwright MCP 无响应** | `claude mcp list` 检查状态，必要时 `claude mcp remove playwright` 后重新添加 |

---

## 工作流总结

```
┌─────────────────────────────────────────────────────┐
│  第一阶段：搜索目标论文（Semantic Scholar API / 浏览器）│
│  → 获取目标论文 paperId、citationCount               │
│  → 获取200篇引用论文title+paperId (全自动)             │
└────────────────────────┬────────────────────────────┘
                         ↓
┌─────────────────────────────────────────────────────┐
│  第二阶段：确定引用论文列表（用户筛选 或 全量收集）      │
│  → 从 Excel 读取/从引用列表筛选                       │
│  → 按可访问性分类 (ieee/arxiv/open)                  │
└────────────────────────┬────────────────────────────┘
                         ↓
┌─────────────────────────────────────────────────────┐
│  第三阶段：下载 PDF                                  │
│  ├─ 用户交互：打开浏览器 → 登录 IEEE → 告诉"已登录"    │
│  ├─ Cookie提取：Node.js Playwright 或 MCP 浏览器工具  │
│  ├─ IEEE批量下载：PowerShell + Invoke-WebRequest     │
│  └─ 开放获取：Python urllib / 浏览器方式             │
└────────────────────────┬────────────────────────────┘
                         ↓
┌─────────────────────────────────────────────────────┐
│  第四阶段：提取引用上下文（PyMuPDF，全自动）            │
│  → 查找参考文献编号                                  │
│  → 提取参考文献条目                                   │
│  → 提取正文引用语句                                   │
└────────────────────────┬────────────────────────────┘
                         ↓
┌─────────────────────────────────────────────────────┐
│  第五阶段：查证作者职称 + 生成 Excel 报告（全自动）    │
│  → 从PDF/网页查证IEEE Fellow/院士等                  │
│  → 生成双Sheet Excel（引用详情 + 汇总统计）           │
└─────────────────────────────────────────────────────┘
```

## 重要注意事项

1. **机构访问权限至关重要**：IEEE Xplore 论文需要机构订阅。需要用户在浏览器中完成机构认证登录。

2. **Cookie 有效期**：IEEE Cookie 约 1-2 小时后过期。提取后应尽快批量下载。

3. **用户只需操作一次**：整个流程中用户只需在浏览器中登录 IEEE 并告知"已登录"。

4. **遵守速率限制**：API 调用间隔 ≥3 秒，PDF 下载间隔 ≥2 秒。

5. **职称以 PDF 为准**：PDF 作者简介是最权威的来源。网页搜索结果可能过时。
