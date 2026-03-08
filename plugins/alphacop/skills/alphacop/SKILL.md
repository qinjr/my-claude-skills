---
name: alphacop
description: |
  AlphaCopilot 项目专属 skill。
  TRIGGER when: 用户提到 AlphaCopilot、A股信号、交易信号、飞书推送、Tushare、财联社爬虫，
  或要求添加/调试 scraper / pipeline / scorer / notifier 等模块。
  ALWAYS trigger when: 用户在 AlphaCopilot 项目目录下工作且问到系统架构、配置、调试方法。
---

# AlphaCopilot 开发助手

AlphaCopilot 是一个 A 股新闻驱动交易信号系统：
**新闻抓取 (Scout) → LLM 提取线索 (Processor) → 技术面验证 → 评分 → 飞书推送 (Notifier)**

## 项目根目录

```
/Users/qinjiarui/Documents/side_projs/AlphaCopilot/
```

---

## 架构总览

```
main.py                          # 入口：初始化DB → 启动Processor(后台) → 启动Scout(阻塞)
config/scrapers.yaml             # 所有 scraper 的开关/参数配置
src/
  config.py                      # Settings (pydantic-settings, 读 .env)
  database/
    models.py                    # NewsItem + TradeSignal ORM 模型
    connection.py                # SQLite engine + get_db() context manager
  scout/
    scheduler.py                 # ScoutScheduler (BlockingScheduler)
    deduplicator.py              # 基于DB的去重（无Redis）
    scraper_registry.py          # 注册中心（动态加载 scrapers）
    scrapers/
      cls_scraper.py             # 财联社电报
      rss_scraper.py             # RSS/Atom feeds (feedparser)
      reddit_scraper.py          # Reddit (PRAW)
  processor/
    pipeline.py                  # 主流程编排
    extractor.py                 # DeepSeek LLM 提取线索
    llm_client.py                # OpenAI-compatible chat → JSON
    tech.py                      # 技术指标 (Tushare A股 / Stooq 美港股)
    scorer.py                    # 纯函数评分
  notifier/
    feishu.py                    # 飞书 webhook 推送
```

---

## 数据库结构（SQLite: alphacop.db）

### news_items
| 字段 | 类型 | 说明 |
|------|------|------|
| id | Integer PK | |
| source | String | 信源名称（cls_telegraph / bloomberg_markets / ...）|
| source_tier | Integer | 信源等级 1-7（1=顶级）|
| title | String | 标题 |
| body | Text | 正文 |
| url | String | 链接（唯一索引，用于去重）|
| fetched_at | DateTime | 入库时间 |
| is_processed | Integer | 0=待处理 1=已处理 |
| extra_meta | Text | JSON 字符串（附加元数据）|

### trade_signals
| 字段 | 类型 | 说明 |
|------|------|------|
| id | Integer PK | |
| news_item_id | Integer FK | 来源新闻 |
| ticker | String | 股票代码（见格式规范）|
| direction | String | long/short/neutral |
| event_type | String | 事件类型枚举 |
| logic_summary | String | 交易逻辑（≤60字）|
| risk_notes | String | 风险提示（≤50字）|
| score_total | Float | 综合得分（0-100）|
| score_source | Float | 信源分（0-30）|
| score_logic | Float | 逻辑分（0-25）|
| score_tech | Float | 技术分（0-20）|
| score_penalty | Float | 惩罚分（0-35）|
| tech_snapshot | Text | JSON 技术指标快照 |
| base_price | Float | 信号生成时收盘价 |
| is_sent | Integer | 0=未推送 1=已推送 |
| created_at | DateTime | 生成时间 |

---

## 评分公式

```
Final Score = 50
            + SourceQuality  (0~+30)  信源等级决定
            + LogicStrength  (0~+25)  事件类型分 × LLM置信度
            + TechResonance  (0~+20)  RSI/MA/量比规则
            - Penalty        (0~-35)  RSI超买/偏离均线/量能萎缩

≥70 分 → 立即推送飞书
```

**信源等级分：** Tier1=30, Tier2=25, Tier3=20, Tier4=12, Tier5=6

**事件类型分（最高25分×置信度）：**
major_order(25), earnings_beat(22), ma(20), major_client(18),
product_launch(15), rating_upgrade(12), fundraising(10), buyback(8)

