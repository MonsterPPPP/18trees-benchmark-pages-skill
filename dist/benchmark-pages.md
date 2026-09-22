# benchmark 主页生成器 · benchmark-pages v0.1

> **怎么用**：把本文件**全部内容**作为指令，交给一个**能读写文件、能执行命令**的 AI 工具
> （Claude Code / Codex / Cursor / Windsurf / 自建 agent），然后告诉它你的
> rawdata、论文和分析报告在哪里。
>
> ⚠️ **网页版聊天机器人用不了本 skill。** 它要做的是"读你的实验数据文件、写出一份
> site.yaml 和几张 CSV、再执行 `build_site.py` 渲染出静态站"——没有文件与命令能力的模型
> 只能凭印象编一个 HTML，而这份 skill 的第一原则就是「页面里每个数字都能从 CSV 重算」。
>
> ⚠️ **本文件不含模板与生成器。** 版面模板（`assets/template/`）、表格引擎
> （`assets/vendor/tabulator/`）和渲染脚本（`scripts/build_site.py`）是仓库里的实体文件，
> 单文件版无法携带。本文件是**规则与规范**的完整版：读完它你能知道每一步该产出什么、
> 每个字段该填什么、哪些错误不许犯；但真正生成站点仍需要那个仓库。
>
> 仓库：https://github.com/MonsterPPPP/18trees-benchmark-pages-skill

---


## benchmark-pages · benchmark 主页生成器

**它不是「生成一个漂亮的 HTML」的工具，是把一次评测的所有产物组织成一个可维护的对外门面。**

```
rawdata + 论文 + 分析报告
        │
        ▼
  Phase 0  盘点输入          ← 哪些是数据，哪些是叙事
  Phase 1  锁定评测口径      ← 分母 / invalid / CI / 聚合权重 / 随机基线
  Phase 2  造三张表          ← leaderboard.csv + breakdown_*.csv + items.csv
  Phase 3  写 site.yaml      ← 站点配置（唯一需要人写的东西）
  Phase 4  渲染 + 自检       ← build_site.py（数字对不上就退出）
  Phase 5  本地验证 + 部署   ← http.server 逐页看 → GitHub Pages
        │
        ▼
  主页 index.html + 榜单 leaderboard.html + data/*.json + static/
```

三样东西容易和其他工具搞混，先说清楚：

1. **它不做评测。** 评测在别处跑（lm-eval-harness、自定义脚本、你自己的 run_qa_eval.py），
   本 skill 只消费评测产物。
2. **它不做数据分析。** 显著性检验、消融、item difficulty 都该在论文的分析脚本里算完，
   本 skill 只把结果渲染成表和图。缺数就回去补，不在这里现算。
3. **它不做视觉设计。** 版面沿用学术主页的通用惯例（见 下文《site.yaml 与页面分区规范》），
   目标是「一眼看出这是个正经 benchmark 页」，不是「好看」。

---

## 站在谁的肩膀上

