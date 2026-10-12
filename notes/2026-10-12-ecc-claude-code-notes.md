# ECC 学习笔记：Claude Code 的资产体系与“文档即一切”设计

> 学习日期：2026-10-12
> 学习对象：[Everything Claude Code（ECC）](https://github.com/affaan-m/ECC)，本地仓库：`ECC/`
> 版本：`ecc-universal 2.2.3` · MIT License
> 配套阅读：《AI Agents in Depth》第 2 章（上下文工程、渐进披露、隔离优于压缩）
> 说明：文中的目录、数量和实现链路是对上述版本的学习记录；升级后应以仓库实际内容和 Claude Code 当前行为为准。

---

## 一、项目速览

ECC(Everything Claude Code)是一个 Claude Code 插件,把工程化流程——plan → test → implement → review → verify → remember → improve——固化到 agent 工作流里。

| 类别 | 数量 | 形态 |
| --- | ---: | --- |
| Agents | 68 | 扁平 `.md`,YAML frontmatter |
| Skills | 293 | 子目录 + `SKILL.md` |
| Commands | 94 | 扁平 `.md`(斜杠命令) |
| Rules | 22 个子目录 | common + 21 种语言 |
| Hooks | JSON 注册 + JS 运行时 | `hooks/hooks.json` + `scripts/hooks/*.js` |
| MCP servers | 30+ 预设 | `mcp-configs/mcp-servers.json` |
| Schemas | 17 | JSON Schema,驱动 manifest 校验 |
| npm dependencies | **仅 4 个** | `@iarna/toml`、`ajv`、`js-yaml`、`sql.js` |

两个关键观察：

1. **运行时并不依赖大量 npm 包。** 依赖主要用于 schema 校验和配置解析；核心资产是 Markdown 提示词与 Node.js 脚本。
2. **`npm run test` 串联了多项 CI 校验。** 例如 `validate-agents`、`validate-skills`、`validate-commands`、`validate-hooks` 和 `validate-install-manifests`，用于检查 manifest 与资产定义是否一致。遗漏 skill 的 frontmatter、给 hook 添加不支持的字段等问题，都可能被校验拦截。

---

## 二、核心心智模型:一份 harness,四种资产

可以用下面的公式概括：

```
agent = model + harness(全局一份,所有实例共享) + agent 定义文件(纯声明,零代码)
```

harness 就是 Claude Code CLI 本体(闭源),它提供三件套,各只有一份:

- **统一 agent loop**(循环控制、工具调度、上下文管理)
- **工具注册表**(内置工具 + MCP 工具)
- **事件系统**(工具执行前后的关卡,即 hook 时机)

四种资产都接入这份 harness：

| 资产 | 角色 | 一句话 |
| --- | --- | --- |
| Agent 文件 | 执行单元 | 定义"这是一个什么岗位的人" |
| Skill | 知识包 | 定义"这类活儿的领域标准是什么" |
| Command | 编排入口 | 定义"这个流程按什么步骤走" |
| Hook | 硬护栏 | 定义"什么事物理上不许发生" |

---

## 三、文档即 agent

### 3.1 核心认知

- agent 文件(如 `agents/planner.md`)**不含任何可执行代码**,是声明式规格。
- 定义一个新 agent,不需要写任何新程序,只需要写一份新规格——**68 个 agent = 68 份配置 × 1 份实现**。
- 类比:Pod YAML 之于 kubelet;SQL 语句之于数据库引擎。你不会说"每个 Pod 都要写一个 kubelet"。

### 3.2 Frontmatter 契约

```yaml
---
name: planner
description: Expert planning specialist... Use PROACTIVELY when users
  request feature implementation, architectural changes...
tools: Read, Grep, Glob     # 工具白名单(harness 裁剪工具表)
model: opus                 # 模型指定
---
(正文整体作为该子会话的 system prompt)
```

spawn 时 harness 做的组装(伪代码):

```python
def spawn_agent(agent_md, task_input):
    fm, body = parse(agent_md)
    system_prompt = compose(
        HARNESS_BASE,          # harness 内置底座,所有实例相同
        body,                  # 该 agent 的角色、流程、自带规则
        load_project_rules(),  # 项目级规则(CLAUDE.md / rules/)
    )
    tools = TOOLS.subset(fm.tools)   # 全表 ∩ 白名单
    return agent_loop(fm.model, system_prompt, tools, Context(), task_input)
```

主对话自己也只是同一个 `agent_loop` 的另一个实例——参数不同(全量工具表、生命周期跟会话走)而已。

### 3.3 Spawn 与上下文隔离

**spawn = 创建并启动一个独立运行的新实例**(系统编程术语,直译"产卵")。它强调"另开一个能独立跑的上下文",一口气做四件事:读配置 → 组装 system prompt → 裁剪工具表 → 新建空上下文并启动 loop。

这直接对应书里的三个概念:

| 书里的概念 | 代码级对应物 |
| --- | --- |
| 子 Agent 隔离替代压缩(2.7.7) | spawn 开独立子会话,父子上下文不共享 |
| worker 工具集收敛 | `tools: Read, Grep, Glob` 白名单 |
| 判官只读摘要 | reviewer 类 agent 的 tools 里没有 Write |

### 3.4 术语链

**主 agent 委派(delegate,决策)→ harness spawn(机械动作,创建并启动子实例)→ 跑完销毁,只回传最终文本**。承载 spawn 动作的工具在 Claude Code 里叫 Task。

---

## 四、agent 与 skill 的边界

### 4.1 本质差异：执行单元 vs. 知识包

| 维度 | Agent | Skill |
| --- | --- | --- |
| 本质 | 执行单元,独立的 agent loop 实例 | 知识包,注入当前上下文 |
| 上下文 | 派生隔离子会话 | 没有自己的上下文,进宿主的 |
| agent loop | 有,自己跑完整循环 | 没有,借用宿主的 |
| 工具 | 自己的白名单(`tools`) | 可选 `allowed-tools` 临时限权 |
| model | 可单独指定 | 无,用宿主的 |
| token 成本 | body 只花在隔离上下文里 | body 直接占宿主的 token 预算 |
| 并行 | 可多个同时跑 | 无并行概念 |
| 生命周期 | 跑完销毁,只回传结果 | 留在上下文里直到压缩/结束 |

类比:**agent 是进程,skill 是动态链接库**。口语版——agent 是"另派一个人去干活",skill 是"给当前这个人递一本操作手册"。

判别口诀：**需要独立执行并产出结果时用 agent；需要指导当前 agent 如何执行时用 skill。**

### 4.2 为什么边界感觉模糊

1. **格式同源**:都是 frontmatter + markdown 正文,长得一模一样。边界不在文件里,在 harness 拿到文件后做的事不同。
2. **格式在互相渗透**:skill 有了 `allowed-tools`;agent 的 description 写法和 skill 一致。
3. **有意混用**:ECC 的 CLAUDE.md 明确规定"spawn 子 agent 时,把对应 skill 的约定注入 agent 的 prompt"——**skill 喂给 agent**。

最直接的判别特征是：**frontmatter 中定义了 `tools` 和 `model` 的通常是 agent**。skill 不独立调用模型或工具，因此这两个字段对它没有相同含义。命名习惯也能提供线索：agent 名称通常对应**岗位**（如 `code-reviewer`、`planner`），skill 名称通常对应**领域**（如 `react-patterns`、`tdd-workflow`）。

### 4.3 交换测试(证明两者正交)

- 删掉所有 skill → agent 照样能跑,只是不懂领域约定,干得糙。
- 删掉所有 agent → skill 照样能用,正文加载进主对话,主 agent 自己干——只是没了隔离、并行和工具裁剪。

```
agent 解决:谁来执行(隔离、并行、工具集、模型)
skill 解决:执行的时候知道什么(领域知识、流程标准)
```

### 4.4 三层分工的真实链路

以 `commands/review-pr.md` 为例:

```
/review-pr(命令,编排层)
   ├──→ code-reviewer         (agent,隔离执行)
   ├──→ comment-analyzer      (agent,并行)
   ├──→ pr-test-analyzer      (agent,并行)
   ├──→ silent-failure-hunter (agent,并行)
   └──→ ... 共 6 个
        每个 agent 的 prompt 里注入对应 skill 的约定
        (React 项目就注入 react-patterns 的规范)
```

**command(编排)→ agent(执行)→ skill(供知识)**,三者可以使用相似的文件格式，却有不同的运行时语义。

---

## 五、skill 的随附脚本

### 5.1 机制:不注册为工具

skill 自带的脚本(`skills/<name>/scripts/`)**不注册进工具表**,是 skill 的随附载荷。以 `skills/ios-icon-gen/` 为例,SKILL.md 正文里写着:

```bash
# Search all collections
$SKILL_DIR/scripts/iconify_gen.sh search "receipt"
```

执行链路四步:

1. skill 被激活,正文进上下文(此刻脚本只是磁盘文件,零成本)。
2. LLM 读到这行命令,结合任务**判断**"现在该跑它了"。
3. 发起一次**普通的 Bash 工具调用**,参数就是那行命令——Bash 是工具表里本来就有的,没有任何新注册。
4. 脚本确定性执行,stdout 作为工具结果回流上下文。

一句话概括：**skill = 说明书 + 随附工具；执行仍通过 Bash，脚本则由 skill 自己提供。**

### 5.2 三个设计点

1. **脚本源码从不进上下文,只有输出进**——渐进披露的最深层,比 references/ 还深。200 行脚本在上下文里只占 3 行输出。
2. **又是判断/确定性分工**:SKILL.md 管"何时跑、跑哪个、怎么解读"(LLM 擅长);脚本管"精确执行"(LLM 手拼 API 参数必出错,脚本零幻觉)。
3. **脚本是供应链攻击面**:任何 skill 都能带任意脚本。ECC 的 CI 有 `scan-supply-chain-iocs.js` 专扫可疑指标;装第三方 skill 前值得看一眼它的 scripts/ 目录。

### 5.3 代码在 agent 系统里的三种存在方式

| | skill 脚本 | MCP 工具 | hook |
| --- | --- | --- | --- |
| 谁发起 | agent 自己判断 | agent 主动调用 | harness 事件触发 |
| 注册成本 | **零**,激活前不存在 | schema 常驻每个上下文 | 零,但全局生效 |
| 执行方式 | 经 Bash 间接执行 | 直接作为工具调用 | harness 直接跑 |
| 适合 | 单一流程内的确定性步骤 | 跨任务的通用能力 | 必须执行的护栏 |

选择准则：**只有某个流程需要的确定性步骤用随附脚本；多个 agent 共享的能力可做成 MCP 工具；必须强制执行的约束应放在 hook 等拦截机制中。**

---

## 六、规则约束三级强度

### 6.1 三级强度

| 规则的性质 | 通道 | 强度 | 例子 |
| --- | --- | --- | --- |
| 违反了就根本没有能力做 | **裁剪**(tools 白名单) | 物理不可能 | 判官不得写文件 → tools 里不放 Write |
| 必须百分之百拦住 | **拦截**(hook + 权限) | 不可绕过 | 抓取后必须记来源日志 → PostToolUse hook |
| 需要临场判断的 | **嘱咐**(prompt 文本) | 软约束 | "引用尽量选近三年的一手来源" |

选择排序:**能裁剪就不拦截,能拦截就不嘱咐。** 嘱咐依赖模型配合,是最后手段;但判断类规则也没法硬编码,三级各有领地。

### 6.2 关卡的三层归属:不是 agent 自己的代码

"执行前关卡(hook 拦截 + 权限校验)"由三种不同来源拼成,agent 文件一层都不在其中:

| 层 | 谁写的 | 在哪 | 作用 |
| --- | --- | --- | --- |
| 检查点本身 | harness 作者(Anthropic) | Claude Code CLI 内部 | 决定"执行前要过哪些关口",全局一份 |
| 分发与引导 | 插件作者(ECC) | `hooks/hooks.json` + `plugin-hook-bootstrap.js` | 声明哪些事件要拦、转成脚本进程 |
| 规则逻辑 | 插件作者 / 你 | 具体 hook 脚本 | 判断 block 还是放行 |

ECC 真实链路:`hooks.json` 里 `"PreToolUse": [{"matcher": "Bash", "hooks": [{"type": "command", "command": "node -e ..."}]}]` → harness 在工具执行前触发 → node 解析 ECC 根目录(插件位置不固定)→ bootstrap 读 stdin(harness 送来的调用上下文)→ spawn 子进程跑规则脚本 → **exit 2 = block**,拒绝理由回流给 LLM。

关键设计:**harness 只提供"开孔时机",不提供"政策"**。机制固化在 harness(全 agent 共享),政策外置成脚本(随时可换)——同一个 harness,换一套 hook 脚本就是完全不同的约束政策。

### 6.3 裁剪的可信度取决于工具集的"表达力"(重要的坑)

`code-reviewer` 的 tools 是 `Read, Grep, Glob, Bash`(Bash 为了跑 `git diff`)。于是"审查员不得改代码"**只靠工具表是不成立的**——Write 虽不在表里,但 LLM 可以 `bash: echo "..." > src/foo.ts` 绕过。

**白名单的可信度 = 工具集的表达力上限。** 有 Bash 这种万能工具的地方,白名单语义会被击穿,硬约束必须"裁剪 + 关卡"双保险。

### 6.4 真实样例:一条必须是"拦截"的规则

`scripts/hooks/block-no-verify.js`(PreToolUse):拦 `git --no-verify` 和 `-c core.hooksPath=`,防 AI agent 绕过 pre-commit/commit-msg/pre-push 钩子,且明确设计成不易被 `ECC_DISABLED_HOOKS` 关闭。

为什么不能是嘱咐?**防"绕过行为"的规则,不能依赖模型配合**——需要配合才能生效的规则,防不了不配合。hook 是物理插在 LLM 决策和执行之间的门,连绕的入口都没有。推广:**凡是"防坏人/防偷懒"性质的规则,都必须是硬的。**

---

## 七、command 编排层:文档即工作流

### 7.1 没有编排引擎

command 编排层**没有调度器代码、没有 DAG、没有状态机**。`commands/review-pr.md` 全文 37 行,编排"6 个 agent 并行审查"的引擎就是一段数字列表:

```markdown
## Steps
1. Identify the PR: use `gh pr view` to get details and diff
2. Find project guidance: look for CLAUDE.md, lint config, repo conventions
3. Run specialized review agents:
   code-reviewer / comment-analyzer / pr-test-analyzer /
   silent-failure-hunter / type-design-analyzer / code-simplifier
4. Aggregate results: dedupe overlapping findings, rank by severity
5. Report findings grouped by severity
```

### 7.2 harness 的机械部分只有三件事

1. **注册表查找**:启动时扫描 `commands/` 目录,建立 `/命令名 → 文件` 映射。
2. **参数替换**:把用户输入填进正文里的 `$ARGUMENTS` 占位符。
3. **正文注入**:把替换后的正文塞进主对话上下文,等价于"用户发了这一大段话"。

之后**没有新引擎接管**——还是主 agent 那个统一的 loop,它读到 Steps,自己跑 `gh pr view`(经 Bash)、逐个委派(经 Task)、按第 4-5 步汇总。**所谓编排,就是模型在解读一份文档。**

### 7.3 "并行"的机械来源

正文只写了一句 "Run specialized review agents",并行来自:**主 agent 在一次模型响应里发出多个 Task 工具调用,harness 收到后并发 spawn**。"并行"不是编排层实现了并发调度,而是主 agent 一次性把委派都发出去,harness 天然并发执行。

### 7.4 可靠性光谱

| 方式 | 样例 | 特点 |
| --- | --- | --- |
| 纯 prompt 编排 | `/review-pr` | 灵活,但依赖模型配合——可能跳步、漏派 |
| 脚本驱动编排 | `/epic-*`、`/cost-report`(正文里直接调 node/python 脚本) | 确定性流程骨架交给脚本,模型只做判断环节 |

ECC 在 review 一个场景铺了三层命令做路由:`/code-review`(单人清单)→ `/review-pr`(多 agent 编排)→ `/orch-review`(对抗验证),三者的 description 互相指路,模型按任务重量选档。

### 7.5 frontmatter 字段盘点(94 个命令)

| 字段 | 次数 | 作用 |
| --- | ---: | --- |
| `description` | 94/94 | 路由(和其他资产一样互相指路) |
| `argument-hint` | 12 | 输入框 UI 自动补全提示 |
| `allowed-tools` | 2 | 命令执行期间临时限权 |
| `disable-model-invocation` | 2 | 禁止模型自调,只有用户能敲 |

---

## 八、渐进披露与三层计费

### 8.1 两种资产的披露层数不同

- **Agent 两阶段**:启动时只把所有 frontmatter 的 `name + description` 注入主对话(常驻);被选中后正文一次性全量进子会话的 system prompt,没有第三层。
- **Skill 三阶段**:层一 name + description 目录常驻前缀;层二 SKILL.md 正文按需加载;层三 references/、examples/ 用 Read 选择性深入。

层数差异的经济学解释:**skill 的正文进共享上下文,每个 token 都是主对话实付成本,必须细切;agent 的正文进隔离子会话,主上下文不付钱,一次全量反而最简单。** 2.5.1 的三层结构是节流,2.7.7 的隔离是分仓——同一个问题的两个方向。

### 8.2 三层计费账本(写任何内容前先问:这段文字在哪个计费层)

| 层 | 计费方式 | 写作纪律 |
| --- | --- | --- |
| 层 1 · description | **常驻成本**,每个会话都付 | 极简:能力一句话 + 触发条件,能省一字是一字 |
| 层 2 · SKILL.md 正文 | **按需成本**,激活时付一次 | 写到够用,清楚优先于简短——过短会把成本转成多轮试错 |
| 层 3 · references/、scripts/ | **深层成本**,读到才付 | 长示例、详细规范、确定性脚本全部外移到这里 |

### 8.3 description 的最佳写法

公式:**一句能力 + 一串触发场景**,两个半句缺一不可。

- 只写功能("React 模式库")→ 没有触发信号,无法路由。
- 只写触发("写 React 时用")→ 不知道提供什么,相邻场景难判断。

标准样例(tdd-workflow):能力句(建立测试先行的开发纪律,覆盖红绿重构循环)+ `Use when starting new features, fixing bugs, or refactoring`。

### 8.4 实测数据与一个纠偏

- ECC 293 条 skill description 实测:最短 136 / 中位 299 / 最长 880 字符。粗算 ≈ **2 万+ tokens 常驻前缀**,再加 68 条 agent description——description 的经济性影响每个会话。
- **纠偏**:「description 必须以 Use when 起头」**不是 ECC CI 强制的**(grep `scripts/ci/` 无此校验)。它是社区约定(源自 Anthropic skill-creator 指南),ECC 293 个 skill 事实上全部遵守,但靠写作纪律不靠校验器。想让它是硬约束,需要自己写校验器。

---

## 九、调研 agent 落地清单

### 9.1 总体结构:通用 worker 执行器 + 角色配置文件

**不要给每个角色写一个 Python 文件。** 写一个通用 worker 执行器(一个 loop + 工具注册 + 子会话派生,约百来行),planner / 检索 worker / 判官全部做成配置文件。收益:加角色 = 加文件;prompt 进版本控制可 diff 可回滚;可以上 CI 校验防配置漂移。

### 9.2 资产划分

| 你的资产 | 做成 | 判据 |
| --- | --- | --- |
| 检索 worker、判官、planner | agent 文件 | 独立干活、有自己的工具集、跑完回传结论 |
| 检索质量打分标准、引用格式规范、数据源用法说明 | skill 文件 | 不执行,只指导执行者;同一份标准可注入判官也可注入 planner |
| 调研工作流(拆题 → 派检索 → 判官评分 → 汇总) | 一份编排文档(或 command) | 步骤是 prose,靠主 loop 解读 |

### 9.3 规则三级分派

| 规则 | 实现 | 为什么 |
| --- | --- | --- |
| 判官不得写工作区 | 裁剪(不给 Write/Edit)+ 关卡(Bash 写操作校验) | 判官若需 Bash 跑统计,单靠裁剪不够 |
| planner 不得直接改代码 | 裁剪(tools 只给 Read/Grep/Glob) | 物理不可能,无需关卡 |
| 检索 worker 只能写自己的目录 | 关卡(PreToolUse 校验路径前缀) | 防越界,必须硬 |
| 每次抓取必须记来源日志(URL/时间/摘要进 sources.jsonl) | 关卡(PostToolUse 自动追加) | 防偷懒,必须硬 |
| 优先近三年一手来源 | 嘱咐(判官 SKILL.md 评分标准) | 判断类,无法硬编码 |

### 9.4 skill 分层写作纪律

1. description 写"评分标准与打分口径。Use when 评估检索结果质量或规划检索任务时"——一句话带触发,锱铢必较。
2. 评分细则正文写全,别怕长——只在判官激活时付一次。
3. arXiv/Semantic Scholar 的完整字段表和长示例放 references/;API 调用与响应解析做成 scripts/(随附脚本,杜绝 LLM 拼错参数)。
4. 团队协作的话,自己补一条 ECC 没做的校验:CI 检查每条 description 含触发条件、长度有上限——把约定升格成护栏。

### 9.5 编排注意

- "判官必须在所有 worker 完成后才启动"这类**必须保证的顺序**,不要只写在 prose 里——要么用脚本做屏障,要么用 hook 校验。prose 里的顺序是建议,不是保证。
- 并行派检索 worker:主 agent 一次响应发出多个 spawn 调用即可,天然并发。

---

## 附录:ECC 里非显而易见的模块

| 模块 | 一句话说明 |
| --- | --- |
| `ecc2/` | Rust 子项目(ratatui TUI + rusqlite + git2),定位 "ECC 2.0 Agentic IDE 控制面",与 Node.js/JSON 路线并行的另一套实现 |
| `ecc_dashboard.py` | Tkinter 跨平台 GUI,扫描 agents/ 的 frontmatter 渲染管理面板 |
| `agent.yaml` | gitagent 协议 manifest(spec 0.1.0),把目录真相导成可移植协议,但原始文件仍是权威 |
| `manifests/` | 5 套安装切片(components/modules/profiles…),安装器靠这层决定装什么 |
| `install.sh` | 42 行包装器:解析 symlink、`npm install --ignore-scripts`(防 RCE)、Git Bash 路径转码 |
| `the-shortform-guide.md` / `the-longform-guide.md` | 作者实战手册:基础设置 / token 经济、记忆持久化、并行化策略、verification patterns——理解设计哲学的入口 |
| `skills/continuous-learning-v2/` | 带运行时的 skill(自带 agents/hooks/scripts/config.json),Stop/PreToolUse/PostToolUse 观测器做模式抽取 + 置信度评分 |

---

## 一页速记

- **一份 harness**:统一 loop + 工具注册表 + 事件系统,所有实例共享。
- **四种文档资产**:agent = 文档即执行单元;skill = 文档即知识包(可带随附脚本);command = 文档即工作流;hook = 配置即护栏。
- **规则三级**:裁剪(物理不可能)> 拦截(harness 层关卡,外部脚本)> 嘱咐(prompt 文本)。能裁剪就不拦截,能拦截就不嘱咐;有 Bash 的地方白名单不可信,要补关卡。
- **渐进披露的经济学**:常驻层(description)锱铢必较,按需层(SKILL.md)够用就好,深层(references/scripts)随便长。下笔前先问:这段文字在哪个计费层。
- **判断/确定性分界线**(今天出现了四次):判断性逻辑写提示词;确定性逻辑写脚本/hook。
