# Citation Check Skill (查引)

A Claude Code skill for automated academic citation analysis.

## What it does

This skill automates the complete workflow of checking which papers cite a target paper:

1. **Search** for the target paper on Google Scholar (via panda985.com)
2. **Find** all papers that cite it
3. **Download** PDFs of citing papers (IEEE Xplore, arXiv, open-access)
4. **Extract** exact citation contexts from each PDF
5. **Lookup** author professional titles (IEEE Fellow, Academy memberships, etc.)
6. **Generate** a comprehensive Excel report with full citation details

## Prerequisites

- Claude Code with Playwright MCP server (`@playwright/mcp`)
- Python 3.12+ with `pymupdf` and `openpyxl`
- PowerShell (Windows)
- IEEE Xplore institutional subscription (for IEEE papers)

## Installation

```bash
# Install to Claude Code skills directory
cp SKILL.md ~/.claude/skills/citation-check/SKILL.md

# Install Python dependencies
pip install pymupdf openpyxl
```

## Usage

In Claude Code, simply say:
- "帮我查引这篇论文：[paper title]"
- "/citation-check"
- "Check citations of [paper title] by [authors]"

The skill will trigger automatically on phrases like "查引", "引用", "被引用", "citation search", "cited by".

## Output

Each paper gets a dedicated folder:
```
{Author}_{ShortTitle}_查引/
├── {Author}_{ShortTitle}_引用分析报告.xlsx    ← Excel report
├── 引用文献PDF/                                ← Downloaded PDFs
└── 中间文件/                                   ← Scripts and data
```

The Excel report contains:
- Complete citing paper information
- Author professional titles (IEEE Fellow, Academy memberships)
- Exact citation context (original text + Chinese summary)
- Publication dates extracted from PDF metadata
- Summary statistics

## Known Issues

- **IEEE cookies expire** after ~1-2 hours; re-authentication needed
- **Cloudflare-protected sites** (TechRxiv) require browser-based access
- **ProQuest** papers require separate subscription
- Some PDFs with Type 3 fonts may not extract text correctly

## Author

Created for automated citation analysis workflow.

## License

MIT
