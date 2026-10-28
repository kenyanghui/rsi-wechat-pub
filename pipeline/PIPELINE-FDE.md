# FDE 线内容流水线配置（AI 软件工程落地 · 玄龙系统钩子）

> v1.0（2026-10-28 立线）。FDE = Forward Deployed Engineer（前向落地）。
> 本线主题：**AI 软件工程的落地实践**——AI 编程工具与工作流、Agent 系统工程化、研发效能变革。
> **每篇文章文末带「玄龙实战注脚」**：本文主题在玄龙系统（自进化 AI FDE 软件开发底座）上的真实实践呼应。

## 概述

每天 **10:00** 自动执行 FDE 线流水线，从 IMA 知识库**「RSIClaw协作区」**选材，产出 **3 篇**长文（AI 软件工程落地方向，角度互不重复）。**发布公众号账号待配置**：当前产物落盘 `push_status=no_account` 即视为完成，账号配置后重推。到人工闸门即停。

> **与量化线（PIPELINE.md）、教育线（PIPELINE-EDU.md）、廖晓老师线（PIPELINE-FE.md）、心光线完全独立**：产物目录、选题台账、知识库、目标公众号均不同，禁止混用、禁止触碰其他线目录。

## 流水线总览（Step -1 → 五步 → 回写）

```
Step -1 素材采集与知识库喂养（主控执行，最先跑，≤40 分钟）
  → topic（选题，从「RSIClaw协作区」知识库挖素材）
  → writer（写作）
    → qa（质检，评分标准同量化线 qa-rubric.md）
      → format（排版；推送目标账号待配置 → push_status=no_account）
        → 成稿回写（sync_to_ima → RSIClaw协作区「5.公众号文章」）
          → 台账回收（SOURCE-LEDGER-FDE.md 更新）
```

> Step -1 失败/超时**不得阻断**主链；Step -1 与台账回收由主控执行，topic/writer/qa/format 复用现有独立 agent。

## Step -1：素材采集与知识库喂养（主控执行）

**源头库**：IMA「RSIClaw协作区」——**KB id 会轮换**，使用前必须按名称刷新：
`get_addable_knowledge_base_list` 翻页找 name="RSIClaw协作区" 取最新 `ima_kb_*` id（2026-10-06 IMA 全量轮换 kb id 实证，旧 id 报 220004 outdated）。

**六步**：①提兴趣点（AI 编程工具/Agent 工程/AI 编程工作流/研发效能/AI 落地案例域，6-10 个）
②`weixin_search.sh` 按词搜公众号文章（候选池 ≥40；⚠️ Tavily 配额可能 432，失败即降级跳过）
③按 `.imported_urls.txt` 去重 ④按 `config/ima-folders-fde.json` 归类 ⑤`import_urls_to_ima.sh` 分组导入（≥30）
⑥目标文件夹条目数增量验收，缺口重提一次。全程 ≤40 分钟，失败不阻断主链。

**硬边界**：不产文章、不进草稿箱、不写台账；素材不足时允许模型自有知识补充选题（台账标 `model-knowledge`），truth-first 减产。

## 编排可靠性硬规则（与量化线 v1.5.2 同源 + v0.5 实战增补）

1. **已完成短路**：当日 `_status.json` finished:true → 简报结束，不重复采集/产文。
2. **断点续跑跳过采集**：目录存在未 finished → 跳过 Step -1 直接补链；缺哪步补哪步，禁止覆盖重写。
3. **禁止被动等待**：spawn 后同 turn 轮询产物落盘（30s×60，单阶段≤30 分钟），超时补 spawn，再失败写 finished:false 告警。
4. **写后必回读**（loong-workflow verify-external-writes L1）：draft/add → draft/get 验证；IMA 导入按条目数增量核对。
5. **完成标记**：`_status.json` finished:true/false + stages + articles_pushed，看门狗只认磁盘标记。

## 产物目录

```
/root/agents/shared/pipeline-fde/<YYYY-MM-DD>/
├── 00_search/          # Step -1 采集产物
├── p1/ p2/ p3/         # 各含 01_topics.md → 04_publish_queue.md + format/
├── _run.log
└── _status.json        # line: "fde"
```

失败产物移至 `/root/agents/shared/pipeline-fde-failed/<date>_<HHMMSS>/`

## 选题（topic）硬规则

- 查重：读 `SOURCE-LEDGER-FDE.md`（7 天内主素材不复用；累计 ≥2 次换角度；重点规避区直接打回）
- IMA「RSIClaw协作区」挖掘优先（kb id 按名称刷新，见 Step -1）；素材不足允许 web_search（若可用）或模型知识补充，源清单标注来源类型
- 受众：工程师、技术负责人/CTO、AI 落地负责人、对 AI 变革研发感兴趣的技术管理者
- 主题域：AI 软件工程落地、AI 编程工具与工作流、Agent 系统工程化、AI 时代的研发组织与效能
- 3 篇角度互不重复（建议「认知层 / 工程方法层 / 落地心智层」三翼），选题写清「爆款点」

## 质检（qa）规则

- 评分同 `/root/agents/shared/qa-rubric.md`（≥80 入库 / 60-79 打回 / <60 丢弃）
- 合规一票否决：不承诺效果/收益；技术断言须可验证或标注"实践参考"；原创性维度标「参考值」

## 排版（format）与发布【本线专属】

- 只处理 ≥80 分条目
- **发布账号：待配置**（未配置期间：`push_status=no_account`，产物照常落盘；账号就绪后 `--account <alias>` 重推）
- **玄龙实战注脚（本线专属硬规则，每篇必须有）**：文末一段「玄龙实战注脚」——用玄龙系统（自进化 AI FDE 软件开发底座：多 agent 工单驱动开发、RSI 台账自进化、看门狗自愈等真实机制）上与本文主题对应的真实实践收尾。要求：具体、可验证、不夸大（写机制与做法，不编造数据）；自然融入不硬推销；无合适呼应时写"本文实践已在玄龙系统的日常迭代中验证"级别的中性注脚
- 合规：不贬损竞品、不泄露内部凭证/拓扑细节（IP、路径、密钥一律不出现）
- 封面：沿用 mxai 生成，风格贴合工程/技术受众

## 成稿回写（S65）

账号就绪前也执行（成稿先沉淀）：

```bash
IMA_KB_ID=<RSIClaw最新id，按名称刷新> IMA_FOLDER_ID=<5.公众号文章 folderId> \
  bash scripts/ima/sync_to_ima.sh <日期> /root/agents/shared/pipeline-fde/<日期>
```

目录映射见 `config/ima-folders-fde.json` 与 `references/folder-map-fde.md`。

## 明确不做

- ❌ GitHub 归档；❌ 企微获客 CTA / 活码注入；❌ 自动群发；❌ 触碰其他线目录
