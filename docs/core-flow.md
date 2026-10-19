# Loom 核心流程梳理

> 项目定位：**「对话即生成单页应用」的智能体平台**。前端 Vue3 SPA + 后端 FastAPI，
> 智能体通过工具调用循环在文件沙箱里写文件，生成结果可预览、可迭代、可发布、可克隆。
>
> 本文覆盖三部分：① 端到端核心流程 ② 智能体循环的控制方式 ③ 记忆管理机制。

---

## 一、总体架构

```
前端 SPA (Vue3 + Pinia)                    后端 FastAPI
┌─────────────────────┐   REST + SSE    ┌──────────────────────────────┐
│ api/client.ts (JWT) │ ──────────────► │ routers/{auth,projects,      │
│ api/sse.ts (fetch + │ ◄────────────── │           publish,world}.py  │
│  ReadableStream)    │   data:{json}   │   ├─ agent/ (循环/工具/意图)  │
│ views/ProjectPage   │                 │   ├─ rag/  (模板向量库)       │
└─────────────────────┘                 │   └─ SQLite + 文件沙箱 + Milvus│
                                        └──────────────────────────────┘
```

- 应用装配在 `backend/app/main.py` 的 `create_app` 工厂：把 Settings、模型工厂、嵌入工厂、
  知识库、限流器挂到 `app.state` 供各层读取。
- 前端因 EventSource 无法携带 Authorization，用 `fetch + ReadableStream` 自实现 SSE 客户端
  （`frontend/src/api/sse.ts`），按 `data: {json}` 行协议解析事件。
- 安全边界：`security.py`（JWT + bcrypt）保护数据，`serving.serve_project_file`（沙箱路径
  白名单）保护文件系统。

---

## 二、首建生成主流程

入口：`POST /api/projects/{id}/messages`（`routers/projects.py` 的 `send_message`）。
消息先过**限流**（每用户每小时额度 + 全局并发，超限 429 并附 `Retry-After`），
再按项目模式和状态**阶段分流**。

### 2.1 工程师模式（engineer）

```
用户首条消息 → 需求澄清 (_clarify_stream，分轮问答)
            → 需求共识卡片 → 用户确认 (/consensus/confirm)
            → 工程师智能体生成 (_engineer_stream)
            → 多文件落盘 + 快照 + SSE 事件流
```

### 2.2 团队模式（team）—— 五阶段流水线

```
需求澄清 → 共识确认 → 需求规格 (_spec_stream) → 规格确认 (/spec/confirm)
        → 工单拆解 (_break_stream，产出纵向切片工单清单) → 清单确认 (/tickets/confirm)
        → 检查点串行执行 (_exec_tickets_stream)：
          按依赖序逐单执行，每单完成建一个检查点快照，失败可从该单重试 (/tickets/resume)
```

### 2.3 关键设计点

| 设计点 | 说明 |
|---|---|
| 确认门（gate） | 每阶段产物（共识/规格/工单清单）都是待确认卡片；确认接口在**项目锁内复查状态**，并发双击不会触发两次生成；确认消息以 `kind=*_confirm` 落对话历史 |
| 阶段智能体无文件工具 | 澄清/规格/拆单智能体唯一工具是 `submit_*` 类结构化提交（解析失败把错误回喂模型自行修正）；只有工程师智能体拥有文件工具 |
| 限流只扣一次 | 名额在首条消息时扣定，流水线内的后续阶段（澄清续轮、各确认、拆单）不重复计数 |
| 执行期拦截 | 团队模式工单执行期间，普通消息以引导拦截（`exec_guide`），不污染执行流 |

---

## 三、智能体循环（生成引擎）

核心：`backend/app/agent/loop.py` 的 `run_generation`，所有生成阶段共用。

```
System 提示 + 历史 + 用户输入
  → 模型流式输出（ThinkSplitter 把 <think> 块拆成独立 thinking 事件）
  → 有 tool_calls？
      有 → 沙箱工具执行（write_file / edit_file / read_file / search_templates …）
           → 结果作为 ToolMessage 回喂 → 继续循环（最多 max_steps 步，默认预算 100）
      无 → final_text_hook 收尾核验
           · 返回字符串 → 作为 HumanMessage 回喂，继续循环（防「口头完成」）
           · 返回 None  → yield done，正常出口
  → 循环耗尽 → error(reason="max_steps_exhausted")，路由层据此走兜底收尾
```

### 3.1 是否用 LangChain / LangGraph？

**循环是手写的，没有 LangGraph，也没有 AgentExecutor。** LangChain 只作「零件级」适配：

