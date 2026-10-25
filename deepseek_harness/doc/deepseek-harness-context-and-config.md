# DeepSeek Harness：上下文长度影响与配置文件位置

> 记录时间：2026-10-25 13:35（Asia/Shanghai）
> 实例：DeepSeek Harness `0.2.1-rc.3`，源码 checkout `master@22ddee37`（2026-10-23）
> 运行方式：`node /usr/bin/pnpm dsh web`（tsx 直跑源码，非 npx 编译包）
> `DSH_HOME=/home/lsz/.dsh`，`DSH_PROFILE=web`

本文分两部分：先讲上下文长度到底在影响什么、设长设短的取舍；再给出本机所有配置文件的准确位置与层叠关系。所有路径和数值均为 2026-10-25 实测，可用文末命令自行复核。

---

## 一、上下文长度是干什么的

**上下文 = 这一次请求真正发给模型的全部内容**：

```
系统提示词 + 工具清单 + 历史消息 + 工具结果 + 当前用户输入
```

模型本身没有跨请求记忆，每个回合都把这一整包重新发过去。所以「上下文大小」不是存档容量，而是**每轮都要重新付一次费的"工作台面积"**。

它决定三件事：

1. **能记住多少** —— 早期读过的代码、定过的方案、跑出的报错是否还在视野内；
2. **会不会被打断** —— 超过阈值触发自动压缩（compaction），把较早历史摘要化，摘要必然丢细节；
3. **每轮多贵、多慢** —— 输入 token 数、首 token 延迟、以及模型的注意力集中度。

### 1.1 设长与设短的影响对比

| 维度 | 窗口设大（如 1M） | 窗口设小（如 32k） |
|---|---|---|
| 单轮输入成本 | 高，每轮重发全量历史 | 低 |
| 累计成本 | N 轮对话、上下文线性增长时，总输入 ≈ O(N²)；缓存命中能压低单价但压不掉总量 | 总量低，但压缩与重复读取会反复烧 token |
| 延迟 | 首 token 更慢（prefill 更久），长上下文吞吐受限 | 快 |
| 信息保真 | 压缩次数少，细节留得住 | 频繁压缩，摘要丢细节、丢边界条件 |
| 重复劳动 | 少，读过的内容仍在上下文里 | 多：文件反复 read、命令反复跑，可能反而更贵 |
| 模型质量 | 过长后注意力稀释、"lost in the middle"，旧噪声干扰判断 | 聚焦、噪声少，但容易忘掉早期决策 |
| 请求缓存 | 前缀稳定时命中率高；压缩一次整段前缀失效 | 前缀短，失效代价小 |
| 出错风险 | 声明容量超过后端真实容量 → 溢出 / 413 | 声明过小 → 压缩过早甚至无法计算预算（见 2.3） |

### 1.2 成本模型

一次任务的输入 token 总量近似为：

```
总输入 ≈ Σ(每轮上下文长度) ≈ 轮数 N × 平均上下文
```

若上下文随轮次线性增长，平均上下文 = N·c/2，则总输入 ≈ N²·c/2。举例：100 回合、平均 30 万 token → 约 3000 万输入 token；即使每轮只增 3k，平均也有约 15 万。

推论：

- **开新会话做新任务**，通常比在一个超长会话里硬撑更便宜、更快；
- **子代理（subagent）是低成本的上下文隔离手段**，把重读取、重搜索放到独立上下文里做，主会话只收回结论；
- 压缩本身也要花钱：摘要调用要把待压缩的历史再读一遍，是一次性额外成本。

---

## 二、本机当前的实际数值

### 2.1 声明容量：1,000,000

- 当前模型：`deepseek-official / deepseek-v4.1-flash`，`reasoningEffort: high`
- 该模型条目未单独声明 `contextWindow`，回落到 `defaultContextWindow`，默认 **1,000,000**
- 同文件中 `deepseek-flash`、`deepseek-v4-pro` 显式声明 `contextWindow: 1000000`
- 会话事件日志中可验证：`session.v4.jsonl.zstd` 内 `"contextWindow":1000000`（6 处）

### 2.2 自动压缩触发点：约 678,464

压缩默认策略（`compaction-basic`，代码默认值）：

```
thresholdRatio = 0.8
headroomTokens = 65,536
retainRatio    = 0.16
maxTokens      = 65,536
```

触发阈值计算（`packages/compaction/compaction-basic/src/config.ts`）：

```
threshold = min(contextWindow × 0.8, contextWindow − 输出预留 − headroom)
          = min(1,000,000 × 0.8, 1,000,000 − 256,000 − 65,536)
          = min(800,000, 678,464)
          = 678,464
```

