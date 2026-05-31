---
name: citation-check
description: 当用户要求查论文引用、搜索引用文献或进行引用分析时使用。触发词包括"查引"、"引用"、"被引用"、"citation search"、"cited by"、"citation analysis"等。
---

# 学术查引（Citation Check）工作流

## 概述

本 Skill 提供完整的端到端论文学术引用分析流程。通过 Google Scholar（panda985.com 代理）搜索目标论文，找出所有引用该论文的文献，下载 PDF 全文，提取引用上下文，查证作者专业职称（IEEE Fellow、中国工程院院士等），最终生成包含完整引用论述的 Excel 分析报告。

---

## 环境配置与安装

### 1. 安装 Playwright MCP 服务器

Playwright MCP 是一个浏览器自动化服务器，让 Claude Code 能直接操控 Chrome 浏览器进行搜索、翻页、下载：

```bash
# 安装到当前项目
claude mcp add playwright npx @playwright/mcp@latest

# 或全局安装（所有项目均可用）
claude mcp add playwright -s user -- npx -y @playwright/mcp
```

验证安装：
```bash
claude mcp list
# 应显示：playwright: npx @playwright/mcp@latest - ✓ Connected
```

**核心机制说明**：Playwright MCP 会打开一个**真实可见的 Chrome 浏览器窗口**运行。你能够实时看到浏览器操作过程，也可以随时接管操作（如手动登录 IEEE、点击验证码）。Claude Code 会从你操作完成后的页面状态继续自动化流程。

安装成功后，对话中会出现以下 Playwright 工具（均以 `mcp__playwright__` 前缀命名）：

| 工具名 | 功能 |
|--------|------|
| `browser_navigate` | 导航到指定 URL |
| `browser_click` | 点击页面元素 |
| `browser_type` | 在输入框中输入文字 |
| `browser_press_key` | 按下键盘按键 |
| `browser_snapshot` | 获取页面无障碍结构快照（优于截图，可直接获取文字） |
| `browser_take_screenshot` | 截取页面截图 |
| `browser_run_code_unsafe` | 执行任意 Playwright/JavaScript 代码（下载 PDF、提取 Cookie 等） |
| `browser_tabs` | 管理浏览器标签页（新建、切换、关闭） |
| `browser_wait_for` | 等待指定文字出现/消失或计时 |
| `browser_console_messages` | 获取浏览器控制台输出（调试用） |
| `browser_evaluate` | 在页面或元素上执行 JavaScript 表达式 |
| `browser_fill_form` | 批量填写表单 |
| `browser_file_upload` | 上传文件 |

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

重启 Claude Code 后，对话中提及以下关键词即自动触发 Skill：
- "查引"、"引用"、"被引用"
- "查一下谁引用了这篇"
- "citation check"、"cited by"、"citation search"

### 4. 完整环境检查清单

| 依赖 | 验证命令 | 安装方式 |
|------|---------|---------|
| Claude Code CLI | `claude --version` | https://code.claude.com |
| Playwright MCP | `claude mcp list` | `claude mcp add playwright npx @playwright/mcp@latest` |
| Python 3.12+ | `py -3 --version` | https://python.org |
| PyMuPDF | `py -3 -c "import fitz"` | `pip install pymupdf` |
| openpyxl | `py -3 -c "import openpyxl"` | `pip install openpyxl` |
| PowerShell | Windows 自带 | 系统预装 |
| IEEE Xplore 访问权限 | 浏览器打开 ieeexplore.ieee.org 查看 | 需学校/机构订阅 |

### 5. 快速上手

环境配置完成后，在 Claude Code 中直接说：

```
帮我查引这篇论文：[论文标题]，作者：[作者名]
```

示例：
```
帮我查引这篇论文：FSOS-AMC: Few-Shot Open-Set Learning for Automatic Modulation Classification Over Multipath Fading Channels，作者：Hao Zhang
```

---

## 完整工作流步骤

### 第一阶段：搜索目标论文

**1.1 导航到搜索引擎**

使用 Playwright 导航到 Google Scholar 代理站点：
```
mcp__playwright__browser_navigate → https://sc.panda985.com/index.html
```

**1.2 处理安全验证（如遇）**

