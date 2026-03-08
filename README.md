# my-claude-skills

qinjr 的 Claude Code 个人 skill 集合。

## 安装

```bash
# 1. 添加 marketplace
claude plugin marketplace add github:qinjr/my-claude-skills

# 2. 安装需要的 skill
claude plugin install alphacop
```

---

## alphacop

AlphaCopilot 项目开发助手 — 新闻驱动 A股/港股/美股 交易信号系统。

配合 [AlphaCopilot](https://github.com/qinjr/AlphaCopilot) 项目使用。

### 触发方式

```
/alphacop
```

或直接提到 AlphaCopilot、Tushare、财联社、A股信号、pipeline、scorer 等关键词时自动触发。

### 系统架构速览

```
新闻抓取 (Scout)
  ├── cls_telegraph     财联社电报
  ├── rss_scraper       RSS/Atom feeds
  └── reddit_scraper    Reddit
        ↓  news_items (DB)
Pipeline (Processor)
  ├── extractor.py      DeepSeek LLM → ticker + direction + event_type
  ├── tech.py           Tushare (A股) / Stooq CSV (US+HK) → RSI/MA/vol
  └── scorer.py         Score = 50 + source + logic + tech - penalty
        ↓  trade_signals (DB, score ≥ 70)
Notifier → feishu.py   飞书 webhook 推送
```
