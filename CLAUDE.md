# CLAUDE.md — Delta v1 实施指南

## 项目定义

Delta是Pan的私人市场机会发现引擎。
自动扫描G2差评和Reddit抱怨，用六个标准评分，推送报告给Pan。
唯一客户：Pan本人（管理背景，非技术）。

## 当前状态

- 设计文档已批准：`~/.gstack/projects/delta/Administrator-master-design-20260404-183640.md`
- CEO计划已批准：`~/.gstack/projects/delta/ceo-plans/2026-04-05-delta-v1-single-script.md`
- CEO Review: CLEAN | Eng Review: CLEAN | 可以开始实施

## v1 架构（已审批）

**单服务架构：Flask + APScheduler 在一个进程里，部署到Railway。**

不要用：多Agent架构、PostgreSQL、Langfuse、Redis、MCP/A2A协议。这些是v2的事。

```
[Railway Web Service] ──always-on──▶ [Flask App]
    │                                     │
    ├── APScheduler (每天06:00 UTC)       ├── GET /decide?id=X&d=do
    │   └── run_pipeline()                │   └── 显示确认页面
    │       ├── backup_sqlite()           ├── POST /decide
    │       ├── fetch_g2()                │   └── 记录决策到SQLite
    │       ├── fetch_reddit()            └── GET /health
    │       ├── deduplicate()
    │       ├── score_opportunities()  ◀── Claude Sonnet API
    │       ├── calculate_trends()
    │       ├── generate_briefs()      ◀── Claude Sonnet API (仅Top 5)
    │       ├── send_email()           ◀── SMTP
    │       └── record_health()
    │
    └── 启动时检查：今天是否已跑过？没有则立即补跑
```

## 关键设计决策（不可更改）

1. **单文件 main.py** — 所有逻辑在一个文件里，v1不拆分模块
2. **SQLite** — 存储在Railway persistent volume，不用PostgreSQL
3. **无阈值** — 发送所有机会按分数排序，不设最低分数线
4. **Top 5简报** — 只给前5名生成详细200字简报，其余只显示分数+一句话
5. **邮件点击决策** — GET显示确认页，POST记录决策（防止邮件bot误触发）
6. **提示词外置** — `prompts/scoring.txt` 和 `prompts/brief.txt`，不硬编码
7. **评分标准外置** — `scoring_criteria.yaml` 定义六标准权重，Pan可编辑
8. **启动时补跑** — Flask启动时检查今天是否已运行，没有则立即执行pipeline
9. **每次运行前备份SQLite** — 复制到 `delta_backup.db`

## 六个评分标准

| # | id | 名称 | 初始权重 | 评分 |
|---|---|---|---|---|
| 1 | signal_credibility | 信号可信 | 1.0 | 0-10 |
| 2 | real_demand | 需求真实 | 1.0 | 0-10 |
| 3 | market_size | 市场够大 | 1.0 | 0-10 |
| 4 | willingness_to_pay | 用户有钱有意愿 | 1.0 | 0-10 |
| 5 | competitor_gap | 竞品有缺口 | 1.0 | 0-10 |
| 6 | closeable_flow | 流程可闭环 | 1.0 | 0-10 |

总分 = [Σ(score_i × weight_i) / Σ(weight_i)] × 10，满分100。

## 数据库Schema

```sql
CREATE TABLE raw_signals (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  source TEXT NOT NULL,          -- 'g2' | 'reddit'
  category TEXT NOT NULL,
  title TEXT,
  content TEXT NOT NULL,
  url TEXT UNIQUE,
  author TEXT,
  signal_date TEXT,
  fetched_at TEXT DEFAULT (datetime('now'))
);

CREATE TABLE scored_opportunities (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  opportunity_name TEXT NOT NULL,
  signal_ids TEXT,               -- JSON array of signal IDs
  score_total REAL,
  score_breakdown TEXT,          -- JSON: {"signal_credibility": 7, ...}
  evidence_summary TEXT,
  report_text TEXT,
  trend_data TEXT,               -- JSON: {"last_week": 3, "this_week": 12, "direction": "accelerating"}
  pushed_at TEXT,
  push_status TEXT DEFAULT 'pending',
  scored_at TEXT DEFAULT (datetime('now'))
);

CREATE TABLE pan_decisions (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  opportunity_id INTEGER REFERENCES scored_opportunities(id),
  decision TEXT NOT NULL,        -- 'do' | 'skip'
  notes TEXT,
  decided_at TEXT DEFAULT (datetime('now'))
);

CREATE TABLE pipeline_runs (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  run_date TEXT NOT NULL,
  g2_status TEXT,                -- 'ok' | 'failed' | 'skipped'
  reddit_status TEXT,
  signals_found INTEGER DEFAULT 0,
  opportunities_scored INTEGER DEFAULT 0,
  emails_sent INTEGER DEFAULT 0,
  started_at TEXT DEFAULT (datetime('now')),
  completed_at TEXT
);
```

