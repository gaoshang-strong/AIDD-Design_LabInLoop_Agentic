# 项目3（12–18个月）：De novo 设计 + Lab-in-the-loop + Agentic AI（闭环系统项目）
> 目标：把你训练成“能驱动湿实验迭代”的 AI scientist：不仅能训练模型，还能 **提出候选、解释理由、设计下一轮实验、自动化执行与审计**。  
> 即便你暂时没有真实 wet-lab，本项目也能用公开数据 + 仿真噪声做闭环演示，重点是“系统能力”。

---

## 0. 6个月末交付清单（像一个内部产品）
**交付物 A：端到端设计闭环（v1）**
- 输入：目标定义（binder/稳定性/特定 motif/口袋环境等）
- 输出：每轮一个候选集（例如 50–200 个序列）+ 每个候选的证据链：
  - 生成来源（版本、参数、prompt/约束）
  - 结构预测/验证结果
  - 性质打分（稳定性/表达/免疫原性 proxy 等）
  - 不确定性与风险项
- 选择策略：多目标 Pareto + 多样性约束 + 预算控制（一次做多少实验）

**交付物 B：Agentic 工作流（可追溯、可复现）**
- 一个 agent（或多 agent）自动完成：
  1) 拉取/更新数据
  2) 训练/刷新预测器
  3) 生成候选
  4) 调用评估工具（结构预测、打分）
  5) 生成实验建议与报告
- 每一步都有日志与审计：输入→输出、版本、失败原因

**交付物 C：闭环演示（无 wet-lab 也能成立）**
- 用公开数据当“实验真实反馈”，或者用“已知打分函数 + 噪声”模拟实验
- 证明：active learning/策略能在有限轮次内提升目标指标

---

## 1. 本项目要训练的核心能力（对齐 JD）
1) de novo design 算法理解与调用（你不一定训练 SOTA，但要会组合与评测）  
2) 多模态打分与筛选（结构+序列+功能/实验）  
3) 实验设计（active learning、预算、批次）  
4) agentic AI（把科学流程自动化与可追溯）  
5) 沟通与交付（让 wet-lab 同事拿到就能做）

---

## 2. 设计闭环的“最小可行版本”（MVP）
> MVP 的关键不是最强模型，而是“闭环能跑，且每步可解释”。

### 2.1 MVP 输入输出规范
**输入（一个 YAML/JSON 任务文件）**
- target_type: binder / stability / enzyme / scaffold
- target_structure: PDB/AF file（可选）
- constraints:
  - fixed_motif residues（可选）
  - interface hotspots（可选）
  - symmetry / length range（可选）
- objectives（多目标权重）：
  - predicted_affinity ↑
  - stability ↑
  - aggregation_risk ↓
  - immunogenicity_proxy ↓
- budget:
  - round_budget: 每轮实验/合成数量
  - total_rounds: 轮数

**输出**
- round_k/
  - candidates.csv（序列、分数、风险、不确定性）
  - structures/（预测结构）
  - reports/（PDF/HTML 报告）
  - provenance.json（审计信息）

### 2.2 MVP 组件拆分
1) Generator（生成候选）
2) Evaluator（结构预测 + 打分）
3) Selector（多目标 + 多样性 + 预算）
4) Learner（基于实验反馈更新预测器）
5) Orchestrator/Agent（串起流程）

---

## 3. 推荐的技术路线（按“先组合→后创新”）
### 3.1 Generator：先从“结构条件序列设计”切入（更稳）
- 第一阶段不必自己训练 RFdiffusion 等重模型；先使用公开工具或简化生成器：
  - 结构条件序列设计（inverse folding）
  - 或者基于 PLM 的序列采样（带约束）
- 关键是：输出多样性 + 满足约束

### 3.2 Evaluator：结构验证是核心
- 对每个候选：做结构预测/回折验证（可用轻量推理路线）
- 结构质量过滤：低置信区域/不合理几何直接淘汰
- 打分模型：用项目1/2 训练的 predictors（稳定性/PPI/ΔΔG proxy）

### 3.3 Selector：多目标+多样性（这是你的“可写贡献点”）
- 多目标 Pareto front
- diversity：序列距离/embedding 距离/结构差异
- 风险约束：例如免疫原性 proxy 上限、聚集风险上限
- 不确定性：对高不确定候选分配探索配额（exploration）

### 3.4 Learner：active learning / Bayesian optimization
- 每轮用新数据更新 predictor
- acquisition：
  - expected improvement（回归）
  - uncertainty sampling（校准后）
  - diversity-aware batch acquisition

