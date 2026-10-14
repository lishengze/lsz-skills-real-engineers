# Prompt.md — 学习引导与跨会话知识迁移

> 本文件是**学习引导器**。新会话/新主机的 Agent 读一遍本文件即可继承全部历史积累。每次沉淀新知识后，同步更新本文件的「学习路径 / 学习要求 / 学习完成标准」。

## 1. 本仓库是什么

**mattpocock-skills（lsz-skills-real-engineers）**：Matt Pocock 的 agent 技能库，用于"真实工程"。5 个分桶（engineering/productivity/misc/in-progress/deprecated），推广技能 27 个。核心是小型、可插拔、可组合的技能，修复 agent 常见失败模式。

## 2. 学习路径（分阶段）

### 阶段 A：掌握仓库基础（必学）
1. 读 `01_仓库总览.md`：仓库定位、目录结构、关键配置、版本发布。
2. 读 `02_技能架构与分桶.md`：分桶规则 + user-invoked vs model-invoked 分类轴（最核心）。
3. 读 `08_复盘与自我进化体系.md`：任务五阶段、四级复盘、工作日志规则。

### 阶段 B：掌握技能（核心）
4. 读 `03_工程技能详解.md`：engineering 20 技能 + 主流程。
5. 读 `04_生产力技能详解.md`：productivity 7 技能 + grilling 原语 + writing-for-agents。
6. 读 `05_其他分桶技能.md`：misc/in-progress/deprecated。
7. 读 `09_工程纪律与设计哲学.md`：TDD、Bug 诊断、双轴 CodeReview、Deep Module、领域建模。

### 阶段 C：掌握发布与工具链
8. 读 `06_文档体系与发布.md`：docs 页结构、写作规范、安装路由。
9. 读 `07_脚本与工具链.md`：scripts、.claude-plugin、.agents、版本同步。

### 阶段 D：实践应用
10. 用 `ask-matt` 路由器判断"当前情况用哪个技能"。
11. 实际执行主流程：grill-with-docs → to-spec → to-tickets → implement → code-review → retro。
12. 每次任务后执行复盘（一句话触发），沉淀经验到知识库 + 本文件。

### 阶段 E：系统排障（daily_life 场景）
13. 读 `10_系统排障与经验.md`：云桌面黑屏（GStreamer 缺解码插件）、DevChat LD_LIBRARY_PATH 干扰、dpkg 单位异常、Wayland 渲染注意。
14. 排障通用思路：按"连接→渲染→进程→解码→环境"逐层排除；先看日志/实测，不凭猜测。

## 3. 学习要求（新增知识时补充）

- **技能使用**：用技能前先读其 `SKILL.md`，判断 user-invoked 还是 model-invoked，遵守调用约定。
- **文档写作**：为 agent 写文档遵循 writing-for-agents 规范（Context Pointer / 引导词 / 渐进披露 / 单一事实来源）。
- **工程纪律**：开发遵循 TDD；bug 按六阶段诊断；提交前双轴 CodeReview；设计遵循 Deep Module。
- **复盘**：每次任务后单任务即时复盘；日/周/月复盘用"进行daily/weekly/monthly复盘"触发。
- **知识库**：每次沉淀后更新对应主题文档 + MEMORY.md + Prompt.md。

## 4. 学习完成标准（新增验收点时补充）

- 能不看文档说出：仓库是什么、5 个分桶、user-invoked vs model-invoked 的区别。
- 能说出工程主流程的完整链条。
- 能说出 Fowler 12 种坏味道并识别。
- 能执行"进行daily复盘"的完整流程。
- 能说出如何为新增技能写 docs 页（结构 + 规范）。

## 5. 关键结论速查（高频复用）

- **调用方式**：user-invoked 仅人类能触发、任何技能不能调用；model-invoked 可被 Skill tool 调用。
- **主流程**：grill-with-docs → to-spec → to-tickets → implement/implement-spec → code-review → retro。
- **双轴审查**：Standards（质量）+ Spec（功能），并行分开报告。
- **Deep Module**：小接口 + 大量行为 + 干净 seam + 可经接口测试。
- **复盘一句话触发**：说"进行daily/weekly/monthly复盘"，agent 先跑 `bash task/review/review_prepare.sh <周期>` 刷新数据再反思。

## 6. 学习钩子（每次任务问自己）

1. 这是什么？（技能/模块的定位与职责）
2. 怎么用/怎么配？（关键路径、配置项、命令）
3. 坑在哪？（报错、边界、易错点）