其中「输出预留」= 请求的 `maxTokens`，未显式指定时取适配器默认 `DEFAULT_MAX_TOKENS = 256,000`。触发后保留最近约 16% 的原文，其余摘要化。

> 注意：阈值统计的是 token meter 估算的 **prompt 压力**，含系统提示词与工具定义，因此真正可用的历史消息量小于阈值。

### 2.3 一个容易踩的坑

`contextWindow` 是声明值，压缩按它做预算，而输出预留和 headroom 是固定扣减。若只把窗口改小：

```
窗口 ≤ 256,000 + 65,536 = 321,536
```

则消息预算 ≤ 0，压缩会抛 `TargetPressureConfigError`（"leaving no pressure budget"）。要把窗口降到 128k，必须同时调小模型 `maxTokens` 和压缩的 `headroomTokens`。

另外，声明容量若大于后端真实容量，会在请求时溢出报错，不会自动降级。

---

## 三、可以调什么、怎么调

### 3.1 模型容量

```yaml
- id: llm-deepseek
  name: "@deepseek-ai/dsh-llm-deepseek-api-key"
  config:
    defaultContextWindow: 1000000   # 未单独声明的模型走这个
    models:
      - id: deepseek-v4.1-flash
        contextWindow: 128000       # 给具体模型单独设
        maxTokens: 32768            # 单请求输出上限（同时是压缩的输出预留）
```

### 3.2 压缩策略（`compaction-basic`）

| 键 | 默认 | 含义 |
|---|---|---|
| `thresholdRatio` | 0.8 | 达到窗口的百分之多少开始压缩 |
| `headroomTokens` | 65,536 | 为输出/工具结果预留的余量 |
| `retainRatio` | 0.16 | 压缩后保留最近消息的比例 |
| `retainTokens` | — | 直接指定保留 token 数（优先级高于 ratio） |
| `maxTokens` | 65,536 | 摘要调用自身的输出上限 |
| `auto` | true | 关掉后只能手动 `/compact` |
| `modelPolicies[]` | — | 按 `provider`/`model` 精确覆盖上述任意键 |
| `summarizationProvider` / `summarizationModel` | — | 换用更便宜的模型做摘要 |

示例：

```yaml
- id: compaction-basic
  config:
    thresholdRatio: 0.8
    headroomTokens: 65536
    retainRatio: 0.16
    modelPolicies:
      - provider: deepseek-official
        model: deepseek-v4.1-flash
        thresholdRatio: 0.6
```

### 3.3 调整建议

- `thresholdRatio` 取 0.75–0.85；
- `headroomTokens` 至少覆盖一次工具输出的峰值；
- `maxTokens` 按实际需要设，默认 256k 会直接吃掉四分之一窗口；
- 长任务优先用子代理隔离，而不是无限拉长主会话；
- 会话中可随时 `/compact` 手动压缩；不相关的任务开新会话。

---

## 四、配置文件的确切位置（实测）

### 4.1 用户侧：`$DSH_HOME = /home/lsz/.dsh`

| 路径 | 作用 | 状态 |
|---|---|---|
| `/home/lsz/.dsh/profiles/web/cordis.patch.yml` | **唯一需要手改的层**：模型清单、contextWindow、baseURL、默认模型、权限预设、主题 | 存在，1786 B |
| `/home/lsz/.dsh/profiles/web/package.json` | profile 清单：`bundles: [dsh-base, dsh-web-app]` | 存在 |
| `/home/lsz/.dsh/profiles/web/cordis.yml` | 根条目列表，固定为空 `[]`，不要编辑 | 存在 |
| `/home/lsz/.dsh/profiles/web/pnpm-workspace.yaml` | profile 内装插件用的 workspace 根 | 存在 |
| `/home/lsz/.dsh/cordis.patch.yml` | home 级补丁，对所有 profile 生效（最终层） | **不存在**，需要时自建 |
| `/home/lsz/.dsh/.credentials.yaml` | 凭据引用：`DEEPSEEK_API_KEY`、浏览器会话密钥（600 权限，勿外传） | 存在 |
| `/home/lsz/.dsh/storages/workspace.json` | 工作区/会话注册表（工作区路径、默认会话），非启动配置 | 存在 |
| `/home/lsz/.dsh/sessions/<workspace-slug>/session-<id>/session.v4.jsonl.zstd` | 会话事件日志，含 `request/context` 的 contextWindow 记录 | 存在，约 224 K |

说明：profile 目录内**没有 `node_modules`**，未装额外插件；所有 bundle 从源码 checkout 解析。

### 4.2 源码侧：`/home/lsz/software/ai/love-deepseek-harness`

