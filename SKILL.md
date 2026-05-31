---
name: citation-check
description: Use when the user asks to check citations of a paper, search for citing papers, or perform citation analysis. Triggers on phrases like "查引", "引用", "被引用", "citation search", "cited by", "citation analysis", or any request to find papers that cite a specific paper.
---

# Citation Check (查引) Workflow

## Overview

This skill provides a complete end-to-end workflow for checking academic paper citations using Google Scholar (via panda985.com proxy) and analyzing citing papers. It automates searching for a target paper, finding all papers that cite it, downloading their PDFs, extracting citation contexts, looking up author professional titles (IEEE Fellow, Academy memberships, etc.), and generating a comprehensive Excel report.

---

## Setup / Installation

### 1. Install Playwright MCP Server

Playwright MCP is a browser automation server that lets Claude Code control a Chrome browser:

```bash
# Install to current project (recommended)
claude mcp add playwright npx @playwright/mcp@latest

# Or install globally (available for all projects)
claude mcp add playwright -s user -- npx -y @playwright/mcp
```

Verify installation:
```bash
claude mcp list
# Should show: playwright: npx @playwright/mcp@latest - ✓ Connected
```

The MCP server provides tools prefixed with `mcp__playwright__`:
- `browser_navigate` — navigate to a URL
- `browser_click` — click on elements
- `browser_type` — type text into inputs
- `browser_press_key` — press keyboard keys
- `browser_snapshot` — get accessibility snapshot of page
- `browser_take_screenshot` — capture screenshot
- `browser_run_code_unsafe` — execute arbitrary Playwright/JS code
- `browser_tabs` — manage browser tabs
- `browser_wait_for` — wait for text or time
- `browser_console_messages` — get console output
- `browser_evaluate` — evaluate JavaScript on page

**Important**: The Playwright MCP uses a **visible Chrome browser window**. When you ask Claude Code to use it, you'll see the browser operate in real-time. You can interact with it yourself (login, click, type) and Claude Code will continue from there.

### 2. Install Python Dependencies

Requires Python 3.12+ (Windows uses `py -3` launcher):

```bash
# Install PyMuPDF for PDF text extraction
py -3 -m pip install pymupdf --quiet --trusted-host pypi.org --trusted-host files.pythonhosted.org

# Install openpyxl for Excel generation
py -3 -m pip install openpyxl --quiet --trusted-host pypi.org --trusted-host files.pythonhosted.org

# Verify
py -3 -c "import fitz; print('PyMuPDF OK'); import openpyxl; print('openpyxl OK')"
```

> **Note for corporate networks (e.g., Chinese university networks)**: If you encounter SSL certificate errors during `pip install`, add the `--trusted-host` flags as shown above. If Python's `urllib` encounters SSL errors when downloading PDFs, use the SSL workaround: `ssl._create_default_https_context = ssl._create_unverified_context`.

### 3. Install the Skill

Copy the skill file to Claude Code's skills directory:

```powershell
# Create skill directory
New-Item -ItemType Directory -Force -Path "$env:USERPROFILE\.claude\skills\citation-check"

# Copy skill file
Copy-Item "SKILL.md" "$env:USERPROFILE\.claude\skills\citation-check\SKILL.md"
```

After installation, restart Claude Code. The skill will auto-trigger on phrases like:
- "帮我查引这篇论文"
- "查一下引用"
- "citation check"
- "check who cited this paper"

### 4. Prerequisites Checklist

| Requirement | How to Check | How to Install |
|-------------|-------------|----------------|
| Claude Code CLI | `claude --version` | https://code.claude.com |
| Playwright MCP | `claude mcp list` \| grep playwright | `claude mcp add playwright npx @playwright/mcp@latest` |
| Python 3.12+ | `py -3 --version` | https://python.org or `winget install python` |
| PyMuPDF | `py -3 -c "import fitz"` | `py -3 -m pip install pymupdf` |
| openpyxl | `py -3 -c "import openpyxl"` | `py -3 -m pip install openpyxl` |
| PowerShell | Windows default | Built into Windows |
| IEEE Xplore access | Open https://ieeexplore.ieee.org in browser | Institutional subscription required |