| 用途 | 是否用 LangChain | 位置 |
|---|---|---|
| LLM 客户端（`ChatOpenAI` + `bind_tools` + `astream`） | ✅ langchain-openai | `agent/model.py` |
| 消息类型（System/Human/Tool/AIMessageChunk） | ✅ langchain-core | `agent/loop.py` |
| 工具 schema 声明（`@tool` 装饰器） | ✅ langchain-core | `agent/tools.py` |
| Agent 编排 / 状态图 / 循环控制 | ❌ 全部手写 | `loop.py` + `routers/projects.py` |

依赖清单只有 `langchain-openai`、`langchain-community`，无 `langgraph`。工具执行也没走
LangChain 的 ToolNode——`execute_tool` 手写：按名字遍历工具、`inspect.signature` 校验参数、
异常捕获后转成失败文本回喂模型（`SandboxViolation` 不中断循环）。

### 3.2 关键控制流技巧

1. **Python 异常做控制流信号**：`submit_*` 工具被执行时抛专用异常
   （如 `_SubmitTicketsInvoked`），沿生成器向上传播，由路由层捕获后携 JSON 终止循环——
   不需要图节点跳转。
2. **`final_text_hook` 收尾钩子**：循环本身不含业务判断；自洽性核验（申报改动 vs 磁盘
   真实改动）、回喂次数上限等降级链逻辑全部由路由层注入（工单 0025 用纯集合运算取代了
   早期的中文措辞检测）。
3. **多阶段编排是数据库推导的状态机，不是图**：`send_message` 按 DB 中消息的 pending
   状态分流到不同 `_*_stream` 函数，每个函数内部各跑一遍 `run_generation`（各自独立工具集）。
4. **模型无绑定（鸭子类型）**：循环只依赖 `bind_tools` / `astream` / `ainvoke` 约定，
   测试用 `tests/fake_model.py` 可编程伪模型走同一接口注入，全链路离线可测。
5. **重试语义**：模型调用失败带指数退避重试，但**一旦已外发内容就不再重试**
   （避免半截流重复），直接以 error 事件收尾。
6. **思考流拆分**：MiniMax 行内 `<think>` 标签由 `ThinkSplitter` 增量拆分（标签可能被
   切断在相邻 chunk，尾部疑似标签前缀的部分暂留缓冲区）；Qwen3 的 `reasoning_content`
   由 `ThinkingChatOpenAI` 子类补采（langchain-openai 基类会丢弃该字段）。

---

## 四、对话式迭代的可靠性防御链（ADR 0004 / 0005）

已有文件后的**用户对话轮**是唯一进意图分类器（`agent/intent.py`）的入口：

- **咨询类** → 只绑只读工具集（能力缺失代替劝说，物理上写不了文件），流式纯文本收尾，
  不产卡片、不核验、不入日志、不留快照；
- **修改类** → 工程师智能体迭代，收尾经过多层防御：

| 层 | 机制 | 说明 |
|---|---|---|
| 1 | 轮次产物终结出口 | `submit_turn_result` 是唯一合法收尾出口，模型必须显式申报本轮改动（intent / summary / changed_files） |
| 2 | 自洽性核验（工单 0025） | 比对「申报的改动集合」与「磁盘真实改动集合」，纯集合运算不看措辞；不一致回喂一次（轮级预算，防死循环） |
| 3 | 意图误判二次防线 | 分类器把明确带改动动词的诉求判成 consult 时，用确定性关键词规则救回 modify_code |
| 4 | 超步兜底 | `max_steps_exhausted` 时磁盘已有的真实改动不被没收，照常收尾建快照 |
| 5 | 零交付闸（工单 0029） | 团队工单执行轮磁盘零改动只有一条合法出口——经工单级确认回喂后的结构化 no_change；其余零改动一律按失败处置 |

---

## 五、记忆管理机制

**没有用向量库做对话记忆**（Milvus 只用于 RAG 模板检索），也没有 LangChain Memory 组件。
记忆是分层设计：短期靠对话窗口，长期靠迭代日志，事实基准靠磁盘指纹。

### 5.1 第 1 层：对话历史（短期记忆）

- 每条消息（用户话、智能体输出、工具事件、思考片段）全量落 SQLite `Message` 表，
  **持久化永不截断**，前端刷新可完整回看；
- 喂给模型的只有 `_llm_history` 组装的**最近 10 轮问答**（`agent_history_window=10`），
  且按 `kind` 过滤：
  - `event`（工具事件）/ `thinking`（思考）/ `ticket`（工单进度）/ system 引导 → 不入上下文；
  - `consensus` / `spec` / `tickets` / `prd`（阶段定案产物）→ 以带前缀的 AIMessage 入上下文；
  - `*_confirm`（确认消息，含用户追加意见）→ 以 HumanMessage 入上下文，后续阶段可见。

### 5.2 第 2 层：迭代日志（长期记忆）

`Project.iteration_log`（JSON 列），**按「改动」而非「对话轮」累积**（工单 0026）：

