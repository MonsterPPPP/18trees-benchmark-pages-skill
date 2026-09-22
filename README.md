<p align="center">
  <img src="./assets/banner.webp" alt="十八木" width="100%" />
</p>

<p align="center">
  中文 · <a href="./README.en.md">English</a>
</p>

# 18trees-benchmark-pages-skill

**把 benchmark 的 rawdata + 论文 + 分析报告，变成一个能托管在 GitHub Pages 上的主页与排行榜。**

做 benchmark 的人都在干同一件事：fork 那个 5.2k★ 的学术主页模板，然后手搓自家 benchmark 特有的部分。

问题是那个模板**一样都没有**——没有数据驱动的表格、没有排序筛选、没有榜单入口、没有提交指引。
它是个论文主页模板，不是 benchmark 模板。于是每个人都要重新解决一遍同样的问题：
表格怎么从实验结果生成、模型更新了榜单怎么改、别人怎么把自己的结果加进来。

**这个 skill 把那些「每个人都重做一遍」的部分做完了。**

> **▶ [看生成出来的页面长什么样](https://monsterpppp.github.io/18trees-benchmark-pages-skill/)**
>
> 用一次真实评测数据（6 个模型 × 2492 道题）现场生成的演示站，
> 由 [`.github/workflows/pages.yml`](./.github/workflows/pages.yml) 自动构建。
> 源目录见 [`site/`](./site/)。

---

## 它解决什么问题

| 你现在的处境 | 结果 |
|-------------|------|
| 实验结果散在 `experiments/*.jsonl`、`analysis/*.csv`、`REPORT.md` 里 | 要发主页时，手工把数字一个个抄进 HTML |
| 想加一个新模型 | 改 HTML 里的 `<td>`，改完发现图表里的数字没同步 |
| 榜单上只有自己跑的 6 个模型 | 看起来像广告，不像 benchmark |
| 读者看到「75.6%」 | 不知道这算好还是差——没有随机基线 |
| 审稿人问「invalid 回答怎么算的」 | 得翻论文才能回答 |

---

## 我们的贡献

**1. 数字全部从 CSV 重算，不许手写进 HTML。**

页面里每一个数字都来自 `data/*.csv`，由 `build_site.py` 渲染。
生成器会拿 `correct / n` 反算主指标做一致性校验，**对不上直接报错退出，不产出半成品**。
想改页面上某个数字？改 CSV，重跑。

**2. 榜单该有的分区，都是现成的。**

不用自己写表格。`site.yaml` 里按需开启，没配的整段不出现：

- **排行榜** —— 列排序、按模型名/机构筛选、置信区间、第一名视觉标记
- **分维度热力图** —— **色阶按列独立归一化**（跨列难度不同，全表一个色阶会把列间差异抹平）
- **随机基线锚点** —— `随机基线 25.0% ▏最佳 75.6% ▏领先基线 +50.6pt`，给了 `baseline` 才出现
- **评测口径脚注** —— 分母、invalid 处理、置信区间、聚合权重，从 `data/metrics.md` 直出
- **结果提交指引** —— 别人怎么把模型加进你的榜

**3. 一个生成器，不是一堆要手改的 HTML。**

```bash
python scripts/build_site.py --source docs --out _site
```

`site.yaml` 是整站唯一需要手写的文件。数据更新 → 重跑 → 页面永远和数字一致。

---

## 安装

### Claude Code

```bash
/plugin marketplace add MonsterPPPP/18trees-benchmark-pages-skill
/plugin install benchmark-pages@18trees-benchmark-pages-skill
```

### 手动安装

把 `skills/benchmark-pages/` 复制到 `.claude/skills/`（项目级）或 `~/.claude/skills/`（用户级）。

Codex 用 `.codex/skills/`。安装细节见 [AGENTS.md](./AGENTS.md)。

### 依赖

```bash
pip install -r scripts/requirements.txt
```

只有一个依赖：PyYAML。表格引擎 Tabulator 已随仓库分发，无需安装，**生成的站点完全自包含、离线可打开**。

---

## 使用

### 1. 备齐三样输入

| 输入 | 典型形态 |
|------|---------|
| **rawdata** | `experiments/<run>/*.jsonl`、`analysis/canonical/*.csv` |
| **论文** | LaTeX 源 / PDF；或已定稿的标题·作者·摘要 |
| **分析报告** | `analysis/REPORT.md`、`analysis/figures/*.png` |

### 2. 让 skill 走六个阶段

```
Phase 0  盘点输入          ← 哪些是数据，哪些是叙事
Phase 1  锁定评测口径      ← 分母 / invalid / CI / 聚合权重 / 随机基线
Phase 2  造三张表          ← leaderboard.csv + breakdown_*.csv + items.csv
Phase 3  写 site.yaml      ← 站点配置（唯一需要人写的东西）
Phase 4  渲染 + 自检       ← build_site.py（数字对不上就退出）
Phase 5  本地验证 + 部署   ← http.server 逐页看 → GitHub Pages
```

### 3. 站点源目录长这样

```
docs/
├── site.yaml                 唯一需要手写的文件
├── data/
│   ├── metrics.md            评测口径
│   ├── leaderboard.csv       主榜：一行一个参赛者
│   └── breakdown_*.csv       分维度表
├── figures/
└── pdfs/
```

跑完得到 `_site/`：`index.html` + `leaderboard.html` + `static/`，直接推到 GitHub Pages。

完整可运行的例子见 [`site/`](./site/)。

---

## 仓库结构

```
├── skills/benchmark-pages/
│   ├── SKILL.md                    规则唯一真相
│   ├── references/                 输入契约 / 榜单规范 / 站点规范 / 部署
│   ├── assets/template/            版面模板（派生自上游，CC BY-SA 4.0）
│   └── assets/vendor/tabulator/    表格引擎（MIT，随站点分发）
├── scripts/
│   ├── build_site.py               生成器
│   └── build-dist.sh               单文件版构建
├── dist/benchmark-pages.md         单文件全约束版（生成物）
├── site/                  演示站点源目录（真实数据）
├── .github/workflows/pages.yml     演示站的自动构建与发布
└── NOTICE.md                       上游致谢与许可分层
```

---

## 开发

```bash
# 改完规则重新生成单文件版
bash scripts/build-dist.sh

# 冒烟测试
python scripts/build_site.py --source site --out _smoke
python -m http.server 8000 --directory _smoke
```

**生成器跑通不等于页面能用。** 至少要在浏览器里验证：表格渲染出来、点表头能排序、
窄屏不横向溢出、模型列在最左且可见。改模板或生成器后必须做这一步。

贡献方向见 [CONTRIBUTING.md](./CONTRIBUTING.md)。

---

## 和同类项目比

> 数据快照：2026-09-22，星数与最近提交取自 GitHub API。

| 项目 | ★ | 最近提交 | License | 它是什么 | 差在哪 |
|------|---|---------|---------|---------|--------|
| [Academic-project-page-template](https://github.com/eliahuhorwitz/Academic-project-page-template) | 5,229 | 2025-09-04 | ⚠️ 无 LICENSE 文件 | 学术论文主页模板，全行业都在 fork | 没有表格、没有榜单、没有提交指引；空手搓 benchmark 部分 |
| [swe-bench/swe-bench.github.io](https://github.com/swe-bench/swe-bench.github.io) | 15 | 2026-09-01 | CC BY-NC 4.0 | 主页 + 5 个榜单同站，Python/Jinja2 生成 | 与 swe-bench 深度耦合，不能拿来用；非商用协议 |
| [evalplus/evalplus.github.io](https://github.com/evalplus/evalplus.github.io) | 13 | 2024-12-26 | Apache-2.0 | `results.json` + 手写 JS 排序筛选 | 只解决「渲染自家榜单」，没有论文页、没有提交流程、没有口径披露 |
| [vividvilla/csvtotable](https://github.com/vividvilla/csvtotable) | 1,183 | 2026-08-24 | MIT | CSV → 自包含可排序 HTML | 是表格工具不是 benchmark 工具：没有基线、没有口径、没有分维度、没有部署 |
| [Tabulator](https://github.com/olifolkerd/tabulator) | 7,771 | 2026-09-15 | MIT | 通用表格库 | 是零件不是产品——本 skill 用它做渲染，但把「benchmark 该怎么呈现」的规范补齐 |

### 怎么选

| 你的情况 | 用什么 |
|---------|--------|
| 给一篇普通论文做项目页，没有榜单 | 直接用 Academic Project Page Template |
| 做一次性页面，数据不会再变 | evalplus 那套 `results.json` + 手写 JS 就够 |
| 要把 CSV 变成能发给别人看的可排序表格 | csvtotable |
| **榜单数据会持续更新，每次改都要重跑一遍** | **本 skill** |

**最后一行是它唯一的目标场景。** 本 skill 的价值不在「页面好看」，
在于**数据变了重跑一次，页面跟着变，且数字永远和 CSV 一致**。
榜单分区（排序筛选、热力图、基线锚点、口径脚注、提交指引）是顺带解决的——
它们本来就得有，只是以前每个人都要自己写一遍。

---

## License

本仓库以 **CC BY-SA 4.0** 授权，因为版面模板派生自
[Academic Project Page Template](https://github.com/eliahuhorwitz/Academic-project-page-template)
与 [Nerfies](https://nerfies.github.io/)（两者同为 CC BY-SA 4.0）。

- **模板层**（`assets/template/`）与其衍生出的站点页面：CC BY-SA 4.0，
  **footer 必须保留上游回链**
- **生成器与规则文本**（`scripts/`、`SKILL.md`、`references/`）：十八木原创，额外许可 **MIT**
- **Tabulator**：MIT，© Oliver Folkerd

完整的上游致谢、第三方声明与许可分层见 [NOTICE.md](./NOTICE.md)。