| 路径 | 作用 |
|---|---|
| `packages/bundle/base/cordis.patch.yml`（第 527 行） | 主机平面默认层，`llm-deepseek` / `llm-deepseek-account` 出厂条目 |
| `packages/bundle/web-app/cordis.patch.yml`（第 545 行） | web 主机层；在此禁用 compaction，交由预设重新插入 |
| `packages/bundle/web-app/presets/standard.patch.yml`（第 73 行） | 默认预设 `standard` 插入 `compaction-basic`（无 config → 全默认值）；同目录另有 ptc / minimal / cordis 三套 |
| `packages/llm/llm-deepseek/src/defaults.ts`（第 6、8 行） | `DEFAULT_CONTEXT_WINDOW = 1_000_000`、`DEFAULT_MAX_TOKENS = 256_000` |
| `packages/compaction/compaction-basic/src/config.ts`（第 20、75 行起） | 压缩默认值：0.8 / 65,536 / 0.16 / 65,536 |
| `apps/cli/src/args.ts`、`apps/cli/src/profile-boot.ts` | 启动参数文法与 profile 装配 |
| `apps/cli/config/examples/*.yml` | 可选 overlay 示例（MCP 记忆、ptc），非默认配置 |

### 4.3 层叠顺序

```
各 bundle 默认层
  → $DSH_HOME/profiles/<name>/cordis.patch.yml
  → $DSH_HOME/cordis.patch.yml
  → --patch 命令行覆盖
```

后者覆盖前者。`--dump-config` 输出里会用注释标出每一段来自哪个 bundle、被哪个文件打过补丁。

### 4.4 验证合成结果

```bash
cd /home/lsz/software/ai/love-deepseek-harness
node /usr/bin/pnpm dsh --profile web --dump-config
```

输出 1274 行，可确认：`agent-default-model` = `deepseek-official / deepseek-v4.1-flash / high`；两个模型 `contextWindow: 1000000`；`deepseek-v4.1-flash` 未声明 → 回落 1M；`preset-standard` 中 `compaction-basic` 无 config → 默认值。

完整输出已另存为同目录 `dsh-web-effective-config.yml`。

### 4.5 环境变量

| 变量 | 值 / 含义 |
|---|---|
| `DSH_HOME` | `/home/lsz/.dsh` |
| `DSH_PROFILE` | `web` |
| `DSH_PROFILE_DIR` | `/home/lsz/.dsh/profiles/web` |
| `DSH_WEB_URL` | `http://127.0.0.1:3080` |
| `DSH_SESSION_ID` | 当前会话 id |
| `DSH_SHELL` | `1`，标记处于 agent shell 中 |

API key 通过 `~/.dsh/.credentials.yaml` 的 `refs: DEEPSEEK_API_KEY` 解析，或由启动环境导出。

### 4.6 运行时事实

- 进程：`node /usr/bin/pnpm dsh web`，tsx 直接执行 `apps/cli/src/bin.ts`，即**源码运行**；
- 版本 `0.2.1-rc.3`，与早期 npx 缓存实例（`~/.npm/_npx/...`，`0.2.1-rc.2`，编译后 `lib/*.js`）不是同一份代码；
- 工作区：`/home/lsz/文档/deepseek-harness/default-workspace`，与 checkout 分离。

---

## 五、附录：关键代码位置

| 主题 | 位置 |
|---|---|
| 默认容量与输出上限 | `packages/llm/llm-deepseek/src/defaults.ts` |
| 压缩默认参数 | `packages/compaction/compaction-basic/src/config.ts` |
| 压缩触发与执行 | `packages/compaction/compaction-basic/src/index.ts` |
| 阈值计算 | `resolveCompactSpec()`（同上 config.ts） |
| 输出预留来源 | `reservedCompletionTokens()`（index.ts，取 envelope maxTokens 否则适配器默认） |
| 模型清单与上下文解析 | `packages/llm/llm-deepseek-api-key/src/index.ts` |
| 默认预设内容 | `packages/bundle/web-app/presets/standard.patch.yml` |

---

## 六、复核清单

```bash
# 1. 当前实例与版本
ps aux | grep "dsh web" | grep -v grep

# 2. 会话里记录的 contextWindow
zstd -dc ~/.dsh/sessions/*/session-*/session.v4.jsonl.zstd | grep -o '"contextWindow":[0-9]*' | sort | uniq -c

# 3. 合成后的完整配置
cd /home/lsz/software/ai/love-deepseek-harness && node /usr/bin/pnpm dsh --profile web --dump-config

# 4. 用户补丁层
cat ~/.dsh/profiles/web/cordis.patch.yml
```
