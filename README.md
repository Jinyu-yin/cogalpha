# CogAlpha

> 一个**可审计的 Alpha 因子自动演化框架**：把因子挖掘拆成"领域智能体 → 候选生成 → 适应度筛选 → 演化算子 → 治理归档"的闭环。

![python](https://img.shields.io/badge/python-3.10%2B-3776ab) ![design](https://img.shields.io/badge/design-skill--graph-6366f1)

---

## 设计出发点

自动挖因子的常见做法是"让 LLM 不断写表达式，然后按 IC 排序"——问题是：**无法解释某个因子为什么被留下，也无法复现那一轮的判断**。

CogAlpha 的思路是把整个流程做成**技能图（skill graph）**：

- 每个"能力"都是一个独立的 Skill（带 `SKILL.md` 声明），而不是散落的 prompt；
- 每轮演化留下 trace、manifest 与治理记录，任何一个入选因子都能回溯到"哪一代、哪个父代、哪个适应度指标"；
- 生成 → 评估 → 反馈 → 再生成，形成**受控的进化循环**，而不是一次性的批量生成。

---

## 七级技能层级

`registry.py` 用 `DomainAgentSpec` 静态声明论文定义的领域智能体，按层级与研究方向分组：

| 层级 | 方向 | 智能体（示例） |
|---|---|---|
| L1 | Market Structure & Cycle | `alpha-market-cycle`、`alpha-volatility-regime` |
| L2 | Extreme Risk & Fragility | `alpha-tail-risk`、`alpha-crash-predictor` |
| L3 | Price-Volume Dynamics | `alpha-liquidity`、`alpha-order-imbalance`、`alpha-price-volume-coherence` |
| … | … | 更多层级见 `registry.py` |

此外还有两类技能：

- **演化算子**：`alpha-mutation`（变异）、`alpha-crossover`（交叉）；
- **质量技能**：用于候选的审查、去重与治理判断。

---

## 目录结构

```
cogalpha/
└── cogalpha/
    ├── registry.py              # ⭐ 技能注册表（七级层级定义）
    ├── skills.py                # 技能清单门面
    ├── skill_loader.py          # 技能发现 + 渐进式上下文装配
    ├── skill_invocation.py      # 技能调用编排
    ├── skill_nodes.py           # DAG 节点级技能调用
    ├── skill_library.py         # 基于 trace 的技能效用与受治理更新
    ├── schemas.py               # 运行期交换的数据结构
    ├── alpha_contract.py        # Alpha 函数与 OHLCV 输入契约
    ├── execution.py             # 受限执行（生成函数跑在 OHLCV 面板上）
    ├── candidate_lifecycle.py   # 候选生命周期与池状态流转
    ├── evolution_plan.py        # ⭐ 演化规划（mutation / crossover）
    ├── fitness.py               # 适应度计算与论文定义的筛选门槛
    ├── evaluation.py            # 基于面板的候选评估
    ├── feedback.py              # 从质量与适应度结果构建自适应反馈
    ├── governance.py            # 证据、实验、晋级决策的治理产物
    ├── data.py                  # 行情加载 + 防泄漏的前向收益构造
    ├── preflight.py             # 固定样本预检
    ├── manifest.py              # 运行清单（可复现性）
    ├── instrumentation.py       # 形式化运行的工具埋点
    ├── tracing.py               # trace 事件 schema + JSONL 落盘
    ├── reporting.py             # 分层报告
    └── config.py                # 配置对象
```

---

## 核心循环

```
        ┌──────────────────────────────────────────────┐
        │                                              │
        ▼                                              │
  领域智能体 (L1–L7)                                    │
        │  产出 Alpha 候选（AlphaCandidate）            │
        ▼                                              │
  受限执行 execution.py  ──▶  OHLCV 面板上计算          │
        │                                              │
        ▼                                              │
  适应度筛选 fitness.py / evaluation.py                 │
        │  IC / IR / 换手 / 衰减 …                      │
        ▼                                              │
  反馈 feedback.py  ──▶  有效 vs 无效的归因摘要          │
        │                                              │
        ▼                                              │
  演化规划 evolution_plan.py                            │
        │  mutation（单亲） / crossover（双亲）          │
        └──────────────────────────────────────────────┘
                       │
                       ▼
              governance.py  ──▶  证据归档 + 晋级决策
```

每一轮都会写：

- **manifest**：本次运行的配置、数据切片、随机种子；
- **trace（JSONL）**：每个节点的输入输出，可离线重放；
- **governance 产物**：候选为什么晋级 / 被拒。

---

## 关键概念

### Alpha 契约（`alpha_contract.py`）

所有生成的因子函数必须满足统一签名——输入是 OHLCV 面板，输出是截面因子值。这是"让 LLM 生成的东西能被安全执行"的前提。

### 受限执行（`execution.py`）

生成代码不在主进程里任意跑，而是在受限环境中对面板求值。**先约束，再执行**，是这类系统必须守住的红线。

### 防泄漏的前向收益（`data.py`）

`data.py` 明确负责"split-safe forward-return construction"——训练/验证切分之后才计算前向收益，避免用未来数据构造标签。

### 技能库自更新（`skill_library.py`）

技能不是写死的。基于 trace 的效用统计，可以生成**受治理的**技能更新记录——哪些提示策略有效、哪些长期无效。

---

## 使用前提

> **重要**：本仓库仅包含 Python 实现层。技能的实际内容位于同级 `skills/` 目录（每个技能一个 `SKILL.md`），**未包含在本仓库中**。

`registry.py` 通过以下方式定位技能文件：

```python
PROJECT_ROOT = Path(__file__).resolve().parents[1]   # 仓库根
SKILLS_ROOT  = PROJECT_ROOT / "skills"               # 需要自行提供
```

因此要真正跑起来，需要在仓库根目录准备好：

```
cogalpha/                 # 本仓库
├── cogalpha/             # 包
└── skills/
    ├── alpha-market-cycle/SKILL.md
    ├── alpha-volatility-regime/SKILL.md
    ├── alpha-mutation/SKILL.md
    └── ...
```

---

## 环境

```bash
pip install pandas numpy scipy
```

无需额外框架依赖——技能调用层是自实现的，便于嵌入任意 agent runtime。

---

## 已知限制

- 技能内容（`skills/`）未开源，仓库只提供**编排与治理骨架**；
- 适应度门槛与指标权重来自论文设定，未在多个市场做过普适性验证；
- 受限执行目前只做接口与沙箱层面的约束，生产使用需追加资源限额与超时控制；
- 缺少端到端示例数据，`preflight.py` 需要自备行情面板。
