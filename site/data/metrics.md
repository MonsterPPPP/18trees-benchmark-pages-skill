# 评测口径

- **分母**：每个模型的全部 canonical 记录（n = 2492）；k/n*100，分母=该模型全部 canonical 记录，invalid 空答案判错且计入分母
- **invalid（空答案）**：判错，且计入分母。剔除会让分数虚高，且网关超时属基础设施因素，跨模型不可比。
- **置信区间**：Wilson score interval, z=1.96
- **多维度聚合**：(macro_theory_acc + macro_case_acc)/2，Theory/Case 各占 50% 权重（Theory 14 个 category / Case 11 个 category）
- **随机基线**：25.0（四选一）
- **领先基线**：overall_acc - 25.0
