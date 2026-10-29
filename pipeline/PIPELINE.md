# 内容流水线配置

> ⚠️ **口径变更（2026-10-29）**：量化线改为**每日 1 篇精品**（单篇 p1/，qa ≥85 入库、70-84 打回重写一次）。
> 本文档中「3 篇 / p1-p3 / ≥80」等历史表述以 cron 编排单（每日 1 篇精品）为准。

## 概述

每天 07:20 自动执行流水线，产出 **3 篇**微信公众号爆款长文（角度互不重复），全部推送草稿箱，由杨辉老师人工选择群发。

## 六步流水线

```
topic（选题）
  → writer（写作）
    → qa（质检）
      → format（排版推送草稿箱）
        → archive（归档 GitHub）
          → sync-ima（同步 IMA 知识库「4.AI生产文章」）
```

前四步为独立 agent；archive 与 sync-ima 由主控执行（无需单独 agent）。

## 编排可靠性硬规则（v1.5.2，优先级最高）

> 背景：2026-09-21/23/24 三次断链，根因是子代理「完成事件(announce)偶发丢失」×主控被动等待无兜底。以下三条为强制契约。

### 1. 禁止被动等待（防 announce 丢失）

spawn 子代理后，主控**不得以「等待完成事件」结束 turn**。必须在**同一 turn 内用 exec 轮询产物文件**（对 announce 丢失免疫；announce 到达可提前推进，但轮询是唯一可信依据）：

```bash
# 示例：等待 3 篇 writer 产物（30s 间隔，单阶段最长 30 分钟）
for i in $(seq 1 60); do
  ok=1
  for p in p1 p2 p3; do
    [ -s "/root/agents/shared/pipeline/<日期>/$p/02_drafts.json" ] || ok=0
  done
  [ "$ok" = 1 ] && break
  sleep 30
done
```

- 超时（30 分钟）：记录 `_run.log`，对该篇**补 spawn 一次**；仍失败则写 `_status.json` 的 `finished:false` 并向用户告警，**不得静默退出**。

### 2. 断点续跑（幂等）

主控每次启动（含看门狗触发的重跑）必须先盘点：

1. 扫描**最近 2 天**的 `/root/agents/shared/pipeline/<日期>/`
2. 检查每天每篇链条：`01_topics.md → 02_drafts.json → 03_qa_scores.md → format/manifest.json`
3. 存在无 `finished:true` 标记的未完成目录 → **从断点续跑**：缺哪步补哪步，已存在的产物直接复用（**禁止覆盖重写**已通过校验的产物）
4. 完成该日全部步骤后写完成标记；若当日全新则正常开跑
5. **续跑优先、不双跑**：若本轮执行了断点续跑，且当天全新目录尚无任何产物，则本轮**不再新开当日全量任务**（留给下一个定时触发，看门狗会持续触发直至 finished:true）——避免一个回合双跑导致重复产出

### 3. 完成标记（看门狗依据）

整轮结束（成功或可控失败）必须更新 `_status.json`：

```json
{
  "date": "<日期>",
  "finished": true,
  "stages": {"p1": "done", "p2": "done", "p3": "format"},
  "articles_pushed": 3,
  "note": "..."
}
```

看门狗**只认** `finished:true`，不信任 cron 表面状态（主控可能"正常结束"但实际断链）。

## 产物目录

```
/root/agents/shared/pipeline/<YYYY-MM-DD>/
├── p1/                 # 第 1 篇（每日 3 篇）
│   ├── 01_topics.md
│   ├── 02_drafts.json
│   ├── 03_qa_scores.md
│   ├── 04_publish_queue.md
│   └── format/         # article.md + manifest.json + imgs/cover.png
├── p2/ …               # 第 2 篇（同上结构）
├── p3/ …               # 第 3 篇（同上结构）
├── format/manifest.json# 合并清单（供归档/同步脚本读取）
├── _run.log            # 执行日志
└── _status.json        # 执行状态
```

失败产物移至 `/root/agents/shared/pipeline-failed/<date>_<HHMMSS>/`

## 选题环节（topic）硬规则

### 强制查重

选题前必须读取 `/root/agents/shared/SOURCE-LEDGER.md`：

1. **7 天内已作主素材**的源 → 不得再次作为主素材
2. **累计作为主素材 ≥2 次**的源 → 换源或标注新角度
3. 命中**重点规避区**的源 → 直接打回重选

### IMA 知识库优先选材

**「AI量化杨老师」知识库**是本流水线的主素材库，2189 条内容，kb_id: `5JU-YyL5WUdMp3ZzS_7M2B6G5XpOB4ofM2rdKkMr3jY=`

topic agent 选材时必须：
1. 优先从该知识库中搜索、挖掘有价值内容作为选题素材
2. 在 `01_topics.md` 末尾附「本轮源清单」，逐条标注：
   - `新增` — 本轮首次使用
   - `复用+<N>` — 累计第 N 次使用