panda985.com 可能出现安全验证页面：
- Canvas 图形验证码，要求点击指定形状（如"绿色三角形"）
- 遇到时用 `browser_run_code_unsafe` 获取页面 HTML 源码，读取 `<script>` 标签中的 `IMAGE_TYPE` 和 `TARGET` 变量
- 分析 canvas 绘制代码，找到目标形状的像素坐标
- 用 `browser_run_code_unsafe` 在 canvas 上点击正确坐标
- 点击"提交验证"按钮完成验证

**超时机制**：如果遇到需要用户手动登录或验证的情况，等待约 30 秒。若用户未响应，跳过当前论文继续下一项。

**1.3 搜索论文**

在搜索框中输入论文标题并回车：
```
mcp__playwright__browser_type → input[type="search"] → 论文标题
mcp__playwright__browser_press_key → Enter
```

**1.4 识别目标论文**

获取页面快照，定位目标论文条目，记录：
- 论文标题、作者、期刊、年份
- "被引用次数：X" 链接（引用数量）
- 引用链接 URL（包含 `cites=XXXXXXXXX`）

若同一论文有多个版本，选择引用次数最多的版本。

### 第二阶段：查看引用文献列表

**2.1 进入引用列表**

点击"被引用次数：X"链接：
```
mcp__playwright__browser_click → a[href*="cites=XXXXXXXXX"]
```

**2.2 收集所有引用论文信息**

获取每页快照，对每篇引用论文记录：
- 论文标题（完整）
- 作者（按原文显示）
- 来源（期刊/会议/arXiv）
- 直接链接（通常指向 ieeexplore.ieee.org 或 arxiv.org）
- PDF 链接（如有，特别是 arXiv PDF 链接）

若有多页结果，点击"下一页"逐页遍历。

**2.3 按可访问性分类**

| 类型 | 可访问性 | 下载方式 |
|------|---------|---------|
| `ieee` | 需 IEEE Xplore 订阅 | PowerShell + `xpluserinfo` + `ERIGHTS` cookies |
| `arxiv` | 开放获取 | Python `urllib`（需 SSL 绕过处理） |
| `techrxiv` | 开放但有 Cloudflare 防护 | 仅限浏览器（需通过 Turnstile 验证） |
| `proquest` | 需单独订阅 | 需手动访问 |
| `open` | 开放获取（phwl.org、clemson.edu 等） | Python `urllib` |
| `other` | 其他 | 先尝试浏览器访问 |

### 第三阶段：获取 IEEE Cookies（针对 IEEE 论文）

**3.1 检查用户登录状态**

在浏览器中打开任意 IEEE Xplore 论文页面：
```
mcp__playwright__browser_navigate → https://ieeexplore.ieee.org/abstract/document/XXXXXXXXXX/
```

如果页面顶部显示"Access provided by [机构名称]"，说明已登录。

**3.2 提取 Cookies**

用 `browser_run_code_unsafe` 获取关键 cookies：
```javascript
async (page) => {
  const cookies = await page.context().cookies();
  const keyCookies = ['xpluserinfo', 'ERIGHTS', 'CloudFront-Key-Pair-Id', 'CloudFront-Policy', 'CloudFront-Signature'];
  const pairs = [];
  for (const c of cookies) {
    if (keyCookies.includes(c.name)) {
      pairs.push(`${c.name}=${c.value}`);
    }
  }
  return pairs.join('; ');
}
```

**3.3 通过 PowerShell 下载 IEEE PDF**

用提取的 cookies 在 PowerShell 中下载 PDF：
```powershell
$cookies = "xpluserinfo=...; ERIGHTS=...; CloudFront-Key-Pair-Id=...; CloudFront-Policy=...; CloudFront-Signature=..."
$headers = @{"Cookie" = $cookies; "User-Agent" = "Mozilla/5.0"}
$outFile = "$env:TEMP\ieee_papers\paper_X.pdf"
Invoke-WebRequest -Uri "https://ieeexplore.ieee.org/stampPDF/getPDF.jsp?tp=&arnumber=XXXXXXXXX" -Headers $headers -OutFile $outFile -UseBasicParsing -TimeoutSec 60
```

`arnumber` 从 IEEE URL 中提取：`https://ieeexplore.ieee.org/abstract/document/XXXXXXXXX/`