### 3.5 Agent：把流程产品化
- 不是聊天；而是把“科学 SOP”写成可执行的 agent
- 输出必须可审计：每个候选为什么被选中

---

## 4. 6个月计划（按月里程碑）
### Month 1：搭建闭环骨架（无模型也能跑）
**目标**
- 用一个 toy generator + toy scorer 把闭环跑起来（round0→round1）
- 建立数据 schema 与 provenance 机制

**任务清单**
1) 设计 input spec（YAML）与 output artifacts 目录
2) 实现 orchestrator：按步骤执行、失败重试、日志
3) toy generator：随机突变/小库生成（可控多样性）
4) toy scorer：用公开规则或简单模型打分
5) 输出报告模板（HTML/PDF）：候选列表 + 解释

**验收**
- 你能演示：给一个目标 → 自动输出一批候选 + 报告

### Month 2：接入真实 predictor（来自项目1/2）
**目标**
- 用项目1/2训练的模型替换 toy scorer
- 引入不确定性与校准

**任务清单**
1) 把 predictor 封装成统一接口：`predict(seqs, structures=None) -> scores, uncertainty`
2) 加入校准：温度缩放/集成/MC dropout（选一个可行）
3) 结构验证：至少做一个“结构质量过滤”步骤（阈值）

**验收**
- 你的候选排序不再是 toy，而是可解释的模型打分

### Month 3：实现多目标选择器（Pareto + diversity）
**目标**
- selection 策略成为系统核心：给定预算选出最优候选批次

**任务清单**
1) 实现 Pareto front 计算与可视化
2) diversity 约束：embedding 距离阈值或最大化覆盖
3) exploration quota：每轮保留一部分高不确定候选

**验收**
- 你能解释：为什么这 100 个候选被选中（不是黑箱）

### Month 4：active learning 闭环验证（模拟实验）
**目标**
- 在“模拟实验”或公开数据上演示：闭环策略能更快提升目标指标

**任务清单**
1) 构造模拟环境：
   - ground-truth function（可用公开数据拟合代理）+ 噪声
2) 比较策略：
   - random
   - greedy top-score
   - uncertainty sampling
   - Pareto+diversity（你的）
3) 画学习曲线：每轮最佳值/均值/成功率

**验收**
- 你的策略在有限轮次内显著优于 random/greedy

### Month 5：Agentic AI 打磨（从脚本到“助手系统”）
**目标**
- agent 能自动写“实验建议”与“下一轮计划”，并能处理异常

**任务清单**
1) 自动生成实验建议：
   - 每轮建议测哪些指标、为什么、预计风险
2) 异常处理：
   - 结构预测失败、打分缺失、输入不合法
3) 全流程审计：
   - provenance.json：版本、输入、输出、hash

**验收**
- 你能在 15 分钟 demo 中展示：端到端跑一次 + 输出解释

### Month 6：论文/专利化包装 + 外部展示材料
**目标**
- 形成可投稿的短文/系统论文稿
- 或形成专利 draft 的“创新点描述”

**任务清单**
1) 写论文结构：
   - 问题（lab-in-loop 的瓶颈）
   - 系统（模块化闭环）
   - 策略（Pareto+uncertainty+diversity）
   - 实验（模拟或公开数据闭环）
2) 做对外演示：
   - 1页架构图
   - 2个案例（成功/失败）
   - 1张学习曲线

**验收**
- 你能把系统清晰讲给非 ML 的实验同事

---

## 5. 风险与备选
- **没有真实 wet-lab**：用公开数据 + 模拟噪声做闭环仍然可发表/可展示  
- **结构预测成本高**：用分级筛选（cheap scorer → top-K 才做结构预测）  
- **目标定义太散**：先聚焦 1 个场景（binder 或 stability）做到极致  

---

## 6. 简历写法（模板）
- Built an agentic, lab-in-the-loop protein engineering platform that generates, scores, selects, and iteratively improves candidate sequences under multi-objective constraints.
- Implemented diversity- and uncertainty-aware batch acquisition, outperforming greedy and random baselines in simulated/benchmark closed-loop experiments.
- Delivered traceable candidate rationales and structure-based visual reports to guide experimental prioritization.

---

## 7. 第一周最小执行清单
1) 写任务输入 YAML spec（目标、约束、预算）  
2) 写 orchestrator（顺序执行 + 产出目录 + 日志）  
3) 用 toy generator/scorer 跑通 Round 0→1  
4) 输出第一份 HTML 报告（候选表 + 简单解释）  
