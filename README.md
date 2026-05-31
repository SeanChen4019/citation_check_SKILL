# Citation Check Skill（学术查引）

一个 Claude Code Skill，用于自动化论文学术引用分析。输入一篇论文标题，自动搜索 Google Scholar、找出所有引用论文、下载 PDF、提取引用上下文、查证作者职称（IEEE Fellow / 院士等）、生成完整的 Excel 分析报告。

---

## 环境安装（一次性）

### 1. 安装 Playwright MCP 服务器

Playwright MCP 是一个浏览器自动化服务器，让 Claude Code 能操控 Chrome 浏览器：

```bash
# 安装到当前项目
claude mcp add playwright npx @playwright/mcp@latest

# 或全局安装（所有项目都可用）
claude mcp add playwright -s user -- npx -y @playwright/mcp
```

验证安装：
```bash
claude mcp list
# 应显示：playwright: npx @playwright/mcp@latest - ✓ Connected
```

**重要**：Playwright MCP 会打开一个**可见的 Chrome 浏览器窗口**。你可以实时看到浏览器操作，也可以随时自己动手点击、登录，Claude Code 会从你操作后的页面继续工作。

MCP 服务器提供的工具（自动以 `mcp__playwright__` 前缀命名）：
- `browser_navigate` — 导航到指定 URL
- `browser_click` — 点击页面元素
- `browser_type` — 在输入框中输入文字
- `browser_press_key` — 按下键盘按键
- `browser_snapshot` — 获取页面无障碍快照
- `browser_take_screenshot` — 截图
- `browser_run_code_unsafe` — 执行任意 Playwright/JavaScript 代码
- `browser_tabs` — 管理浏览器标签页
- `browser_wait_for` — 等待文字出现或计时
- `browser_console_messages` — 获取浏览器控制台输出

### 2. 安装 Python 依赖

需要 Python 3.12+（Windows 下使用 `py -3` 启动器）：

```bash
# 安装 PyMuPDF（PDF 文本提取）
py -3 -m pip install pymupdf --quiet --trusted-host pypi.org --trusted-host files.pythonhosted.org

# 安装 openpyxl（Excel 生成）
py -3 -m pip install openpyxl --quiet --trusted-host pypi.org --trusted-host files.pythonhosted.org

# 验证安装
py -3 -c "import fitz; print('PyMuPDF OK'); import openpyxl; print('openpyxl OK')"
```

> **高校/企业网络特别说明**：如果在 `pip install` 时遇到 SSL 证书错误，请务必加上 `--trusted-host` 参数。如果 Python 的 `urllib` 下载 PDF 时报 SSL 错误，在代码中使用：`ssl._create_default_https_context = ssl._create_unverified_context`。

### 3. 安装本 Skill

```powershell
# 创建 skill 目录
New-Item -ItemType Directory -Force -Path "$env:USERPROFILE\.claude\skills\citation-check"

# 复制 skill 文件
Copy-Item "SKILL.md" "$env:USERPROFILE\.claude\skills\citation-check\SKILL.md"
```

安装后重启 Claude Code。Skill 会在检测到以下关键词时自动触发：
- "帮我查引这篇论文"
- "查一下引用"
- "被引用次数"
- "citation check"
- "cited by"

### 4. 环境检查清单

| 依赖 | 检查命令 | 安装方法 |
|------|---------|---------|
| Claude Code CLI | `claude --version` | https://code.claude.com |
| Playwright MCP | `claude mcp list \| Select-String playwright` | `claude mcp add playwright npx @playwright/mcp@latest` |
| Python 3.12+ | `py -3 --version` | https://python.org 或 `winget install python` |
| PyMuPDF | `py -3 -c "import fitz"` | `py -3 -m pip install pymupdf` |
| openpyxl | `py -3 -c "import openpyxl"` | `py -3 -m pip install openpyxl` |
| PowerShell | Windows 自带 | 系统预装 |
| IEEE Xplore 访问 | 浏览器打开 https://ieeexplore.ieee.org | 需要机构订阅（学校图书馆） |

---

## 使用方法

在 Claude Code 中直接说：

```
帮我查引这篇论文：[论文标题]，作者：[作者名]
```

示例：
```
帮我查引这篇论文：FSOS-AMC: Few-Shot Open-Set Learning for Automatic Modulation Classification Over Multipath Fading Channels，作者：Hao Zhang
```

也可以说：
- "查这篇论文被谁引用过：[标题]"
- "/citation-check"
- "帮我看看这些论文的引用情况"

---

## 功能流程

1. **搜索** → Google Scholar（通过 panda985.com 代理，自动处理验证码）
2. **列表** → 找出所有引用论文及其元数据
3. **下载** → PDF（IEEE Xplore 通过 cookie 认证；arXiv 开放获取；Cloudflare 防护站点通过浏览器下载）
4. **提取** → 从每篇 PDF 中提取引用上下文（原文引用句 + 参考文献条目）
5. **职称** → 查证作者职称（IEEE Fellow、中国工程院院士、长江学者等）
6. **报告** → 生成双 Sheet 的 Excel 分析报告

---

## 输出目录结构

```
{第一作者}_{论文简称}_查引/
├── {作者}_引用分析报告.xlsx                  ← Excel 报告（2个Sheet）
├── 引用文献PDF/                              ← 已下载的引用论文 PDF（以论文标题命名）
│   ├── 01_马_小样本自动调制识别_半监督度量学习.pdf
│   └── 02_张_联邦学习_轻量级网络_无人机认证.pdf
└── 中间文件/                                 ← 脚本和中间数据
    ├── generate_excel.py
    └── citation_results.json
```

---

## 常见问题

| 问题 | 解决方案 |
|------|---------|
| Playwright MCP 未找到 | 运行 `claude mcp add playwright npx @playwright/mcp@latest` 安装 |
| pip SSL 证书错误 | 添加 `--trusted-host pypi.org --trusted-host files.pythonhosted.org` |
| IEEE 下载报 Error 418 | Cookie 过期，需要在浏览器中重新提取（浏览器会话保持登录状态） |
| Cloudflare 拦截下载 | 通过 Playwright 浏览器下载（浏览器能通过 Turnstile 验证） |
| PDF 下载为 0 字节 | 网站屏蔽了非浏览器请求，改用浏览器方式下载 |
| Python GBK 编码报错 | 在代码中包裹 stdout：`sys.stdout = io.TextIOWrapper(sys.stdout.buffer, encoding='utf-8')` |
| PowerShell `$pid` 变量冲突 | PS 5.1 中 `$pid` 是只读变量，改用 `$paperId` 等变量名 |
| PDF 提取文字为乱码 | PDF 可能使用了 Type 3 字体或扫描图片，跳过并标注 |
| 参考文献编号未找到 | 尝试其他搜索模式（arXiv ID、作者名、部分标题匹配） |

---

## 许可

MIT
