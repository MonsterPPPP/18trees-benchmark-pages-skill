# AGENTS.md

给 AI 编程助手看的安装与维护说明。

## 这个仓库是什么

一个跨工具 AI skill：把一份 benchmark 的 rawdata + 论文 + 分析报告，
渲染成能托管在 GitHub Pages 上的主页与排行榜。

- **规则源文件**：`skills/benchmark-pages/`
- **版面模板**：`skills/benchmark-pages/assets/template/`（派生自上游，CC BY-SA 4.0）
- **第三方引擎**：`skills/benchmark-pages/assets/vendor/tabulator/`（MIT）
- **生成器**：`scripts/build_site.py`
- **生成物**：`dist/benchmark-pages.md`（单文件版，由脚本生成，不要直接编辑）

## 安装这个 skill

### Claude Code

项目级：把 `skills/benchmark-pages/` 复制到项目的 `.claude/skills/`。
用户级：复制到 `~/.claude/skills/`。

或走插件市场：

```bash
/plugin marketplace add MonsterPPPP/18trees-benchmark-pages-skill
/plugin install benchmark-pages@18trees-benchmark-pages-skill
```

### OpenAI Codex

项目级：复制到 `.codex/skills/`。用户级：复制到 `~/.codex/skills/`。

目录结构应为：

```
~/.codex/skills/benchmark-pages/
├── SKILL.md
├── references/
│   ├── input-contract.md
│   ├── leaderboard-spec.md
│   ├── site-spec.md
│   └── deploy-pages.md
├── assets/
│   ├── template/
│   └── vendor/tabulator/
└── agents/openai.yaml
```

### 能执行命令、但不支持 skill 机制的 AI 工具

如 Cursor、Windsurf、自建 agent。把 `dist/benchmark-pages.md` 的全部内容作为指令或系统提示交给它。

注意单文件版的边界：它**不含模板与生成器**，只有规则与规范。
真正生成站点仍需要有仓库里的实体文件。

### ⚠️ 不适用：网页版聊天机器人

**ChatGPT / Gemini / DeepSeek / Kimi / 豆包 的网页版跑不了本 skill。**

它要做的是：读你的实验数据文件 → 写出 `site.yaml` 与几张 CSV → 执行 `build_site.py`
渲染静态站。没有文件系统与命令执行能力的模型只能**凭印象编一个 HTML**，
而本 skill 的第一原则是 **页面里每个数字都能从 CSV 重算**。编出来的页面恰好违背它存在的理由。

判断标准：skill 需不需要读写文件或执行命令。需要 → 必须有工具能力。

## ⚠️ 许可义务（改动前必读）

模板层派生自 **Academic Project Page Template** 与 **Nerfies**，两者均为 **CC BY-SA 4.0**。

- **不要移除 `sections/footer.html` 里的上游回链。** 那是授权条件，不是致谢装饰。
- **不要给 `build_site.py` 添加关闭回链校验的开关。** `REQUIRED_BACKLINKS` 是硬校验。
- 本仓库整体以 CC BY-SA 4.0 授权；贡献进来的代码默认按同一协议授权。
- 详细的许可分层与第三方声明见 `NOTICE.md`。

## ⚠️ 数据边界

- **不做评测，不做数据分析。** 本 skill 只消费已完成的分析产物。
  跑模型、算显著性、做消融都该在别处完成。
- **不编造数字。** 缺数据就报错让用户补齐，不许插值、不许估算、不许填 0 冒充「没测」。
- **涉及真实个体的案例内容必须匿名化**后才可进入站点。

## 修改规则时

1. 只改 `skills/benchmark-pages/` 下的文件
2. 运行 `bash scripts/build-dist.sh` 重新生成 `dist/`
3. 两个目录一起提交

改完后必须跑一次冒烟测试：

```bash
python scripts/build_site.py --source site --out _smoke
```

四项数据校验 + 产物校验都必须通过。

## 验证改动

- `dist/` 与 `skills/` 一致性：重跑 `build-dist.sh` 后 `git diff` 应无输出
- frontmatter：`name` ≤ 64 字符，`description` 非空
- 改 `build_site.py` 或模板后，**必须在浏览器里实际打开生成出的站点**——
  生成器跑通不等于页面能用。至少验证：表格渲染出来、点表头能排序、
  窄屏不横向溢出、模型列在最左且可见、图上文字没有和背景撞色
- 所有新增的 `site.yaml` 字段都要同步更新 `references/site-spec.md`