3. 知识库操作路径：使用 `ima-skills` skill，读取 `knowledge-base/SKILL.md`

### 常驻兴趣主题与内容主线（2026-10-13 新增，v1.6）

素材采集（Step 0.7 兴趣点画像）与 topic 选题每轮必须覆盖以下 3 个常驻主题（可与量化三翼融合出题）：

1. **RSI 递归自我改进（Recursive Self-Improvement）**：AI 智能体自我改进/自我进化前沿。⚠️ 检索用「递归自我改进 / Recursive Self-Improvement / AI 自我进化」，勿只搜「RSI」（撞量化指标相对强弱指数）。
2. **逆向工程（Reverse Engineering）**：参考 GitHub 爆火项目 REA（Reverse Engineer Anything）——AI Agent 逆向一切；延伸为「拆解-理解-复刻」方法论（拆产品、拆策略、拆商业模式、拆财富路径）。
3. **AI 财富自由之路**：用 AI 复刻「财务自由」——追踪、拆解、复刻当下可落地的个人财务自由之路（AI 副业、自动化收入、一人公司、AI 量化自营），目标是让人在 AI 时代财富自由。

**内容主线倾斜（写作总纲）**：在**保持量化特征**（数据实证、因子/回测/风控视角、理性投资纪律、必要风险提示）的前提下，选题与写作向「AI 时代财富自由」目标倾斜——把 AI 技术、工具、智能体讲成普通人可追踪、可拆解、可复刻的财富自由路径。「财富自由」只作路径拆解与目标叙事，**严禁收益承诺/保证财富自由类表述**（qa 合规一票否决）。

### 选题输出格式（每天 1 篇）

`01_topics.md` 必须包含：

```markdown
# 公众号选题 <YYYY-MM-DD>

## 本轮源清单
| 源 | 类型 | 用途 | 来源 |
|----|------|------|------|

## 选题列表

### 选题 1：<标题>
- **风格卡**：<受众/调性/长度>
- **核心素材**：<来自哪个知识库条目>
- **角度**：<新角度 vs 复用>
- **爆款点**：<为什么这篇能爆？钩子/痛点/反差>
- **Brief**：<200字以上的完整写作指引>
```

> 每天产出 3 篇，**3 篇角度/素材必须互不重复**（建议「认知层 / 商业层 / 心智层」或「思维层 / 心智层 / 方法论层」三翼），质量优先，宁缺毋滥；选题必须能解释清楚「爆款点」。
> 拼装每篇 topic task 时，除查重台账外，还要把「本日已选定的其他 2 篇角度/源」贴进去，强制 3 篇不撞题。

## 质检（qa）规则

- 评分维度和标准：见 `qa-rubric.md`
- **≥80 分**：入库待发布
- **60-79 分**：打回 writer 重写
- **<60 分**：丢弃
- 合规一票否决：命中合规红线直接不采纳
- 原创性维度：因历史文章库暂搁置，标「参考值」

## 排版（format）规则

- 只处理 ≥80 分过审条目
- 按公众号格式排版
- 推送至微信公众号草稿箱
- 到人工闸门即停，绝不自动群发

## 归档（archive）规则

- 主控调用 `skills/rsi-wechat/scripts/archive_article.sh <日期> <产物目录>`
- 只归档已进草稿箱（≥80 分且带 media_id）的文章，同日多篇自动编号 `articles/<日期>[-n]/`
- 同步到 GitHub `kenyanghui/rsi-wechat`；推送目标固定 `origin master:main`

## 同步 IMA（sync-ima）规则

- 主控调用 `skills/rsi-wechat/scripts/sync_to_ima.sh <日期> [产物目录] [--dry-run]`
- 把文章以 Markdown（media_type=7）同步到 IMA 知识库「AI量化杨老师」的 **「4.AI生产文章」** 文件夹（folder_id: `folder_7507449254775045`）
- 只同步已进草稿箱的文章；失败不阻断主流程（凭证缺失直接跳过）

## IMA 知识库接入（必读）

topic 环节使用 IMA 知识库的标准流程：

1. 读取 skill：`skills/ima-skills/knowledge-base/SKILL.md`
2. 调用接口：`openapi/wiki/v1/search_knowledge`，参数：
   - `knowledge_base_id`：`5JU-YyL5WUdMp3ZzS_7M2B6G5XpOB4ofM2rdKkMr3jY=`
   - `query`：搜索关键词
3. 浏览内容：`openapi/wiki/v1/get_knowledge_list`，参数：
   - `knowledge_base_id`：同上
   - `limit`：50

IMA API Base URL：`https://ima.qq.com/openapi/wiki/v1/`
凭证：从 `~/.config/ima/client_id` 和 `~/.config/ima/api_key` 自动读取
