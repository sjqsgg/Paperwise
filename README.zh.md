[English](README.md) | [中文](README.zh.md)

# Paperwise — Claude Code 学术论文 Skill

> 把任意论文转成**可在浏览器直接阅读的批注网页** — 双栏布局、彩色逻辑高亮、逐段批注卡片。数据来自 OpenAlex，无需任何 API Key。

![Demo](Demo-cn.png)

## 功能一览

| 命令 | 输出 |
|------|------|
| `/paper find "cognitive load LLM"` | 搜索 OpenAlex → 生成 top N 篇论文的 Digest 浏览卡片 HTML |
| `/paper read /path/to/file.pdf` | 对本地 PDF 生成完整双栏批注 |
| `/paper digest` | 抓取今日 arxiv 新论文，按关键词过滤后批注 |
| `/paper cite [[Author-YYYY-note]]` | 从已有批注生成 APA 7th + BibTeX 引用 |

每篇批注是一个**独立 HTML 文件**，包含：
- **Paper Quality Bar** — 来源、Venue、年份、引用量徽章（🔥 高引 · ⭐ 重要 · ✓ 有效 · 📄 预印本）
- **5色逻辑高亮** — 核心论点 · 关键概念 · 实证证据 · 让步/反驳 · 方法论
- **双栏布局** — 左侧原文，右侧批注卡片
- **底部论证结构总览** — 全文逻辑骨架
- **一键复制按钮** — APA 和 BibTeX 引用直接复制

## 安装

把 `SKILL.md` 复制到项目的 skills 目录：

```
你的项目/
└── .agents/
    └── skills/
        └── paper/
            ├── SKILL.md
            └── references/
                └── template.css   ← 所有 HTML 输出的可自定义 CSS
```

然后在 Claude Code 里运行 `/paper find "你的主题"` 即可，无需其他配置。

## 配置

打开 `SKILL.md`，编辑顶部的 `## Default Config` 区块，所有字段均为可选：

```yaml
output_dir: "./papers"        # HTML 文件保存路径

venues: [CHI, ACL, NeurIPS, ...]   # 用于 Venue 标签识别
keywords: [...]                     # /paper digest 的默认关键词

min_citations: 5              # /paper find 最低引用量过滤
daily_max: 10                 # /paper digest 每次最多论文数

annotation_lang: zh           # zh = 中文批注（默认）| en = 英文批注
                              # 单次覆盖：--lang en

openalex_email: ""            # 可选：加入 OpenAlex Polite Pool（更高限速）
semantic_scholar_api_key: ""  # 可选：启用 Semantic Scholar 作为次要数据源
```

**项目级覆盖** — 在你的项目 `CLAUDE.md` 里添加：
```
paper_output_dir: 30_Research
paper_annotation_lang: zh
```

## 使用方法

### 搜索论文

```
/paper find "检索增强生成"
/paper find "transformer attention" --top 10
/paper find "transformer attention" 10            # 裸数字 = --top 10
/paper find "扩散模型" --source venues    # 仅 config.venues 列表中的期刊/会议
/paper find "LLM agents 2025" --source arxiv  # 仅 arxiv 最新预印本
/paper find "X" --lang en                 # 切换为英文批注

# 本地 PDF 粗读 Digest — 无需联网：
/paper find --local ~/Downloads/papers/   # 扫描文件夹内所有 PDF
/paper find --local a.pdf b.pdf c.pdf     # 指定多个文件
/paper find --local ~/Downloads/ --top 3  # 最多处理 3 个文件
```

输出自动按类型存入子目录：

```
papers/
├── {Author-YYYY-keyword}.html            ← /paper read 输出（平铺）
├── FindResults/
│   ├── find-2026-03-14-1430-cognitive-load-llm.html    ← /paper find
│   └── find-local-2026-03-14-1445-downloads.html       ← /paper find --local
└── PaperDigests/
    └── 2026-03-14-digest.html                           ← /paper digest
```

对同一查询重复执行时，已有 HTML 的论文会标注 `[已有]` 并跳过生成。

### 读本地 PDF

```
/paper read ~/Downloads/paper.pdf
/paper read paper.pdf --context "NLP课第3周"
/paper read paper.pdf --questions "Q1: 方法是什么？ Q2: 与baseline如何对比？"
```

### 每日摘要

```
/paper digest
```

抓取今日 arxiv 新论文（按配置关键词过滤），保存到 `{output_dir}/PaperDigests/YYYY-MM-DD-digest.html`。

### 生成引用

```
/paper cite [[Jin-2025-llm-teachable-agent]]
```

直接输出 APA 7th 和 BibTeX，不保存文件，复制即用。

## `--source` flag

| Flag | 搜索策略 | 排名公式 | 适合场景 |
|------|---------|---------|---------|
| *(默认)* | 两次 OpenAlex（按引用量 + 按年份），合并去重 | `0.5 相关度 + 0.3 log(引用量) + 0.2 时效` | 经典论文 + 新论文兼顾 |
| `--source venues` | 仅 OpenAlex，只保留 config.venues 列表中的期刊/会议 | `0.5 相关度 + 0.5 log(引用量)` | 只要同行评审论文 |
| `--source arxiv` | 仅 arxiv | `0.5 相关度 + 0.5 时效` | 只要最新预印本 |
| `--local /path/` | 仅本地 PDF，无 API 调用 | 字母序，按 `--top N` 截断 | 离线阅读 / 已有下载 |

默认模式会发起**两次** API 请求（分别按引用量和年份排序），保证高引经典和最新论文都有机会出现在结果里。

> **Flag 语法宽松** — 所有 flag 可带或不带 `--`，Claude 均能识别。例：`top 5`、`source venues`、`local /path/`、`questions "..."`。

## 批注模式

**Mode B（默认）：逻辑分析模式。** 右栏批注卡片分析每段的段落功能、逻辑角色、论证技巧或潜在漏洞。零配置即可使用。

**Mode A（`--questions` 触发）：问题驱动模式。** 高亮和批注按你的研究问题组织，适合系统性文献综述。

```
# 内联输入问题：
/paper read paper.pdf --questions "Q1: 方法是什么? Q2: 与 baseline 如何比较?"

# 交互式（Claude 先问你）：
/paper read paper.pdf --questions
# → Claude: "请输入你的阅读问题（每行一个，或逗号分隔，最多6个）："

# Q标签可选，以下格式均支持：
/paper read paper.pdf --questions "方法, 数据集大小, 局限性"
/paper read paper.pdf --questions "方法; 数据集大小; 局限性"
```

**Mode A 输出：**
- 每个问题分配专属颜色（Q1 = 色1，Q2 = 色2，……）
- 批注卡片说明每段回答哪个问题
- 底部 = Q&A Worksheet（每题一张卡：核心论点 + 关键证据 + 局限/反驳）

## 数据来源

| 来源 | 角色 | 是否需要 Key |
|------|------|------------|
| **OpenAlex** | 主力数据源 | 无需 |
| **arxiv** | 每日摘要 + 备用 | 无需 |
| **Semantic Scholar** | 可选次要来源 | 免费申请 |

## 环境要求

- Claude Code
- 网络访问（OpenAlex / arxiv API）— `--local` 和 `read` 命令无需联网
- 基础功能无需任何 API Key

## License

MIT © [sjqsgg](https://github.com/sjqsgg)