**⚠️ 重要**：IEEE cookies 会过期（约 1-2 小时）。如果下载返回"Error Code: 418"，说明 cookie 已失效，需要重新从浏览器提取（浏览器会话保持活跃）。

### 第四阶段：下载非 IEEE 论文

**4.1 arXiv 论文**

使用 Python + SSL 绕过（部分企业网络需要）：
```python
import urllib.request, ssl
ssl._create_default_https_context = ssl._create_unverified_context
req = urllib.request.Request(f'https://arxiv.org/pdf/{arxiv_id}', headers={'User-Agent': 'Mozilla/5.0'})
with urllib.request.urlopen(req, timeout=60) as f:
    data = f.read()
```

保存到：`$env:TEMP\paper_{id}.pdf`

**4.2 其他开放获取论文**

先尝试 PowerShell：
```powershell
Invoke-WebRequest -Uri "URL" -OutFile "$env:TEMP\paper_X.pdf" -UseBasicParsing -TimeoutSec 60
```

如果失败（403/空文件），改为通过 Playwright 浏览器下载。TechRxiv、Clemson 等站点使用 Cloudflare 机器人检测，仅允许浏览器访问。

**4.3 TechRxiv 论文（Cloudflare 防护）**

必须通过浏览器：
1. 在浏览器中导航到 PDF URL
2. 等待 Cloudflare Turnstile 验证完成
3. 通过浏览器 fetch 下载 PDF：
```javascript
const res = await fetch(window.location.href);
const buffer = await res.arrayBuffer();
// 通过 base64 编码保存（大文件可能不适用此方式）
```

### 第五阶段：提取引用上下文

使用 PyMuPDF 从 PDF 中提取文本，搜索目标论文的引用：

```python
import fitz, re

doc = fitz.open(pdf_path)
full_text = ''
for page in doc:
    full_text += page.get_text()
doc.close()

lines = full_text.split('\n')

# 1. 定位参考文献条目
ref_num = None
ref_entry = ''
for i, line in enumerate(lines):
    if re.search(r'论文关键词|FSOS|few.shot.open.set', line, re.IGNORECASE):
        m = re.search(r'\[(\d+)\]', line)
        if m:
            ref_num = m.group(1)
            # 获取跨行的完整参考文献条目
            entry_lines = [line]
            j = i + 1
            while j < len(lines) and not re.match(r'^\s*\[\d+\]', lines[j]) and lines[j].strip():
                entry_lines.append(lines[j])
                j += 1
            ref_entry = ' '.join(entry_lines).strip()
            break

# 2. 定位正文中的引用（在 References 章节之前）
if ref_num:
    in_text = []
    refs_start = None
    for i, line in enumerate(lines):
        if re.match(r'^\s*(References|REFERENCES|Bibliography)\s*$', line):
            refs_start = i
            break
    
    if refs_start:
        for i, line in enumerate(lines[:refs_start]):
            if re.search(rf'(?<!\w)\[{ref_num}\](?!\w)', line):
                start = max(0, i-3)
                end = min(refs_start, i+5)
                ctx = ' '.join(lines[j].strip() for j in range(start, end))
                in_text.append(ctx)
```

**重要**：打印输出时需指定 UTF-8 编码以正确显示中文字符：
```python
sys.stdout = io.TextIOWrapper(sys.stdout.buffer, encoding='utf-8')
```

**常见问题**：
- 部分 PDF 使用加密流 — `fitz.open()` 能处理大多数情况
- 参考文献编号格式多样：有的用 `[1]`，有的用 `(1)` 或上标
- 目标论文可能引用的是 arXiv 预印本号而非正式期刊版本
- 中英文作者名在参考文献中可能有拼写错误（如 "Yuen, Q. W. C." 实际应为 Qihui Wu 和 Chau Yuen）

### 第六阶段：查证作者专业职称

**6.1 从 PDF 作者简介中提取**

许多 IEEE 论文末尾有作者 Biography 段落。搜索模式：
```python
patterns = [
    r'(IEEE\s+(Fellow|Senior\s+Member|Member|Life\s+Fellow|Life\s+Member))',
    r'(Fellow,\s*IEEE)',
    r'(IET\s+Fellow)',
    r'(AAAS\s+Fellow)',
    r'(ACM\s+Fellow)',
    r'Academy\s+of\s+Engineering',
    r'(Optica|OSA|SPIE)\s+Fellow',
]
```