底版派生自 [Academic Project Page Template](https://github.com/eliahuhorwitz/Academic-project-page-template)
（5.2k★，它自己又派生自 [Nerfies](https://nerfies.github.io/) 的项目页），**CC BY-SA 4.0**。
上游 footer 原文：

> You are free to borrow the source code of this website, we just ask that you link back to this page in the footer.

所以这是一次**有明确条件的授权借用**，条件有二：

1. 每个生成出来的站点，footer 必须回链 **Academic Project Page Template** 与 **Nerfies**；
2. 派生部分（模板层）以 **CC BY-SA 4.0** 继续开放。

这两条写进了 P0 铁律（[§3](#3--p0-铁律)），不是可选装饰，删掉就违约。
许可分层与第三方组件清单见仓库根目录 `NOTICE.md`。

---

## 0 · 什么时候用 / 不用

**用**：
- 手上有一个 benchmark（数据集 + 若干模型的评测结果），准备连同论文一起开源
- 已经有 `REPORT.md` 级别的分析报告和 `figures/`，缺的是对外门面
- 已经有榜单页，但加一个模型要手改 HTML —— 需要变成数据驱动
- 想要一个「别人能提交自己结果」的榜单，而不是只展示自己跑的那几个模型

**不用**：
- 只是想给一篇普通论文做个项目页 —— 直接 fork Academic template 更快，本 skill 的榜单部分用不上
- 榜单需要服务端逻辑（用户登录、实时评测提交、需要跑代码验证分数）—— 静态站做不到，
  去用 HF Spaces 或自建后端
- 评测还没跑完 / 数据还在变 —— 先把口径定死再回来，否则每改一次分母整站数字全废

---

## 1 · 输入：先备齐三样

开工前逐项确认存在。缺哪样就说清楚缺哪样，不要拿占位内容糊过去。

| # | 输入 | 典型形态 | 用途 |
|---|------|---------|------|
| 1 | **rawdata** | `experiments/<run>/*.jsonl`、`analysis/canonical/*.csv` | 一切数字的来源 |
| 2 | **论文** | LaTeX 源 / PDF / markdown；或已定稿的标题·作者·摘要 | 主页的 title / authors / abstract / bibtex |
| 3 | **分析报告** | `analysis/REPORT.md`、`analysis/figures/*.png`、`analysis/tables/*.csv` | 叙事段落、图、细分表 |

**最关键的是第 1 项里的 canonical 逐题表**：一行一道题、一列一个模型、值是对错（或预测）。
有了它，所有榜单数字都能重算；没有它，本 skill 只能搬运别人已经算好的数，
最后一定会出现「表格里的数和报告里的数对不上」而没人知道谁对。

字段映射、脏数据形态、以及「只有聚合结果没有逐题结果」时怎么办，
见 下文《输入契约》。

---

## 2 · 六阶段工作流

### Phase 0 · 盘点输入

扫一遍用户给的目录，产出一张**输入清单表**交给用户确认，每行是：
`文件 → 它是什么 → 会用在页面哪个位置 → 有没有问题`。

这一步唯一的目的是：在动手前把「这份数据到底支不支持这个页面」问清楚。
典型发现：报告里引用了但目录里不存在的图；两个 run 的题目数不一样；
canonical 表比报告少了几道题。**全部当场提出来，不要自己消化掉。**

### Phase 1 · 锁定评测口径

从论文和报告里把下面五项抽出来，**写成文字**存进 `data/metrics.md`：

1. **分母** —— 全部题目，还是有效的题目？
2. **invalid / 空答案怎么算** —— 判错并计入分母（保守），还是剔除（会虚高）？
3. **置信区间方法** —— Wilson score？bootstrap？几倍标准差？
4. **多维度聚合权重** —— 各 category 等权 macro？任务族之间再取平均？
5. **随机基线** —— 单选题是 `100/选项数`；开放题必须说明基线怎么定的。

这五项如果整理成 `data/metrics.md`，会原样渲染进榜单页的脚注。
口径改了记得重跑 Phase 2。

> **为什么值得做**：一个 79.5% 的准确率，在「invalid 判错」和「invalid 剔除」
> 两种口径下可能差 10 个百分点。把口径写在页面上，别人引用你的榜单时就不用猜。
> 不想写也可以跳过——本 skill 不强制。

### Phase 2 · 造三张表

| 文件 | 形状 | 用途 |
|------|------|------|
| `data/leaderboard.csv` | 一行一个参赛者，列 = 指标 | 主榜单 |
| `data/breakdown_<维度>.csv` | 一行 = 参赛者 × 维度取值 | 细分热力图 / 子表 |
| `data/items.csv`（可选） | 一行一道题 | item difficulty 页 |

主榜只需要两列：`model` 和主指标列。
`n`（分母）、`correct`、区间列（`*_ci_low` / `*_ci_high` 或一列 `ci95`）都是可选的——
给了就多渲染一列或一条提示，没给就不出现。

生成器会在 `n` 和 `correct` 都在时，用 `correct / n` 反算主指标**提示**不一致。
**这只是提示，页面照常产出**——数据对不对由你判断，见 [§3 铁律](#3--p0-铁律)。

列名、类型、可选列、扩展维度见 下文《榜单数据规范》。

### Phase 3 · 写 site.yaml

站点里唯一需要手写的东西。最小可用版本只有十几行：

```yaml
title: 领域知识四选一 Benchmark
tagline: 2400 道四选一题，测量大模型的专业知识与推理
authors:
  - {name: 你的名字, url: "https://github.com/你的账号"}
venue: arXiv preprint 2026
links:
  paper: pdfs/paper.pdf
  code: "https://github.com/..."
  dataset: "https://huggingface.co/datasets/..."
abstract: |
  ...
leaderboard:
  source: data/leaderboard.csv
  baseline: 25.0
  baseline_label: 随机基线（四选一）
  primary_metric: overall_acc
```

全字段（含 `sections[]` 顺序控制、`submission` 提交指引、SEO/OG 元数据）见
下文《site.yaml 与页面分区规范》。

### Phase 4 · 渲染 + 自检

```bash
python scripts/build_site.py --source <站点源目录> --out _site
```

脚本分两类处理，**分界线是「页面能不能渲染出来」**：

| 类别 | 例子 | 行为 |
|------|------|------|
| **结构性错误** | 缺 `model` 或主指标列、数值列填了非数字、`site.yaml` 引用的文件不存在 | 非零退出，不产出半成品 |
| **数据提示** | `correct / n` 与主指标对不上、各参赛者分母不一致 | 打印警告，**照常产出** |

**数据提示默认不挡。** 本 skill 是网站生成器，不是数据审计工具——
你的数字可能换了分母口径、可能是手工修正过的，这些我们判断不了，所以只提示。
要把它当失败，加 `--strict`（CI 里可以这么配）。

`--check-only` 只跑校验不产出文件，适合放在 CI 里挡结构性问题。

### Phase 5 · 本地验证 + 部署

```bash
python -m http.server 8000 --directory _site
```

**必须真的在浏览器里逐页点一遍**——排序点一下、筛选点一下、图能打开、
PDF 能打开、BibTeX 复制按钮可用、窄屏不塌。生成器跑通 ≠ 页面能用。

部署与「别人怎么提交自己的结果上榜」见 下文《部署与结果提交》。

---

## 3 · P0 铁律

违反任何一条，站点就不许发布。**只有三条。**

1. **数字不许手写进 HTML。** 页面里每一个数字都来自 `data/*.csv`，
   经 `build_site.py` 渲染。想在页面上改个数字 → 改 CSV → 重跑。
   **这条管的是机制，不是内容**：我们不校你的账，只保证页面由数据生成、
   随时可重跑。数据本身对不对，是使用者的事。
2. **每个生成站点的 footer 必须回链** Academic Project Page Template 与 Nerfies。
   这是上游的授权条件，不是致谢装饰。
3. **不许只产出页面而丢掉数据源。** 站点源目录（site.yaml + data/）和生成出的
   `_site/` 都要留在仓库里，前者是真相，后者是产物。

> **本 skill 不规定你的 benchmark 该怎么写。** 基线标不标、口径写不写、开不开提交通道，
> 都是你的选择——对应分区在 `site.yaml` 里按需开启即可。本 skill 只保证：
> 页面上的数字和你的 CSV 一致，以及模板层的许可义务被遵守。

---

## 4 · 让榜单好用的四件事（可选，但值得做）

`site.yaml` 里对应的开关都是可选的。做了这四件事，陌生人才能在 30 秒内看懂你的榜。

1. **这是什么、多难？** → 标题下面一行 tagline +（可选）随机基线锚点
2. **谁最好，好在哪？** → 主榜按指标降序，区间可见，第一名有视觉标记
3. **好在哪些维度？** → 细分表（按 category / 任务族 / 难度分层）
4. **我怎么加进去？** →（可选）「提交结果」段落，说清要交什么文件、提到哪个仓库

写不写由你。写了的，页面会自动渲染成对应分区。

---

## 5 · 资源导览


以下内容全部内联在本文件里，见文末各章。

---

## 6 · 致谢与许可

- 本 skill 的模板层派生自 **Academic Project Page Template**（作者 Eliahu Horwitz）与 **Nerfies**，
  以 **CC BY-SA 4.0** 授权。生成出的站点必须保留 footer 回链。
- 榜单表格使用 **Tabulator**（MIT，© Oliver Folkerd），以 vendored 形式随站点分发。
- 本 skill 的规则文本与生成脚本由 **十八木** 原创，可用 MIT 使用，
  但不得用于移除生成站点的上游回链。
- 完整分层与第三方声明见仓库根目录 `NOTICE.md`。

---

## 输入契约

Phase 0 / Phase 1 用。目标：在写任何 HTML 之前，把「这份数据支不支持这个页面」问清楚。

---

## 1 · 三类输入，各自的判定标准

### 1.1 rawdata（数字的来源）

按可信度从高到低：

| 等级 | 形态 | 能否重算榜单 | 处理 |
|------|------|------------|------|
| **A** | 逐题结果：一行一道题 × 一列一模型，值是 0/1 或 pred | ✅ 全部能算 | 直接进 Phase 1 |
| **B** | 逐题结果长表：一行 = 模型 × 题 × 结果，含 `correct` 字段 | ✅ 全部能算 | 先 pivot 成 A |
| **C** | 只有聚合结果（每模型一行准确率） | ❌ 只能搬运 | 能做榜，但**报告里的细分维度全做不了**；必须在输入清单里标红 |
| **D** | 只有图（PNG/PDF），数字在图上 | ❌ | 属于「无数据」，先回去把数导出来 |

**B 类 pivot 示例**（这是最常见的形态）：

```bash
## 从 experiments/<run>/*.jsonl 造 A 类宽表
python - <<'PY'
import json, glob, csv, collections
rows = collections.defaultdict(dict)
for f in glob.glob("experiments/*/*.jsonl"):
    for line in open(f, encoding="utf-8"):
        r = json.loads(line)
        rows[r["qid"]][r["model"]] = 1 if r["correct"] else 0
models = sorted({m for v in rows.values() for m in v})
with open("data/wide.csv", "w", newline="", encoding="utf-8") as fh:
    w = csv.writer(fh); w.writerow(["question_id", *models])
    for q, v in sorted(rows.items()):
        w.writerow([q, *[v.get(m, "") for m in models]])
PY
```

**C 类怎么办**：能做榜单页，但要在 `site.yaml` 里把 `leaderboard.breakdowns` 留空，
并在页面上明确写「本榜仅为汇总结果，无逐题明细」。不要用插值或估算假装有细分数据。

### 1.2 论文（叙事与元信息的来源）

需要的字段和它们的出处：

| 字段 | 出处 |
|------|------|
| `title` | 论文标题（用论文里的**英文原文**，除非论文本身是中文） |
| `authors` + 机构 | 作者列表；注意共同一作的 `*` 标注 |
| `venue` | 投稿/发布去向 |
| `abstract` | 摘要原文，**不要改写** |
| `bibtex` | 从 arXiv / 会议模板导出的 `.bib`，不要手搓 |
| `links.paper` | 论文 PDF，放进站点源目录的 `pdfs/` |

没有 LaTeX 源时：从 PDF 里抽（`pdftotext -f 1 -l 1`），但**作者列表和单位必须人工核对**——
PDF 抽出来的作者名和上标经常错位。

### 1.3 分析报告（叙事与图）

`REPORT.md` 这类文件的价值不在数字，在**已经写好的结论句**。直接复用，不要重写：

- 报告里的「Key Findings」→ 主页的 findings 卡片
- 报告里的「Limitations」→ 主页的 limitations 段（**不许省**，这是 benchmark 页的可信度来源）
- 报告里的「必须在论文中披露的五项」→ 榜单页脚注
- 报告引用的 `figures/*.png` → gallery

**逐图核对**：报告里 `![](figures/xx.png)` 引用的每张图是否真的存在、是否是最新版
（重跑过分析脚本后老图常留在目录里）。

---

## 2 · 输入清单表（Phase 0 的产出）

写成这样交给用户确认，一行一个文件：

```markdown
| 文件 | 是什么 | 用在哪 | 问题 |
|------|--------|--------|------|
| experiments/v1_native_latest/*.jsonl | A 类逐题结果，6 模型 × 3000 题 | 全部数字的来源 | — |
| analysis/canonical/v1_wide.csv | A 类宽表 | 主榜 + 细分 | 比报告少 12 题，需确认是否有意剔除 |
| analysis/figures/01_leaderboard.png | 主结果图 | gallery | 报告引用的是 .pdf 版，两者是否同版待确认 |
| analysis/REPORT.md | 12 节分析报告 | findings / limitations / 脚注 | — |
| Paper/main.tex | 论文源 | title / abstract / bibtex | 作者单位未确定，缺 ORCID |
```

**不允许出现「待确认」之外的模糊表述。** 每一项要么确认可用，要么明确列为阻塞项。

---

## 3 · 口径抽取（Phase 1 的产出）

从论文的方法节 + 报告的脚注里抽这五项，产出 `data/metrics.md`：

```markdown
## 评测口径

- **分母**：每个模型的全部 3000 条 canonical 记录
- **invalid（空答案）**：判错，并计入分母。理由：空答案反映的是模型未能给出可解析回答，
  剔除会让分数虚高，且在网关超时等基础设施因素下不可比。
- **置信区间**：Wilson score interval，z = 1.96
- **多维度聚合**：先算 Theory 14 个 category 的等权平均、Case 11 个 category 的等权平均，
  再取两者均值（两个任务族各占 50% 权重）
- **随机基线**：25.0（四选一）
```

**证据等级**：每一项都要能指到论文/报告的原文位置。指不到的写「未声明」，
并在输入清单里列为阻塞项——**不要替用户假设一个口径**。

---

## 4 · 常见脏数据与处理

| 现象 | 处理 |
|------|------|
| 同一模型跑了多个 run，题目数不同 | 以 canonical 表为准；在榜单加 `run` 列或分成两张榜，**不要合并** |
| 模型名在不同文件里不一致（少版本号 / 大小写不同） | 建一张 `data/model_aliases.csv` 显式映射，不做模糊匹配 |
| `correct` 字段缺失，只有 `pred` 和 `gold` | 现算 `pred == gold`，但要注意多选题答案顺序 |
| 部分题所有模型都答错 | 保留。item difficulty 页正好需要它们（区分度信息） |
| `reasoning_tokens` / `sec` 有 `null` | 允许。这类列渲染时空着，不要填 0——0 和「没测」是两件事 |
| 一个模型有多次重试（`attempts > 1`） | 默认取最后一次；若论文用的是 best-of-n，必须在口径里写明 |

---

## 榜单数据规范

Phase 2 用。三张表的列名、类型与校验规则。

生成器 `scripts/build_site.py` 按此规范解析。**只有结构性问题会挡住**（缺列、文件不存在），
数据本身合不合理只提示——见下面「校验规则」。

---

## 1 · `data/leaderboard.csv` —— 主榜

一行一个参赛者（一个模型 / 一个 agent / 一个配置）。

### 必填列

| 列名 | 类型 | 说明 |
|------|------|------|
| `model` | string | 显示名。**必须唯一**，不得为空 |
| `<primary_metric>` | float | 主指标，列名由 `site.yaml` 的 `leaderboard.primary_metric` 指定 |

只有这两列是硬要求。其余全部可选。

### 可选列

| 列名 | 类型 | 渲染 |
|------|------|------|
| `n` | int | 概览条上的「题目数」；也用于下面的正确性提示 |
| `correct` | int | **不显示**，只用于正确性提示 |
| `ci_low` / `ci_high` | float | 合并成一列 `95% CI`，形如 `[77.2, 81.6]` |
| `link` | url | **不单独占列**，而是让模型名变成外链 |
| `org` | string | 单独一列 |
| `date` | `YYYY-MM-DD` | 单独一列 |
| `open_weights` | `true`/`false` | 单独一列，原样显示文本 |
| `cost_per_1k` | float | 单独一列，可排序 |
| `notes` | string | 单独一列 |

**任何未在上面列出的列都会原样渲染成一个可排序列**，不需要在配置里声明。
小数按数值排序，其余按字符串。

### 校验规则

分界线是「页面能不能渲染出来」。

**结构性问题 —— 硬失败，非零退出：**

1. 缺 `model` 或主指标列
2. `model` 为空或重复
3. 主指标列里出现非数值（`—`、`N/A` 这类请留空）

**数据提示 —— 打印警告，页面照常产出：**

4. `n` 和 `correct` 都在时，`correct / n` 与主指标对不上
   （百分数容忍 0.05，`unit: ratio` 时容忍 0.0005）
5. `correct` 超出 `[0, n]`
6. `n` 不是正整数
7. 各参赛者的 `n` 不一致（设 `leaderboard.allow_unequal_n: true` 可关掉这条提示）

> **为什么 4–7 不挡：** 本 skill 是网站生成器，不是数据审计工具。
> 你的数字可能换了分母口径、可能经过手工修正，这些我们判断不了。
> 需要把提示也当失败时，跑 `--strict`。

### 数值尺度

默认全部是 **0–100 的百分数**（`79.5` 表示 79.5%）。页面上自动加 `%` 后缀。

确实要用 0–1 比值时，在 `site.yaml` 里设 `leaderboard.unit: ratio`，
生成器会乘 100 后渲染。**同一张表里不许混用两种尺度。**

### 列的显示格式

按列名自动判断，不需要配置：

- `n`、`correct`、`count` 开头或等于 `num_questions` → **整数**，不加 `%`
- 以 `_acc`、`_rate`、`_pct`、`_share` 结尾 → **百分数**，加 `%`
- 其余数值列 → 两位小数，不加后缀

### 数值尺度

默认全部是 **0–100 的百分数**（`79.5` 表示 79.5%）。页面上自动加 `%` 后缀。

确实要用 0–1 比值时，在 `site.yaml` 里设 `leaderboard.unit: ratio`，
生成器会乘 100 后渲染。**同一张表里不许混用两种尺度。**

---

## 2 · `data/breakdown_<维度>.csv` —— 细分表

回答「好在哪些维度」的那张表。

**宽表**，第一列 `model`，其余每列是一个维度取值，值是该维度下的主指标：

```csv
model,事业,健康,财富,婚姻,学业,...
DeepSeek-V4-Pro-0813,72.4,68.1,75.9,70.2,77.0,...
GLM-5.3,69.8,66.5,73.1,68.9,74.2,...
```

- 列名即维度取值，直接渲染成表头，**不要**在 CSV 里写中文标点分隔
- 缺测的格子留空，渲染成灰色「—」，**不要填 0**
- 列顺序 = 渲染顺序。想让某个维度排前面，就在 CSV 里排前面
- 维度取值超过 20 个时，页面自动横向滚动 + 冻结 `model` 列

渲染形态：带色阶的表格（每列独立归一化，最差→最好 由浅到深）。
同一行内跨列比较无意义，**色阶必须按列算**，否则列间难度差异会被掩盖。

### 分会话拆分

同一维度想拆成多张表时，用 `data/breakdown_<维度>_<子集>.csv`，
在 `site.yaml` 的 `breakdowns` 里分别列出并给不同标题。

---

## 3 · `data/items.csv` —— 逐题明细（可选）

有逐题数据时强烈建议出这张，它让榜单从「排行榜」变成「可分析的 benchmark」。

```csv
question_id,task,category,gold_answer,correct_count,difficulty,DeepSeek-V4-Pro-0813,GLM-5.3,...
me4_000001,mingli_example,事业,D,0,hard,0,0,...
```

| 列 | 说明 |
|----|------|
| `question_id` | 唯一 ID |
| `correct_count` | 答对的模型数，用来算 difficulty |
| `difficulty` | `easy` / `medium` / `hard`，由 `correct_count` 分档 |
| 其余列 | 每个模型一行，值为 0/1 —— **与主榜的模型名保持一致** |

> **隐私红线**：逐题明细若包含原始题面或真实案例，
> 发布前必须核对数据集本身的公开范围。案例题涉及真实人物的，一律匿名化。

---

## 4 · 一致性：三张表对不上怎么办

生成器对数据可信度只提示、不挡：`correct/n` 与主指标不一致会打印警告，页面照常产出。
只有缺列、`model` 重复这类结构性问题才会中断构建。

**细分表的行集与主榜不一致时**（少一个模型、多一个模型）：
生成器会警告并在页面上标注，但**不会**自动补行——
缺的模型说明它没跑过这个维度，补一行 0 是编造数据。

**主榜有 n=3000，细分表某列加起来只有 2400**：
这是真问题。退回 Phase 2 查是哪个维度丢了 600 题，
不要在页面上两个数并排显示。

---

## site.yaml 与页面分区规范

Phase 3 / Phase 4 用。

---

## 1 · 站点源目录结构

`build_site.py` 吃的就是这个目录。**它是真相，`_site/` 是产物。**

```
<站点源目录>/
├── site.yaml                 唯一需要手写的文件
├── data/
│   ├── metrics.md            评测口径（Phase 1 产出，会渲染进榜单页脚注）
│   ├── leaderboard.csv       主榜
│   ├── breakdown_*.csv       细分表（0..N 张）
│   └── items.csv             逐题明细（可选）
├── figures/                  gallery 用图（png/jpg/svg/webp）
├── pdfs/                     paper.pdf / poster.pdf
└── bibtex.bib                可选，给了就覆盖 site.yaml 里的 bibtex
```

`data/` 之外的文件按需添加。**图一律放 `figures/`，不要散在根目录。**

---

## 2 · site.yaml 全字段

```yaml
## ══════════ 必填 ══════════
title: 八字四选一 Benchmark
tagline: 3000 道中国传统命理四选一题，测量大模型的领域知识与推理
abstract: |
  多行摘要原文。不要改写论文摘要，逐字复制。
  首行缩进不保留，段落之间空一行。

authors:
  - name: 十八木
    url: "https://github.com/MonsterPPPP"     # 可选
    affiliation: 独立研究者                     # 可选
    equal_contribution: true                  # 可选，渲染成 *
venue: arXiv preprint 2026
date: 2026-09-22                              # 发布日，用于 OG/article:published_time

links:
  paper: pdfs/paper.pdf                       # 相对源目录；渲染成 PDF 按钮
  code: "https://github.com/..."              # 外部链接直接用
  dataset: "https://huggingface.co/datasets/..."
  arxiv: "https://arxiv.org/abs/..."
  poster: pdfs/poster.pdf                     # 可选

## ══════════ 站点 ══════════
base_url: "https://monsterpppp.github.io/18trees-benchmark-pages-skill"   # 末尾不要斜杠
lang: zh                                      # zh | en；影响 <html lang> 与界面文案
brand_color: "#2F6B4F"                        # 主题色，用 18trees 绿
keywords: [benchmark, 命理, LLM 评测, 中文]     # SEO，5–10 个

## ══════════ 榜单 ══════════
leaderboard:
  source: data/leaderboard.csv
  primary_metric: overall_acc
  primary_label: 总准确率
  higher_is_better: true                      # false 时升序排，最优在最后
  unit: percent                               # percent | ratio
  baseline: 25.0                              # 随机基线数值
  baseline_label: 随机基线（四选一）
  allow_unequal_n: false                      # true 时页面强制显示 n 列
  default_sort: overall_acc                   # 默认排序列，默认 = primary_metric
  highlight: [DeepSeek-V4-Pro-0813]           # 可选，给这些行加视觉标记
  breakdowns:
    - title: 理论题 vs 命例题
      source: data/breakdown_task.csv
      note: 两类任务等权，最终排名由命例题决定
    - title: 分类细项
      source: data/breakdown_category.csv
  items:
    source: data/items.csv                    # 可选
    title: 逐题明细

## ══════════ 内容分区 ══════════
gallery:
  source: figures/                            # 目录下所有图（按文件名排序）
  exclude: [09_deepseek_ablation.png]         # 可选
  captions:                                   # 可选，按文件名键控
    01_leaderboard.png: 六个模型的主结果对比，误差棒为 95% Wilson 置信区间
    02_theory_vs_case.png: 理论题与命例题之间的系统性断层

findings:                                     # 主页的关键发现卡片（3–5 张为宜）
  - title: 理论与命例之间存在系统性断层
    body: 理论题 84–86%，命例题 66–73%，最终排名完全由命例题决定。
  - title: 推理不是免费的
    body: 开启推理链在两代 DeepSeek 上带来不同方向的位移，见消融实验。

limitations: |                                # ★ 不许省
  本 benchmark 的题目覆盖范围、评分方式与已知干扰项见论文 §Limitations。
  六模型评测经由统一网关，网关超时会产生空答案，已按判错计入分母。

bibtex: |
  @article{yourkey2026,
    title={你的论文标题},
    author={作者},
    year={2026}
  }

submission:                                   # ★ 榜单页的「我怎么加进去」
  enabled: true
  repo: "https://github.com/.../benchmark"
  instructions: |
    把结果 CSV 放到 results/<模型名>.csv 后提 PR。
    需要包含：模型名、跑测日期、prompt 模板哈希、每题原始输出。
  template_row: "MyModel, 3000, 2100, 70.00, 68.1, 71.6"

## ══════════ 分区顺序（可选）══════════
sections: [abstract, leaderboard, gallery, findings, limitations, bibtex]
```

---

## 3 · 分区顺序

`index.html` 的默认顺序（`sections` 不写时）：

| # | 分区 | 内容 | 来源 |
|---|------|------|------|
| 1 | `hero` | 标题 · 作者 · 单位 · 链接按钮组 | `title` / `authors` / `venue` / `links` |
| 2 | `teaser` | 主图或 teaser 视频 + caption | `gallery.source` 里文件名含 `teaser` 的图 |
| 3 | `abstract` | 摘要 | `abstract` |
| 4 | `leaderboard` | **主榜（本期差异点）** | `leaderboard.source` |
| 5 | `breakdowns` | 细分表 + 色阶 | `leaderboard.breakdowns[]` |
| 6 | `gallery` | 结果图 | `gallery.source` |
| 7 | `findings` | 关键发现卡片 | `findings[]` |
| 8 | `limitations` | 局限 | `limitations` |
| 9 | `bibtex` | 引用 + 复制按钮 | `bibtex` |
| 10 | `footer` | 上游回链 + 许可 | 固定，不可删 |

`leaderboard.html` 的顺序：主榜 → 细分表 → 逐题明细（若有）→ 口径脚注 → 提交指引 → footer。

**`hero` 和 `footer` 不可通过 `sections` 删除。**

---

## 4 · 可选分区与唯一的必选项

下面三块，前两块**给了配置才渲染，没给就整段不出现**；第三块是本 skill 唯一的强制项。

### 4.1 随机基线锚点（可选）

给 `leaderboard.baseline` 才会出现。主榜上方显示：

```
随机基线 25.0%  ▏最佳 79.5%  ▏领先基线 +54.5pt
```

`baseline_label` 可自定义（如「无基线」「人类水平」）。
**不给 `baseline` 就不渲染这根锚点条**——四选一之外的题型没有天然基线，
硬填一个反而误导。

### 4.2 口径脚注（可选）

有 `data/metrics.md` 才渲染，从文件直出、**不加工**：

> **口径**：分母为每模型全部 3000 条记录；invalid（空答案）判错并计入分母；
> 置信区间为 Wilson score interval (z=1.96)；细分维度先按 category 等权、再按任务族等权。

同样地，`submission` 段给了 `submission.enabled: true` 才出现。

### 4.3 上游回链（唯一必选）

footer 固定文案（`lang: en` 时用英文版）：

> 本页基于 [Academic Project Page Template](https://github.com/eliahuhorwitz/Academic-project-page-template)
> 构建，该模板派生自 [Nerfies](https://nerfies.github.io/) 项目页。
> 本作品以 [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/) 授权。

**生成器不提供关闭这个 footer 的开关。** 它是上游的授权条件，删掉就违约。

---

## 5 · 窄屏与无障碍

生成器已处理，改模板时不要破坏：

- 主榜在窄屏横向滚动，`model` 列冻结
- 图表在窄屏改为纵向堆叠
- 所有图片带 `alt`（未在 `captions` 里给的用文件名兜底并打警告）
- 表头 `<th scope="col">`，排序状态用 `aria-sort`
- 正文对比度 ≥ 4.5:1；`brand_color` 用作文字时自动加深

---

## 部署与结果提交

Phase 5 用。

---

## 1 · 本地验证（部署前必做）

```bash
python -m http.server 8000 --directory _site
```

浏览器逐项点一遍。**生成器跑通不等于页面能用。**

- [ ] 主榜点表头能排序，升降序都对，数字列按数值排不是按字符串排
- [ ] 筛选框能用，清空后恢复全部
- [ ] 基线锚点的数字和表里第一名的数字一致
- [ ] 细分表色阶按列独立，不是全表一个色阶
- [ ] 每张图能打开，caption 和报告里的一致
- [ ] `paper.pdf` 能打开
- [ ] BibTeX 复制按钮能复制（`file://` 下剪贴板 API 会失败，必须用 http server 测）
- [ ] 窗口缩到 375px 宽：主榜横向滚动且 `model` 列不跑，图不溢出
- [ ] 页面源码里搜 `TODO` / `PLACEHOLDER` / `lorem`，**零命中**
- [ ] footer 的上游回链存在且可点

---

## 2 · 部署到 GitHub Pages

三种方式，按仓库形态选：

### 2.1 站点就在 benchmark 主仓库里（推荐）

```
<benchmark-repo>/
├── paper/  data/  eval/      评测与论文
└── docs/                      ← 站点放这里
    ├── site.yaml
    ├── data/
    └── index.html  ...
```

设置 → Pages → Source 选 `Deploy from a branch` → 分支 `main` / 目录 `/docs`。

优点：数据和站点同仓库，改数据提 PR 时顺手就能重跑生成器。

### 2.2 独立站点仓库

`<org>.github.io` 或 `xxx.github.io` 仓库，根目录即站点。
适合站点需要独立历史、或 benchmark 主仓库不想被站点产物污染。

### 2.3 CI 自动构建

主仓库里放 `.github/workflows/pages.yml`，每次 `data/` 或 `site.yaml` 变更时自动重跑生成器：

```yaml
name: build-site
on:
  push:
    branches: [main]
    paths: ['docs/site.yaml', 'docs/data/**', 'docs/figures/**']
permissions:
  contents: read
  pages: write
  id-token: write
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: {python-version: '3.12'}
      - run: pip install -r docs/requirements.txt
      - run: python docs/scripts/build_site.py --source docs --out _site
      - uses: actions/upload-pages-artifact@v3
        with: {path: _site}
  deploy:
    needs: build
    runs-on: ubuntu-latest
    environment: github-pages
    steps:
      - uses: actions/deploy-pages@v4
```

### 2.4 必做的一次性配置

- 仓库根目录放 **`.nojekyll`** 空文件。
  没有它，GitHub Pages 的 Jekyll 会**吞掉**以下划线开头的文件名和目录。
- Pages 设置里勾上 **Enforce HTTPS**。
- `site.yaml` 的 `base_url` 必须和最终 URL 一致，否则 OG 卡片和 citation 元数据全指向错误地址。

---

## 3 · 第三方结果提交

榜单页的 `submission` 段要能让人**不问你**就把结果交上来。缺任何一项都会变成邮件来回。

### 3.1 提交物清单（写进页面）

| 必交 | 说明 |
|------|------|
| 结果 CSV | 一行一题或聚合行，列名照 `leaderboard-spec.md` |
| 原始输出 | 每题的模型原始回答（用来复核解析逻辑，防止 prompt 差异导致的假高分） |
| 模型标识 | 确切版本号，不是 `gpt-4` 这种泛称 |
| 跑测日期 | `YYYY-MM-DD` |
| prompt 模板 | 或模板的 sha256；不用官方模板的要显式说明并标在榜上 |

### 3.2 提交方式

**方式 A：PR 到榜单数据文件**（轻，适合小 benchmark）

`submission.repo` 指向存放 `data/` 的仓库。提 PR 改 `leaderboard.csv`，
CI 跑结构校验（`build_site.py --check-only`）挡掉缺列之类的错误，过了就合。
要连数据可信度一起卡，CI 里加 `--strict`。

**方式 B：独立 results 仓库**（重，swe-bench 的做法）

第三方把结果推到**他们自己的**公开仓库，在你这边只提一个引用 PR。
好处：主仓库不被大文件污染，且结果是可审计的。

选 A 还是 B 取决于是否要求提交者提供可复现的原始日志。要求 → B。

### 3.3 页面上的写法

```markdown
## 提交你的结果

1. 按 [results/TEMPLATE.csv](...) 的格式整理你的结果
2. 确保包含每题原始输出（不是只有聚合分数）
3. 提 PR 到 <repo>，标题格式 `[Result] <模型名> <日期>`
4. 维护者会在 7 天内核对；通过后榜单自动更新

**我们不接受**：无法追溯到具体版本的模型、使用了未公开 prompt 模板而未标注的结果。
```

### 3.4 审核标准要写出来

不写标准的榜单会被「调过 prompt 的刷分结果」占领。至少写清：
是否要求用官方 prompt 模板、是否接受 few-shot、是否要求公开日志、谁有权合并。

---

## 4 · 发布前检查

- [ ] `.nojekyll` 存在
- [ ] `base_url` 正确，OG 卡片用实际 URL 验证过
- [ ] 站点源目录（`site.yaml` + `data/`）已进版本控制，**不是只在本地**
- [ ] 数据集本身的公开范围已核对（案例题是否匿名化、有无真实个人信息）
- [ ] 结构校验通过：`python scripts/build_site.py --source <源目录> --check-only`
- [ ] 数据提示已逐条看过（`correct/n` 不一致、分母不整齐 —— 确认是有意的还是漏更新了）
- [ ] footer 上游回链存在（生成器会硬校验，改模板时别把它弄丢）

以下按需，不是必须：

- [ ] 榜单页写了口径、基线、提交方式（对应分区开了才有，见 `site-spec.md §4`）

---

*本文件由 `scripts/build-dist.sh` 从 `skills/benchmark-pages/` 自动生成，请勿直接编辑。*
*仓库：https://github.com/MonsterPPPP/18trees-benchmark-pages-skill · License: CC BY-SA 4.0*
*模板层派生自 Academic Project Page Template 与 Nerfies（CC BY-SA 4.0），详见仓库 NOTICE.md*
