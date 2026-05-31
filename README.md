# Citation Check Skill (查引)

A Claude Code skill for automated academic citation analysis. Given a paper title, it searches Google Scholar, finds all citing papers, downloads their PDFs, extracts citation contexts, looks up author professional titles, and generates a comprehensive Excel report.

---

## Setup (One-Time)

### 1. Install Playwright MCP Server

```bash
claude mcp add playwright npx @playwright/mcp@latest
```

Verify: `claude mcp list` should show `playwright: ... ✓ Connected`

### 2. Install Python Dependencies

```bash
py -3 -m pip install pymupdf openpyxl --trusted-host pypi.org --trusted-host files.pythonhosted.org
```

### 3. Install This Skill

```powershell
New-Item -ItemType Directory -Force -Path "$env:USERPROFILE\.claude\skills\citation-check"
Copy-Item "SKILL.md" "$env:USERPROFILE\.claude\skills\citation-check\SKILL.md"
```

### 4. Prerequisites Checklist

| Requirement | How to Check | How to Install |
|-------------|-------------|----------------|
| Claude Code CLI | `claude --version` | https://code.claude.com |
| Playwright MCP | `claude mcp list` | `claude mcp add playwright npx @playwright/mcp@latest` |
| Python 3.12+ | `py -3 --version` | https://python.org |
| PyMuPDF | `py -3 -c "import fitz"` | `pip install pymupdf` |
| openpyxl | `py -3 -c "import openpyxl"` | `pip install openpyxl` |
| IEEE Xplore access | Browser login at ieeexplore.ieee.org | Institutional subscription |

---

## Usage

```
帮我查引这篇论文：[paper title]，作者：[authors]
```

Examples:
```
帮我查引这篇论文：FSOS-AMC: Few-Shot Open-Set Learning for Automatic Modulation Classification Over Multipath Fading Channels，作者：Hao Zhang
查一下Dong Peihao这篇论文的被引情况：Differentially Private Federated Learning Based Wideband Spectrum Sensing
```

The skill auto-triggers on: `查引`, `引用`, `被引用`, `citation search`, `cited by`

---

## What It Does

1. **Search** → Google Scholar via panda985.com (handles CAPTCHA automatically)
2. **Find** → All citing papers with metadata
3. **Download** → PDFs (IEEE Xplore via cookie auth, arXiv open access, browser-based for Cloudflare sites)
4. **Extract** → Exact citation context from each PDF (text + reference entry)
5. **Lookup** → Author professional titles (IEEE Fellow, 中国工程院院士, etc.)
6. **Generate** → Excel report with full details

## Output Structure

```
{FirstAuthor}_{ShortTitle}_查引/
├── {Author}_引用分析报告.xlsx               ← Excel report (2 sheets)
├── 引用文献PDF/                              ← Download PDFs (named by paper title)
│   ├── 01_Ma_Few-Shot_AMC_...pdf
│   └── 02_Zhang_Federated_Learning_...pdf
└── 中间文件/                                 ← Scripts and JSON data
```

## Key Features

- **Visible browser**: Playwright opens a real Chrome window — you can see and interact with it
- **IEEE cookie extraction**: Automatically handles institutional authentication
- **Author title lookup**: Scans PDF biographies and web searches for IEEE Fellow, Academy memberships, etc.
- **Citation context extraction**: Finds exact sentences where your paper is discussed
- **Multi-source support**: IEEE Xplore, arXiv, TechRxiv, open-access repositories

## Troubleshooting

| Problem | Solution |
|---------|----------|
| Playwright MCP not found | Run `claude mcp add playwright npx @playwright/mcp@latest` |
| pip SSL error | Add `--trusted-host pypi.org --trusted-host files.pythonhosted.org` |
| IEEE Error 418 | Cookies expired — re-extract from browser |
| Cloudflare blocks download | Use browser-based download (Playwright passes Turnstile) |
| Python GBK encoding error | Wrap stdout with `io.TextIOWrapper(sys.stdout.buffer, encoding='utf-8')` |

## Author

Created for automated citation analysis workflow.

## License

MIT