**6.2 关键作者网页搜索**

对主要作者（第一作者、通讯作者等），使用 WebSearch 查询：
```
WebSearch → "[作者名] IEEE Fellow 教授"
WebSearch → "[作者名] 中国工程院 院士"
```

**6.3 职称层级体系（中国学术语境）**：

| 级别 | 职称 |
|------|------|
| 院士 | 中国工程院院士 (CAE Academician)、中国科学院院士 (CAS Academician) |
| 国际学会 Fellow | IEEE Fellow, IET Fellow, AAIA Fellow, Optica Fellow, SPIE Fellow, SAEng Fellow |
| 高级会员 | IEEE Senior Member |
| 国家级人才 | 长江学者特聘教授、国家杰青、国家优青、NSF CAREER Award |
| 会员 | IEEE Member, CCF Member |

**6.4 已知作者职称速查表（NUAA 团队常用）**：

| 作者 | 核心职称 |
|------|---------|
| 吴启晖 (Qihui Wu) | IEEE Fellow, IET Fellow, 长江学者特聘教授, 南航副校长 |
| 周福辉 (Fuhui Zhou) | IEEE Senior Member, 南航教授, 国家优青 |
| Chau Yuen | IEEE Fellow (2021), AAIA Fellow, NTU Singapore Provost's Chair, 高被引学者 |
| 马凯光 (Kai-Kuang Ma) | IEEE Life Fellow, 新加坡工程院 Fellow (SAEng), 南航特聘教授 |
| Shilong Pan | IEEE Fellow, Optica Fellow, SPIE Fellow, IET Fellow, 南航教授 |
| 杨小牛 (Xiaoniu Yang) | 中国工程院院士 (2013), 中国电子学会会士, 中国电科首席科学家 |
| 张浩 (Hao Zhang) | IEEE Member, CCF Member, 南航博士后 |
| Philip H.W. Leong | IEEE Fellow, 悉尼大学教授 |
| Mérouane Debbah | IEEE Fellow (2015), WWRF Fellow, EURASIP Fellow, AAIA Fellow |

### 第七阶段：生成 Excel 报告

**7.1 Excel 列结构**：

| 列 | 内容 |
|----|------|
| 序号 | 顺序编号 |
| 被引文献 | 目标论文完整标题 |
| 被引文献作者 | 作者名 + 职称全称 |
| 被引文献期刊/发表时间 | 期刊名、卷号、页码、年份 + JCR 分区 + 影响因子 |
| 引用文献 | 引用论文完整标题 |
| 引用文献作者 | 作者名 + 职称全称（如 "John Smith (IEEE Fellow, IET Fellow)"） |
| 引用文献来源 | 期刊/会议名称及发表信息 |
| 引用文献发表时间 | 发表日期（从 PDF 元数据或来源页提取） |
| 引用文献网站地址 | 论文直接 URL |
| 引用文献中参考文献编号 | 引用论文中使用的参考文献编号，如 "[41]" |
| 引用时的描述/论述 | 综合分析：中文论述摘要 + 英文原文上下文 + 参考文献条目 |

**7.2 Excel 格式规范**：

使用 `openpyxl` 生成：
- 表头行：蓝色背景（#2F5496），白色粗体字
- 所有单元格：自动换行，垂直顶端对齐
- 所有单元格：细线边框
- 冻结首行
- 自动筛选
- 列宽按内容调整（标题列宽，编号列窄）
- 字体：微软雅黑，正文 10 号，表头 11 号粗体

**7.3 双 Sheet 结构**：

- **Sheet 1「引用详情」**：主数据表
- **Sheet 2「汇总统计」**：汇总统计，包含：
  - 目标论文信息
  - 总引用次数
  - 已分析/无法获取论文数量
  - 引用论文发表时间分布
  - 引用描述类型分布（详细描述/简略提及/仅参考文献）
  - 作者职称汇总

**7.4 保存位置**：

每篇论文创建独立文件夹：
```
E:\平台文件\杰青文件查引\{第一作者}_{论文简称}_查引\
```

### 第八阶段：整理项目文件

每篇论文的标准文件夹结构：