- 每轮成功收尾追加一条：`{seq, user_text, user_goal(分类器蒸馏), files(真实触及文件),
  verdict, scope_verdict}`；
- 每轮开始读出并渲染进系统提示（`build_system_prompt`），弥补窗口截断的失忆，
  避免重复改或漏改；
- **不截断、不聚合**（注释明确算过 token 账：净开销持平或更短，不预先优化）；
- 写入纪律：团队工单执行（`record_iteration=False`）不读也不写——内部编排不是用户对话轮，
  避免把工单指令当「用户说」污染后续提示；咨询轮无改动不入日志；超步兜底轮只要磁盘有
  真实改动也留一条（否则后续轮次无法理解磁盘现状）。

### 5.3 第 3 层：磁盘状态即事实基准（防「记忆过时」）

不信任模型的叙述性记忆，**以磁盘为权威**（工单 0022）：

- 每轮开始扫描项目目录生成 `file_summaries`：**路径 + 行数 + 内容哈希**，注入系统提示。
  哈希的作用：*若与记忆中的版本不同，说明记忆已过时，必须 read_file*——模型不许凭记忆
  写 `edit_file`；
- 同一份扫描复用为**轮前指纹**：轮末自洽性核验拿它与磁盘终态做集合比对，看磁盘不看措辞。

### 5.4 第 4 层：单轮内的临时记忆

`run_generation` 里的 `messages` 列表只在内存中活一轮（模型输出、ToolMessage 不断追加），
不持久化；断流/取消时尽力把已流出的 thinking 片段落库，供刷新后回看诊断。

### 5.5 第 5 层：跨项目记忆（RAG）

Milvus Lite 知识库（`seed:{key}` 模板种子 + `pub:{slug}` 已发布应用）是跨项目经验库，
经 `search_templates` 工具按需语义检索——服务生成质量，不服务对话连续性。

### 5.6 记忆分层总览

| 层 | 载体 | 生命周期 | 注入方式 |
|---|---|---|---|
| 对话窗口 | SQLite `Message` 表 | 全量持久化 / 只喂最近 10 轮 | langchain 消息列表 |
| 迭代日志 | `Project.iteration_log` JSON 列 | 按改动累积、永不截断 | 系统提示渲染 |
| 磁盘指纹 | 每轮重扫（路径+行数+哈希） | 每轮刷新 | 系统提示 + 轮末核验基准 |
| 轮内消息 | 内存 `messages` 列表 | 单轮 | 直接对话上下文 |
| RAG 知识库 | Milvus Lite | 跨项目长期 | `search_templates` 工具按需检索 |

---

## 六、成果出口：预览 / 发布 / 世界

```
生成落盘 → 项目快照 (snapshots.py，支持回滚 + Monaco 代码视图 / Diff，上限 50 版)
        → 预览 (/p/ 沙箱直出，serving.py 路径白名单 + CSP sandbox 头)
        → 发布 (publish.py：稳定公开链接，沉淀进 RAG 知识库)
        → 应用世界画廊 (world.py：浏览 + 一键克隆，克隆失败事务性回滚)
```

---

## 七、SSE 事件协议（前后端契约）

| 事件 | 含义 |
|---|---|
| `text` / `thinking` | 正文 / 思考过程增量（打字机效果） |
| `tool` (start/done/error) | 工具调用卡片（参数超长只留摘要，结果截 500 字符） |
| `clarify` | 结构化澄清问题（选项卡片） |
| `consensus` / `spec` / `tickets` | 阶段定案产物卡片 |
| `ticket_progress` | 团队模式逐单执行进度 |
| `done` | 本轮结束（可携 verdict / warning / artifact 字段） |
| `error` | 失败（可携结构化 reason，如 `max_steps_exhausted`） |

---

## 附：关键文件索引

| 模块 | 文件 |
|---|---|
| 应用装配 | `backend/app/main.py` |
| 路由 / 阶段分流 / 流水线 | `backend/app/routers/projects.py` |
| 智能体循环 | `backend/app/agent/loop.py` |
| 工具与沙箱 | `backend/app/agent/tools.py` |
| 系统提示拼装 | `backend/app/agent/prompts.py` |
| 意图分类 / 正确性裁判 | `backend/app/agent/intent.py`、`agent/judge.py` |
| 模型工厂 | `backend/app/agent/model.py` |
| ORM 模型（记忆载体） | `backend/app/models.py` |
| RAG 知识库 | `backend/app/rag/{store,seeds,embeddings}.py` |
| 快照 / 发布 / 世界 | `backend/app/snapshots.py`、`routers/publish.py`、`routers/world.py` |
| 前端 SSE 客户端 | `frontend/src/api/sse.ts`、`api/client.ts` |
| 架构决策记录 | `docs/adr/0001–0005`、工单规格 `docs/tracker/0001–0030` |