## 文件结构

```
delta/
├── CLAUDE.md                    # 本文件
├── main.py                      # 所有逻辑：Flask + APScheduler + pipeline
├── scoring_criteria.yaml        # 六个标准的权重配置
├── sources.yaml                 # G2品类 + subreddit列表
├── prompts/
│   ├── scoring.txt              # Claude评分提示词
│   └── brief.txt                # Claude简报生成提示词
├── templates/
│   ├── email_daily.html         # 每日邮件模板
│   ├── email_weekly.html        # 周报模板
│   └── decide_confirm.html      # 决策确认页面
├── requirements.txt             # Python依赖
├── Procfile                     # Railway启动命令
├── .env.example                 # 环境变量模板
└── tests/
    ├── test_fetchers.py
    ├── test_scoring.py
    ├── test_trends.py
    ├── test_email.py
    ├── test_pipeline.py
    └── test_flask.py
```

## 环境变量

```
FIRECRAWL_API_KEY=              # Firecrawl API密钥
ANTHROPIC_API_KEY=              # Claude API密钥
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=                      # Gmail地址
SMTP_PASSWORD=                  # Gmail应用专用密码
NOTIFY_EMAIL=                   # Pan的接收邮箱
DELTA_BASE_URL=                 # Railway部署URL（用于决策链接）
SQLITE_PATH=/data/delta.db      # Railway persistent volume路径
```

## 错误处理规则

- Firecrawl失败 → 标记source_health='failed'，跳过，继续其他源
- Claude API失败 → 等30秒重试1次，仍失败则跳过该批次
- SMTP失败 → 记录到pipeline_runs，下次运行时检查并告警
- SQLite错误 → try/except包裹，失败时尝试发告警邮件
- 未响应报告 → 不阻塞，继续下一周期，不重复推送同一机会

## PRE-BUILD BLOCKER（实施前必须解决）

1. **Firecrawl测试G2** — 拿到API key后先测试能否抓G2差评页面。如果不行，v1只用Reddit
2. **种子扫描范围** — Pan需要定义初始的G2品类和subreddit列表

## 实施顺序

1. 搭建项目骨架（requirements.txt, .env, main.py Flask骨架）
2. 实现SQLite初始化 + 数据模型
3. 实现fetch_reddit()（Reddit API更可靠，先做）
4. 实现fetch_g2()（Firecrawl，可能失败）
5. 实现score_opportunities()（Claude API + 外置prompt）
6. 实现calculate_trends()
7. 实现generate_briefs()（仅Top 5）
8. 实现send_email()（每日排序 + 周日摘要）
9. 实现Flask /decide端点（GET确认页 + POST记录）
10. 实现APScheduler定时 + 启动补跑逻辑
11. 实现SQLite备份
12. 写测试
13. 部署到Railway

## 开发规则

1. 用中文沟通
2. 每步commit + push
3. 先写能跑的代码，再优化
4. 不要过度架构——这是v1，一个文件搞定
5. 所有API key从环境变量读取，绝不硬编码

## Skill routing

When the user's request matches an available skill, ALWAYS invoke it using the Skill
tool as your FIRST action. Do NOT answer directly, do NOT use other tools first.

Key routing rules:
- Bugs, errors, "why is this broken" → invoke investigate
- Ship, deploy, push, create PR → invoke ship
- QA, test the site, find bugs → invoke qa
- Code review, check my diff → invoke review
- Architecture review → invoke plan-eng-review
- Code quality, health check → invoke health