```
{作者}_{论文简称}_查引/
├── {作者}_{论文简称}_引用分析报告.xlsx        ← 最终 Excel 报告
├── 引用文献PDF/                                ← 已下载引用论文 PDF
│   ├── 01_第一作者_论文关键词_来源.pdf
│   └── ...
└── 中间文件/                                   ← 脚本和中间数据
    ├── generate_excel.py
    ├── citation_results.json
    └── ...
```

完成后清理临时文件：
```powershell
Remove-Item "$env:TEMP\ieee_papers" -Recurse -Force
Remove-Item "$env:TEMP\paper_*.pdf" -Force
```

### 第九阶段：异常处理与故障排除

| 问题 | 解决方案 |
|------|---------|
| IEEE Cookie 过期 (Error 418) | 重新从浏览器提取 Cookie（浏览器保持活跃会话） |
| Python SSL 错误 | 使用 `ssl._create_default_https_context = ssl._create_unverified_context` |
| Cloudflare Turnstile 阻止下载 | 通过 Playwright 浏览器 `fetch()` 下载 + base64 编码传输 |
| PDF 下载为 0 字节 | 网站屏蔽非浏览器请求，改用浏览器方式下载 |
| PDF 文本提取为乱码 | PDF 可能使用 Type 3 字体或为扫描图片，跳过并标注 |
| 参考文献编号在正文中找不到 | 尝试备选搜索模式（arXiv ID、作者名、部分标题） |
| PowerShell `$pid` 变量冲突 | PS 5.1 中 `$pid` 为只读保留变量，改用 `$paperId` |
| Python GBK 编码报错 | 包裹 stdout：`sys.stdout = io.TextIOWrapper(sys.stdout.buffer, encoding='utf-8')` |
| pip SSL 证书错误 | 添加 `--trusted-host pypi.org --trusted-host files.pythonhosted.org` |
| Playwright MCP 无响应 | 运行 `claude mcp list` 检查状态，必要时重启 Claude Code |

---

## 脚本模板速查

**批量 IEEE PDF 下载模板（PowerShell）：**
```powershell
$cookies = "xpluserinfo=...; ERIGHTS=..."
$headers = @{"Cookie" = $cookies; "User-Agent" = "Mozilla/5.0"}
$outDir = "$env:TEMP\ieee_papers"; New-Item -Force -Type Directory $outDir | Out-Null
$papers = @(@{id=1; arnumber="XXXXXXXX"})
foreach ($p in $papers) {
    Invoke-WebRequest -Uri "https://ieeexplore.ieee.org/stampPDF/getPDF.jsp?tp=&arnumber=$($p.arnumber)" -Headers $headers -OutFile "$outDir\paper_$($p.id).pdf" -UseBasicParsing -TimeoutSec 60
}
```

**批量引用上下文提取模板（Python）：**
```python
import fitz, re, os, json, sys, io
sys.stdout = io.TextIOWrapper(sys.stdout.buffer, encoding='utf-8')

def find_citation(pdf_path, target_keywords):
    doc = fitz.open(pdf_path)
    text = ''.join(page.get_text() for page in doc)
    doc.close()
    lines = text.split('\n')
    # ... 参考文献定位逻辑（详见第五阶段）...
    return {'ref_num': ref_num, 'ref_entry': ref_entry, 'in_text': in_text}

results = []
for pid, path in pdf_map.items():
    cit = find_citation(path, ['目标', '关键词'])
    results.append({'id': pid, **cit})
```

---

## 重要注意事项

1. **机构访问权限至关重要**：IEEE Xplore 论文需要从已认证的浏览器会话中提取 Cookie。如果用户尚未登录 IEEE，请先提示其登录。

2. **Cookie 有效期**：IEEE Cookie 约 1-2 小时后过期。如果下载突然失败，从浏览器重新提取 Cookie 即可。

3. **遵守速率限制**：不要频繁快速请求 Google Scholar。连续多篇论文查引时适当放缓节奏。

4. **验证提取数据**：PDF 文本提取可能不完美。尽可能与 Google Scholar 列表交叉验证。

5. **非英文论文**：中文论文（如电子与信息学报）的作者名格式可能有差异。同时使用中英文模式搜索。

6. **职称以 PDF 为准**：PDF 作者简介是最权威的来源。网页搜索结果可能过时。冲突时以 PDF（即论文发表版本）中的标注为准。