---

## Ticker 格式规范

| 市场 | 格式 | 例子 | 数据源 |
|------|------|------|--------|
| A股 | `NNNNNN.SH` / `NNNNNN.SZ` | `600519.SH` | Tushare Pro |
| 港股 | `NNNNN.HK`（无前导零！）| `700.HK` | Stooq CSV |
| 美股 | 标准代码 | `AAPL`, `NVDA` | Stooq CSV |

Stooq 港股转换：`0700.HK` → `700.hk`（strip leading zeros，lowercase）

---

## 关键配置 (.env)

```bash
DATABASE_URL=sqlite:///alphacop.db
LLM_API_KEY=sk-...          # DeepSeek API key
LLM_BASE_URL=https://api.deepseek.com/v1
LLM_MODEL=deepseek-chat
TUSHARE_TOKEN=...           # Tushare Pro token
FEISHU_WEBHOOK_URL=https://open.feishu.cn/open-apis/bot/v2/hook/...
SCOUT_BACKFILL_ON_STARTUP=false
LOG_LEVEL=INFO
```

---

## 常用运行命令

```bash
cd /Users/qinjiarui/Documents/side_projs/AlphaCopilot

# 启动完整系统（阻塞）
python main.py

# 启动并立即跑一次所有 scraper
python main.py --once

# 手动触发 pipeline（调试）
python -c "from src.processor.pipeline import run_pipeline; run_pipeline()"

# 查看最新信号
python -c "
from src.database.connection import get_db
from src.database.models import TradeSignal
with get_db() as db:
    sigs = db.query(TradeSignal).order_by(TradeSignal.created_at.desc()).limit(5).all()
    for s in sigs: print(s.ticker, s.score_total, s.direction, s.logic_summary)
"

# 查看未处理新闻数量
python -c "
from src.database.connection import get_db
from src.database.models import NewsItem
with get_db() as db:
    n = db.query(NewsItem).filter(NewsItem.is_processed==0).count()
    print(f'Unprocessed: {n}')
"
```

---

## 添加新 Scraper 的步骤

1. 在 `src/scout/scrapers/` 下新建 `xxx_scraper.py`
2. 继承 `BaseScraper`，实现 `fetch() -> list[RawSignal]`
3. 用 `@ScraperRegistry.register("xxx")` 装饰类
4. 在 `src/scout/scrapers/__init__.py` 中 import（触发注册）
5. 在 `config/scrapers.yaml` 添加配置块（`enabled: true`）

```python
from src.scout.base_scraper import BaseScraper, RawSignal
from src.scout.scraper_registry import ScraperRegistry

@ScraperRegistry.register("my_source")
class MySourceScraper(BaseScraper):
    def fetch(self) -> list[RawSignal]:
        ...
        return [RawSignal(
            source="my_source",
            source_tier=3,
            title="...",
            body="...",
            url="https://...",
        )]
```

---

## 技术栈

- Python 3.11 — **注意：pandas-ta 不兼容，改用 `ta>=0.11.0`**
- SQLAlchemy ORM + SQLite
- APScheduler 3.x（BackgroundScheduler + BlockingScheduler）
- pydantic-settings（配置管理）
- feedparser（RSS）、PRAW（Reddit）
- Tushare Pro（A股日线）、Stooq CSV（美港股，免费无限流）
- DeepSeek API（OpenAI-compatible）

---

## 常见问题

**Q: 如何测试单只股票的技术快照？**
```python
from src.processor.tech import get_tech_snapshot
print(get_tech_snapshot("600519.SH"))   # A股
print(get_tech_snapshot("AAPL"))        # 美股
print(get_tech_snapshot("700.HK"))      # 港股
```

**Q: 如何手动测试 LLM 提取？**
```python
from src.processor.extractor import extract_leads
result = extract_leads("宁德时代签署大订单", "宁德时代今日与大众签署500亿供应合同...")
print(result)
```

**Q: extra_meta 怎么存？**
```python
import json
item.extra_meta = json.dumps({"key": "value"}, ensure_ascii=False)
# 读取
meta = json.loads(item.extra_meta) if item.extra_meta else {}
```

**Q: 如何修改推送阈值？**
在 `src/processor/pipeline.py` 找到 `if score_result["total"] >= 70:` 修改即可。
