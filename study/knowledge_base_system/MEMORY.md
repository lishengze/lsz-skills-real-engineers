# MEMORY.md — 知识库索引（lsz-skills-real-engineers）

> 本文件是知识库的"搜索引擎"，保持精简。详细内容见各主题文件。超过 200 行会被截断，故只放导航 + 核心速查。

## 1. 主题文件导航表

| 文件 | 内容 |
|------|------|
| `01_仓库总览.md` | 仓库定位、目录结构、关键配置、版本发布、关键约定、常见坑 |
| `02_技能架构与分桶.md` | 分桶规则、user-invoked vs model-invoked 分类轴、技能间依赖约定 |
| `03_工程技能详解.md` | engineering 20 个技能（user-invoked + model-invoked）、主流程 |
| `04_生产力技能详解.md` | productivity 7 个技能、grilling 原语、writing-for-agents |
| `05_其他分桶技能.md` | misc/in-progress/deprecated 技能 |
| `06_文档体系与发布.md` | docs 页结构、写作规范、发布机制、安装路由 |
| `07_脚本与工具链.md` | scripts 脚本、.claude-plugin、.agents、版本同步 |
| `08_复盘与自我进化体系.md` | 任务五阶段、四级复盘、复盘自动化、工作日志、知识沉淀 |
| `09_工程纪律与设计哲学.md` | TDD、Bug 诊断、双轴 CodeReview、Fowler 坏味道、Deep Module、领域建模 |
| `10_系统排障与经验.md` | 云桌面黑屏（GStreamer 缺解码插件）、DevChat LD_LIBRARY_PATH 干扰、dpkg 单位异常 |

## 2. 核心结论速查

### 仓库是什么
- **mattpocock-skills v1.3.1**：Matt Pocock 的 agent 技能库，用于"真实工程"，非 vibe coding。
- 5 个分桶：engineering（工程）/productivity（生产力）/misc（保留）/in-progress（beta）/deprecated（弃用）。
- 推广技能 27 个（engineering 20 + productivity 7），需在 README + plugin.json + docs 登记。

### 最核心分类轴：调用方式
- **User-invoked**：仅人类触发。配置 `disable-model-invocation: true` + `policy.allow_implicit_invocation: false`。任何技能都不能调用它。
- **Model-invoked**：模型或用户触发。默认配置。可被其他技能经 Skill tool 调用。
- 技能间依赖 = `Call the Skill tool with "name"`（仅对 model-invoked 有效）。

### 主流程
`grill-with-docs → to-spec → to-tickets → implement(或 implement-spec) → code-review → retro`

### 工程纪律关键词
- **TDD**：红-绿-重构，一次一垂直切片。
- **Bug 诊断**：先建反馈回路→最小化→假设→插桩→修复→回归。
- **双轴 CodeReview**：Standards（质量）+ Spec（功能），并行分开报告。
- **Deep Module**：大量功能藏在小接口后，干净 seam，可经接口测试。
- **Fowler 12 坏味道**：Mysterious Name / Duplicated Code / Feature Envy / Data Clumps / Primitive Obsession / Repeated Switches / Shotgun Surgery / Divergent Change / Speculative Generality / Message Chains / Middle Man / Refused Bequest。

## 3. 关键路径速查

| 用途 | 路径 |
|------|------|
| 技能本体 | `skills/<bucket>/<name>/SKILL.md` |
| 文档页 | `docs/<bucket>/<name>.md` |
| 插件清单 | `.claude-plugin/plugin.json` |
| 调用规范 | `.agents/invocation.md` |
| docs 写作规范 | `.agents/writing-docs.md` |
| 安装措辞 | `.agents/install-block.md` |
| 工作规则 | `task/task_rule.md` |
| 对话模板 | `task/task_prompt.md` |
| 复盘脚本 | `task/review/review_prepare.sh` |
| 工作日志 | `work_log/YYYYMMDD.md` |
| 记忆 | `memory/self_evolution.md` |

## 4. 工作准则（速记）

- **一次任务，三份产出**：交付物 + 工作日志 + 知识条目。
- **先把事做对，再求快**；**验证优先，不凭猜测**。
- **任务五阶段**：接单分类 → 规划 → 执行验证 → 复查 → 沉淀提交。
- **四级复盘**：单任务即时（全自动）/ 每日 / 每周 / 每月（半自动，cron 备数据 + 用户触发）。
- **复盘一句话触发**：说"进行daily/weekly/monthly复盘"，agent 先跑 review_prepare.sh 刷新数据再反思。
- **工作日志日期**：用系统 `date`，非对话上下文日期。

## 5. 待办 / 遗留
- 暂无。

## 6. 待验证假设
- 暂无。