### 5. Quick Start

Once everything is installed, start a new Claude Code session and say:

```
帮我查引这篇论文：[paper title]，作者：[authors]
```

Example:
```
帮我查引这篇论文：FSOS-AMC: Few-Shot Open-Set Learning for Automatic Modulation Classification Over Multipath Fading Channels，作者：Hao Zhang
```

---

## Workflow Steps

### Phase 1: Search for the Target Paper

**1.1 Navigate to the search engine**

Use Playwright to navigate to the Google Scholar proxy:
```
mcp__playwright__browser_navigate → https://sc.panda985.com/index.html
```

**1.2 Handle security verification (if needed)**

panda985.com may present a Cloudflare/security verification page. Look for:
- A canvas-based CAPTCHA asking to click on a specific colored shape (e.g., "绿色三角形")
- If encountered, extract the HTML source with `browser_run_code_unsafe` to read `<script>` tags containing `IMAGE_TYPE` and `TARGET`
- Analyze the canvas drawing code to find the target shape's pixel coordinates
- Use `browser_run_code_unsafe` to click the canvas at the correct coordinates
- Click the "提交验证" button to proceed

**Timeout rule**: If the user needs to login or complete verification manually, wait up to 30 seconds. If they don't respond, skip the current paper.

**1.3 Search for the paper**

Type the paper title into the search box and press Enter:
```
mcp__playwright__browser_type → input[type="search"] → paper title
mcp__playwright__browser_press_key → Enter
```

**1.4 Identify the target paper in results**

Take a snapshot and locate the paper. Record:
- Title, authors, journal, year
- The "被引用次数：X" link (citation count)
- The citation URL (contains `cites=XXXXXXXXX`)

If the paper has multiple versions, use the one with the highest citation count.

### Phase 2: View Citing Papers

**2.1 Navigate to citation list**

Click the "被引用次数：X" link to see all citing papers.
```
mcp__playwright__browser_click → a[href*="cites=XXXXXXXXX"]
```

**2.2 Collect all citing paper entries**

