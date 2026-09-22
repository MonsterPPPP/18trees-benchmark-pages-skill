# 贡献指南

## 最有价值的贡献

**1. 新的输入形态适配。**

`references/input-contract.md` 现在覆盖了「逐题宽表」「逐题长表」「只有聚合结果」三种情况。
真实的 benchmark 还有更多形态：多轮对话评测、agent trajectory、人工评分量表、
带子任务的层级评测。每种都该有明确的判定标准和处理方式。新增一类需要给出**真实文件样例**
（可以脱敏），说明它是哪一类、怎么 pivot、哪些字段是必填的。

**2. 榜单的交互缺陷。**

在真实 benchmark 上用的时候发现某个功能不好使——候选太多时找不到自己的模型、
热力图配色区分度不够、窄屏下表头跑位——提出来。
`site.yaml` 的开关设计（哪些分区该可选、默认开还是关）也属于这一类。

**3. 修正已有的判断。**

`references/deploy-pages.md` 的 GitHub Pages 配置、`assets/template/` 的版面细节，
如果在新版 GitHub Pages / 新浏览器上不成立了，提 PR 修正并附上实测环境。

## 不要提交

- **编造的数字或示例。** 示例数据必须来自真实产物，可以脱敏，不可以现编。
  一份假的样例会让后来者照着错的东西建模。
- **移除上游回链的改动。** footer 里的 Academic Project Page Template / Nerfies 回链
  是 CC BY-SA 4.0 的署名条件，见 `NOTICE.md`。
- **绕过 `build_site.py` 的数据校验。** `correct/n` 与主指标的一致性检查是硬失败，
  不要加开关让它变成警告。
- **把 skill 扩展成评测框架或数据分析工具。** 本 skill 只消费已完成的分析产物，
  「不做评测、不做分析、不做设计」是边界，不是可选项。
- **包含真实个人信息的示例或测试数据。** 案例题、人物故事类内容必须匿名化。

---

## 累计式优化

和本组织的其他 skill 一样：**只增量优化，不回退。**

- 新版本只能在上一版本基础上增加、修正或细化。
- 除非维护者明确要求废弃某条规则，否则旧规则持续有效。
- 用 patch / append / refine，不要 rewrite from scratch。

---

## 规范的一致性

改规则时注意三处必须同步：

| 改动 | 还要改哪里 |
|------|-----------|
| `site.yaml` 新增字段 | `references/site-spec.md` 的字段表 |
| `leaderboard.csv` 新增列 | `references/leaderboard-spec.md` 的列规范 |
| `build_site.py` 的渲染逻辑 | 对应的 `assets/template/sections/*.html` |

改完之后跑冒烟测试：

```bash
python scripts/build_site.py --source site --out _smoke
```

**并在浏览器里实际打开 `_smoke/`。** 生成器零报错、校验全过，页面仍然是可能坏的——
历史上有过「hero 分区从未被渲染」和「所有本地资源走了绝对 CDN 地址」两个 bug
都是浏览器验证才发现的。

## 规则只维护在一个地方

`dist/benchmark-pages.md` 是**生成物，不要直接编辑**。

```bash
bash scripts/build-dist.sh
```

改完规则跑一次，把 `dist/` 和 `skills/` 一起提交。
