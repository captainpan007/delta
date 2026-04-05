# CLAUDE.md — Delta v2

## 项目定义

Delta是Pan的私人作战系统，一家由agent构成的公司。
本质：自动把全球市场信号转化成Pan可以直接行动的决策。
交付物：不是产品，不是信息，是客户状态的变化量（Δ）。
唯一客户：Pan本人。

## 核心原则

- NEVER STOP：系统在Pan睡觉时持续运转
- 不预设市场：跟着信号走，数据说了算
- 不重复发明轮子：优先组合现有最优agent
- 最小闭环优先：先跑通情报+分析，再扩展

## 协议架构

MCP（垂直）：每个agent连接自己的工具和数据源
A2A（水平）：agent之间互相通信和协作
两者互补，构成Delta的神经系统

## 六个筛选标准

1. 信号可信：伪造成本高（付费>招聘>搜索>差评）
2. 需求真实：付费已存在 + 抱怨持续并存
3. 市场够大：竞品融资+ARR+搜索量+招聘+替代方案
4. 用户有钱有意愿：竞品定价+职业收入+抱怨类型
5. 竞品有缺口：差评内容+更新频率（降权参考）
6. 流程可闭环：起点终点清晰+结果可归因

## 七个部门及其现有最优Agent

| 部门 | 现有最优 | 接入方式 |
|---|---|---|
| 情报 | Firecrawl | MCP原生支持Claude |
| 分析 | Claude Sonnet | 内置 |
| 验证 | 人工节点 | Pan拍板 |
| 建造 | Claude Code | 内置 |
| 销售 | Artisan/Clay | API适配层 |
| 交付 | 产品本身 | 取决于产品 |
| 学习 | autoresearch模式 | 自建 |

## 人的角色（董事会）

只在三个节点出现：
1. 押注：看报告，决定要不要做
2. 审批验证：验证报告后决定进不进建造
3. 处理异常：正常运转不需要在场

## 报告格式

每次有高分候选时推送，结构：
- 机会描述（是什么，谁有这个痛苦）
- 六维评分卡（每项得分+原始证据）
- 针对Pan的执行建议（第一步做什么，成功标准是什么）
Pan回复"做"即启动下一步

## 技术栈

编排层：Claude SDK（主agent + 子agent）
采集层：Firecrawl（主）+ Crawl4AI（备）
协议层：MCP（工具接入）+ A2A（agent协作，v0.3注意版本稳定性）
记忆层：PostgreSQL + Redis，后期加RAG
观测层：Langfuse（Day 1接入）
人格层：七个部门各自独立prompt
部署层：Railway + Cloudflare

## 开发顺序

第一阶段（最小闭环，目标3-7天）：
- Firecrawl接入G2差评 + Reddit抱怨
- Claude分析部门用六标准打分
- 生成报告推送Pan
- Pan确认/否决，记录结果

第二阶段（验证闭环）：
- 自动生成落地页或冷邮件
- 收集真实回应信号

第三阶段（建造接入）：
- Claude Code自动执行开发

第四阶段（销售接入）：
- Artisan/Clay API适配

第五阶段（学习闭环）：
- autoresearch模式迭代prompt和权重

## autoresearch在Delta中的位置

学习部门的运作方式：
可修改文件：七个部门prompt + 六标准权重配置
可量化指标：押注成功率、验证回复率、销售转化率
自动验证：历史结果做测试集，自动打分
NEVER STOP：系统持续自我进化

## 参考项目

- GPT Researcher：情报层参考实现
- karpathy/autoresearch：学习部门模式
- agency-agents：prompt写法参考
- a2aproject/A2A：agent协作协议
- Firecrawl：采集层
- Langfuse：观测层

## 开发规则

1. /plan before execute
2. 一个session一个任务
3. Opus规划，Sonnet执行
4. 每步commit + push
5. 用/compact压缩历史
6. Prompt以"Be concise, no acknowledgment"开头
7. ⚠️ A2A目前v0.3，注意版本变更风险

## Skill routing

When the user's request matches an available skill, ALWAYS invoke it using the Skill
tool as your FIRST action. Do NOT answer directly, do NOT use other tools first.
The skill has specialized workflows that produce better results than ad-hoc answers.

Key routing rules:
- Product ideas, "is this worth building", brainstorming → invoke office-hours
- Bugs, errors, "why is this broken", 500 errors → invoke investigate
- Ship, deploy, push, create PR → invoke ship
- QA, test the site, find bugs → invoke qa
- Code review, check my diff → invoke review
- Update docs after shipping → invoke document-release
- Weekly retro → invoke retro
- Design system, brand → invoke design-consultation
- Visual audit, design polish → invoke design-review
- Architecture review → invoke plan-eng-review
- Save progress, checkpoint, resume → invoke checkpoint
- Code quality, health check → invoke health