Take a snapshot of each page. For each citing paper, record:
- Title (full)
- Authors (as shown)
- Source (journal/conference/arXiv)
- URL (the paper's direct link, usually to ieeexplore.ieee.org or arxiv.org)
- PDF link if available (especially arXiv PDF links)

If there are multiple pages of results, click "下一页" to navigate.

**2.3 Categorize papers by accessibility**

| Type | Accessibility | Download Method |
|------|--------------|-----------------|
| `ieee` | Requires IEEE Xplore subscription | PowerShell with `xpluserinfo` + `ERIGHTS` cookies |
| `arxiv` | Open access | Python `urllib` (with SSL workaround) |
| `techrxiv` | Open but Cloudflare-protected | Browser only (requires Turnstile pass) |
| `proquest` | Requires separate subscription | Manual access |
| `open` | Open access (phwl.org, clemson.edu, etc.) | Python `urllib` |
| `other` | Various | Try browser first |

### Phase 3: Obtain IEEE Cookies (for IEEE papers)

**3.1 Check if user is logged in**

Open any IEEE Xplore paper page in the browser:
```
mcp__playwright__browser_navigate → https://ieeexplore.ieee.org/abstract/document/XXXXXXXXXX/
```

If the page loads with an "Access provided by [institution name]" banner, the user is logged in.

**3.2 Extract cookies**

Use `browser_run_code_unsafe` to get the critical cookies:
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

**3.3 Download IEEE PDFs via PowerShell**

Use the extracted cookies in PowerShell to download PDFs:
```powershell
$cookies = "xpluserinfo=...; ERIGHTS=...; CloudFront-Key-Pair-Id=...; CloudFront-Policy=...; CloudFront-Signature=..."
$headers = @{"Cookie" = $cookies; "User-Agent" = "Mozilla/5.0"}
Invoke-WebRequest -Uri "https://ieeexplore.ieee.org/stampPDF/getPDF.jsp?tp=&arnumber=XXXXXXXXX" -Headers $headers -OutFile "$env:TEMP\ieee_papers\paper_X.pdf" -UseBasicParsing -TimeoutSec 60
```

The `arnumber` is extracted from the IEEE URL: `https://ieeexplore.ieee.org/abstract/document/XXXXXXXXX/`

**IMPORTANT**: IEEE cookies expire. If you get an "Error Code: 418" response, the cookies need to be refreshed. Re-extract them from the browser (which maintains the session).

### Phase 4: Download Non-IEEE Papers

**4.1 arXiv papers**

Use Python with SSL workaround (required for some corporate networks):
```python
import urllib.request, ssl
ssl._create_default_https_context = ssl._create_unverified_context
req = urllib.request.Request(f'https://arxiv.org/pdf/{arxiv_id}', headers={'User-Agent': 'Mozilla/5.0'})
with urllib.request.urlopen(req, timeout=60) as f:
    data = f.read()
```

Save to: `$env:TEMP\paper_{id}.pdf`

**4.2 Other open-access papers**

Try PowerShell first:
```powershell
Invoke-WebRequest -Uri "URL" -OutFile "$env:TEMP\paper_X.pdf" -UseBasicParsing -TimeoutSec 60
```

If that fails (403/empty file), try through the Playwright browser. Some sites (like TechRxiv, Clemson) use Cloudflare bot detection that only allows browser-based access.

**4.3 TechRxiv papers (Cloudflare-protected)**

These require browser access:
1. Navigate to the PDF URL in the browser
2. Wait for Cloudflare Turnstile verification to complete
3. Download the PDF via browser fetch:
```javascript
const res = await fetch(window.location.href);
const buffer = await res.arrayBuffer();
// Save via base64 encoding (large files may be impractical)
```

### Phase 5: Extract Citation Contexts

Use PyMuPDF to extract text and search for the target paper's citation:

```python
import fitz, re

doc = fitz.open(pdf_path)
full_text = ''
for page in doc:
    full_text += page.get_text()
doc.close()

lines = full_text.split('\n')

# 1. Find the reference entry
ref_num = None
ref_entry = ''
for i, line in enumerate(lines):
    if re.search(r'FSOS|few.shot.open.set|target_paper_keywords', line, re.IGNORECASE):
        m = re.search(r'\[(\d+)\]', line)
        if m:
            ref_num = m.group(1)
            # Get full multi-line reference entry
            entry_lines = [line]
            j = i + 1
            while j < len(lines) and not re.match(r'^\s*\[\d+\]', lines[j]) and lines[j].strip():
                entry_lines.append(lines[j])
                j += 1
            ref_entry = ' '.join(entry_lines).strip()
            break

# 2. Find in-text citations (before References section)
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

**Important**: When printing to stdout, encode as UTF-8 to handle special characters:
```python
sys.stdout = io.TextIOWrapper(sys.stdout.buffer, encoding='utf-8')
```

**Common pitfalls**:
- Some PDFs have encrypted streams - use `fitz.open()` which handles most cases
- Reference numbering varies: some use `[1]`, others use `(1)` or superscript
- The target paper may be referenced by arXiv ID instead of journal publication
- Chinese/author names may have typos in reference entries (e.g., "Yuen, Q. W. C." instead of separate Wu and Yuen)

### Phase 6: Lookup Author Professional Titles

**6.1 Extract titles from PDF author bios**

Many IEEE papers have author biography sections at the end. Search for:
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

**6.2 Web search for key authors**

For prominent authors and first/last authors, use WebSearch:
```
WebSearch → "[author name] IEEE Fellow [institution] researcher"
WebSearch → "[author name] Chinese Academy Engineering academician"
```

**6.3 Key title hierarchy (for Chinese academic context)**:
| Level | Titles |
|-------|--------|
| Academy | 中国工程院院士 (CAE Academician), 中国科学院院士 (CAS Academician) |
| International Fellow | IEEE Fellow, IET Fellow, AAIA Fellow, Optica Fellow, SPIE Fellow, SAEng Fellow |
| Senior Member | IEEE Senior Member |
| Major Honors | 长江学者特聘教授, 国家杰青, 国家优青, NSF CAREER Award |
| Member | IEEE Member, CCF Member |

**6.4 Known author titles reference** (NUAA team common authors):

| Author | Key Titles |
|--------|-----------|
| Qihui Wu (吴启晖) | IEEE Fellow, IET Fellow, Changjiang Distinguished Professor, Vice Principal NUAA |
| Fuhui Zhou (周福辉) | IEEE Senior Member, Full Professor NUAA, National Science Fund for Outstanding Youth |
| Chau Yuen | IEEE Fellow (2021), AAIA Fellow, Provost's Chair NTU Singapore, Highly Cited Researcher |
| Kai-Kuang Ma (马凯光) | IEEE Life Fellow, Fellow Singapore Academy of Engineering, Distinguished Professor NUAA |
| Shilong Pan | IEEE Fellow, Optica Fellow, SPIE Fellow, IET Fellow, Professor NUAA |
| Xiaoniu Yang (杨小牛) | 中国工程院院士 (2013), Fellow Chinese Institute of Electronics, CETC Chief Scientist |
| Hao Zhang | IEEE Member, CCF Member, Postdoc NUAA |
| Philip H.W. Leong | IEEE Fellow, Professor University of Sydney |
| Mérouane Debbah | IEEE Fellow (2015), WWRF Fellow, EURASIP Fellow, AAIA Fellow |

### Phase 7: Generate Excel Report

**7.1 Required columns**:

| Column | Content |
|--------|---------|
| 序号 | Sequence number |
| 被引文献 | Full title of the cited paper |
| 被引文献作者 | Authors with professional titles |
| 被引文献期刊/发表时间 | Journal name, volume, pages, year + JCR zone + IF |
| 引用文献 | Full title of the citing paper |
| 引用文献作者 | Authors with professional titles (e.g., "John Smith (IEEE Fellow, IET Fellow)") |
| 引用文献来源 | Journal/conference name and publication details |
| 引用文献发表时间 | Publication date (extract from PDF metadata or source page) |
| 引用文献网站地址 | Direct URL to the paper |
| 引用文献中参考文献编号 | The reference number used in the citing paper, e.g., "[41]" |
| 引用时的描述/论述 | Comprehensive description including: Chinese summary + original English context + reference entry |

**7.2 Excel formatting**:

Use `openpyxl` with:
- Header row: blue background (#2F5496), white bold text
- Wrap text, vertical-align top for all cells
- Thin borders on all cells
- Freeze top row
- Auto-filter on header row
- Column widths appropriate for content (title columns wider, number columns narrower)
- Font: 微软雅黑 (Microsoft YaHei), size 10 for content, 11 bold for headers

**7.3 Multi-sheet structure**:

- **Sheet 1 "引用详情"**: The main data table
- **Sheet 2 "汇总统计"**: Summary statistics including:
  - Target paper info
  - Total citation count
  - Number of analyzed vs. inaccessible papers
  - Temporal distribution of citing papers
  - Citation type distribution (detailed description vs. passing reference vs. reference-only)
  - Author title summary

**7.4 Save location**:

Save the Excel file in a dedicated folder per paper:
```
E:\平台文件\杰青文件查引\{FirstAuthor}_{ShortTitle}_查引\
```

### Phase 8: Organize Project Files

Create a clean folder structure for each paper:

```
{Author}_{ShortTitle}_查引/
├── {Author}_{ShortTitle}_引用分析报告.xlsx    ← Final Excel report
├── 引用文献PDF/                                ← Downloaded citing paper PDFs
│   ├── 01_FirstAuthor_Short_Title_Source.pdf
│   └── ...
└── 中间文件/                                   ← Scripts and intermediate data
    ├── generate_excel.py
    ├── citation_results.json
    └── ...
```

Clean up temporary files after completion:
```powershell
Remove-Item "$env:TEMP\ieee_papers" -Recurse -Force
Remove-Item "$env:TEMP\paper_*.pdf" -Force
```

### Phase 9: Edge Cases and Troubleshooting

| Problem | Solution |
|---------|----------|
| IEEE cookie expired (Error 418) | Re-extract cookies from browser, which maintains live session |
| Python SSL error | Use `ssl._create_default_https_context = ssl._create_unverified_context` |
| Cloudflare Turnstile blocks download | Download through Playwright browser with `fetch()` and base64 encoding |
| 0-byte PDF download | Site blocks non-browser requests; try browser-based download |
| PDF text extraction returns garbage | PDF may use Type 3 fonts or scanned images; skip and note |
| Reference number not found in text | Try alternative search patterns (arXiv ID, author names, partial title) |
| PowerShell variable conflict | `$pid` is read-only in PS 5.1; use `$paperId` or other variable name |
| Python UnicodeEncodeError with GBK | Wrap stdout: `sys.stdout = io.TextIOWrapper(sys.stdout.buffer, encoding='utf-8')` |
| pip SSL certificate error | Add `--trusted-host pypi.org --trusted-host files.pythonhosted.org` |
| Playwright MCP not responding | Run `claude mcp list` to check status; restart Claude Code if needed |

### Quick Reference: Script Templates

**Template for batch IEEE PDF download:**
```powershell
$cookies = "xpluserinfo=...; ERIGHTS=..."
$headers = @{"Cookie" = $cookies; "User-Agent" = "Mozilla/5.0"}
$outDir = "$env:TEMP\ieee_papers"; New-Item -Force -Type Directory $outDir | Out-Null
$papers = @(@{id=1; arnumber="XXXXXXXX"})
foreach ($p in $papers) {
    Invoke-WebRequest -Uri "https://ieeexplore.ieee.org/stampPDF/getPDF.jsp?tp=&arnumber=$($p.arnumber)" -Headers $headers -OutFile "$outDir\paper_$($p.id).pdf" -UseBasicParsing -TimeoutSec 60
}
```

**Template for batch citation extraction:**
```python
import fitz, re, os, json, sys, io
sys.stdout = io.TextIOWrapper(sys.stdout.buffer, encoding='utf-8')

def find_citation(pdf_path, target_keywords):
    doc = fitz.open(pdf_path)
    text = ''.join(page.get_text() for page in doc)
    doc.close()
    lines = text.split('\n')
    # ... reference finding logic (see Phase 5) ...
    return {'ref_num': ref_num, 'ref_entry': ref_entry, 'in_text': in_text}

results = []
for pid, path in pdf_map.items():
    cit = find_citation(path, ['target', 'keywords'])
    results.append({'id': pid, **cit})
```

### Important Notes

1. **The user's institutional access is critical**: IEEE Xplore papers require cookies from an authenticated browser session. If the user hasn't logged in, ask them to do so first.

2. **Cookie lifetime**: IEEE cookies expire after ~1-2 hours. If downloads suddenly start failing, re-extract cookies from the browser.

3. **Respect rate limits**: Don't hammer Google Scholar with rapid requests. Use pauses between searches if doing many papers in sequence.

4. **Always verify extracted data**: PDF text extraction can be imperfect. Cross-reference with Google Scholar listing when possible.

5. **Non-English papers**: Papers in Chinese (like 电子与信息学报) may use different author name formats. Use both English and Chinese search patterns.

6. **Author title verification**: PDF author bios are the most authoritative source. Web search results may be outdated. When in conflict, trust the PDF bio (which is from the published paper itself).
