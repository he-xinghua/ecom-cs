# 电商智能客服系统 — 第一章：纯对话链路

- 日期：2026-09-15
- 状态：设计已定稿，待实施
- 仓库：`D:\Hxh\Coding\ecom-cs`

## 1. 背景与目标

为电商业务搭建智能客服系统。第一章只跑通**纯对话**：不做工具调用、不做 Agent 循环，把「模型接入 → 多轮会话 → 流式输出 → 结构化抽取」这条最基础链路做扎实，为后续章节（工具调用、Agent、RAG、可观测）留出干净的接口契约。

第一章交付的是**一组 HTTP API**，不是产品界面。

## 2. 范围

### 2.1 本章做

1. SSE 流式对话接口，逐 token 推送
2. Prompt 模板化管理，System Prompt 写清客服角色设定与行为约束
3. 结构化抽取独立端点，用 `with_structured_output` 把售后描述提取为固定字段
4. 多轮上下文最简版：内存会话表 + 历史消息裁剪 + token 预算控制

### 2.2 本章不做

- 工具调用、Agent 循环（明确排除）
- 聊天前端页面（Vibe Coding 方式做，**延后到后续章节**）
- 会话持久化（SQLite/Redis）、多 worker 共享会话
- 鉴权、限流、计费、可观测性（Langfuse 等）

### 2.3 技术选型（已定死，不自作主张更换）

- Python + FastAPI + LangChain
- 模型接入直连上游，**应用侧统一说 OpenAI 协议**
- 地址、模型名、密钥全部在 `.env`
- GPT / Claude / DeepSeek / Ollama 可换着接

## 3. 关键设计决策

### 3.1 决策一：会话状态归服务端，存内存

**选择**：服务端维护 `{session_id: [messages]}` 内存字典。请求只传 `session_id` + 本轮新消息。

**理由**：真实客服系统中会话属于服务端资产；curl 验收最干净（两轮只换问题文本，不用手写全量历史 JSON）。

**已知代价**（显式接受，不修）：进程重启历史全丢；多 worker 之间不共享。这两条都属于「持久化」范畴，是本章明确的非目标。

### 3.2 决策二：结构化抽取做独立端点，不做对话流旁路

**选择**：`POST /extract`，传入一段文本，返回结构化 JSON。

**理由**：职责单一、验收直接、可单独跑评估集。与对话流解耦，不会拖慢 SSE 首字延迟，也不会把「本章不做 Agent」的边界搅浑。

**被否决的备选**：对话流内每轮自动抽取（成本翻倍、延迟增加、调试面变大）。

### 3.3 决策三：结构化输出 method 按 provider 配置化

这是本章最重要的一个技术决策，起因是一个**真实矛盾**。

**矛盾**：需求同时要求「统一 OpenAI 协议接入四家模型」和「用 `with_structured_output` 做结构化输出」，但没有任何单一 method 能在四家上都有 schema 保证：

| Provider | `json_schema`（严格） | `json_mode` | `function_calling` |
|---|---|---|---|
| OpenAI GPT | 原生严格支持 | 支持 | 支持 |
| DeepSeek | **不支持** | 只保证合法 JSON，**不保证 schema** | 支持；`strict` 仍在 beta，需把 `base_url` 换到 `/beta` |
| Claude（Anthropic 的 OpenAI 兼容层） | **`response_format` 被静默忽略** | **同上被忽略** | 可调用，但 `strict` 被忽略，**不保证 schema** |
| Ollama | OpenAI 兼容端点声称支持，但社区多处报告被忽略 / 属性乱序 / **静默失败**；原生机制是 `format` 参数 | 同样口径不一 | 取决于具体模型 |

**危险点是「静默」而非「报错」**：不在 GPT 上运行不会拿到任何错误，只会拿到缺字段或字段错的 JSON。

**选择**：保留统一 OpenAI 协议不动，加一层薄的能力表。

- `.env` 提供 `STRUCTURED_OUTPUT_METHOD`，显式配置优先级最高
- 未显式配置时，按 `base_url` 关键字匹配 provider 给默认值：OpenAI→`json_schema`，Claude / DeepSeek / Ollama→`function_calling`
- 无论走哪个 method，外层一律套 **Pydantic 校验 + 失败重试一次**

**理由**：这没有更换既定选型，而是给固定选型加适配层。把「静默降级」变成「可配置的显式降级 + 应用侧兜底」。

**依据来源**：
- Anthropic 官方兼容文档将 `response_format` 列入「被忽略参数」，并声明该兼容层「主要面向测试和模型能力对比，不是长期或生产方案」
- LangChain 官方在 `ChatOpenAI` + 自定义 `base_url` 处警告：只保证官方 OpenAI 规范，路由器/代理的特有字段可能不被保留
- Ollama 的 OpenAI 兼容端点有记录的 issue：`json_schema` 被静默忽略、属性顺序错乱，pydantic-ai 因此把 schema 校验整个跑空

**待实测确认项（见 §7.1）**：`ChatOpenAI.with_structured_output` 在 LangChain 1.4 的**默认 method 究竟是什么**，不凭记忆断言，实现阶段第一步用 introspect 实测。

## 4. 目录结构

```
ecom-cs/
├─ .env                     # 真实配置，gitignore
├─ .env.example             # 模板，入库
├─ .gitignore
├─ requirements.txt
├─ README.md                # 演示命令
├─ app/
│  ├─ __init__.py
│  ├─ main.py               # FastAPI 实例 + lifespan + 路由挂载
│  ├─ config.py             # pydantic-settings 读 .env + provider 能力表
│  ├─ llm.py                # 模型工厂：全应用唯一构造模型的地方
│  ├─ schemas.py            # Pydantic 请求/响应契约
│  ├─ session.py            # 内存会话表 + 裁剪逻辑（纯函数）
│  ├─ prompts/
│  │  ├─ __init__.py
│  │  └─ cs.py              # ChatPromptTemplate + 客服 System Prompt
│  └─ routers/
│     ├─ __init__.py
│     ├─ chat.py            # POST /chat/stream
│     └─ extract.py         # POST /extract
├─ evals/
│  ├─ cases_extract.jsonl       # 抽取评估集（标注样例）
│  ├─ run_extract_eval.py       # 跑抽取评估集并出准确率
│  ├─ cases_prompt.jsonl        # 客服行为约束评估集
│  └─ run_prompt_eval.py        # 跑行为评估集并出判定
├─ tests/                   # 纯函数/契约的 TDD 测试
└─ dev-notes/
   └─ ch01.md               # 开发过程留痕
```

## 5. 组件设计

### 5.1 配置层（`config.py`）

用 `pydantic-settings` 读 `.env`：

| 变量 | 说明 | 默认 |
|---|---|---|
| `OPENAI_BASE_URL` | 上游地址 | 必填 |
| `OPENAI_API_KEY` | 密钥 | 必填 |
| `MODEL_NAME` | 模型名 | 必填 |
| `STRUCTURED_OUTPUT_METHOD` | 覆盖能力表默认值 | 空（走能力表） |
| `MAX_HISTORY_TOKENS` | 历史 token 预算 | `2000` |
| `REQUEST_TIMEOUT` | 上游超时（秒） | `60` |

`get_provider_capability(base_url)` 做 base_url 关键字匹配，返回该 provider 的默认 method。

### 5.2 模型工厂（`llm.py`）

全应用**唯一**构造模型的地方，避免 base_url/api_key 散落各处：

```python
def get_chat_model() -> ChatOpenAI:
    return ChatOpenAI(
        base_url=settings.openai_base_url,
        api_key=settings.openai_api_key,
        model=settings.model_name,
        timeout=settings.request_timeout,
    )
```

切换 provider 只改 `.env` 三行，代码零改动。

### 5.3 会话管理（`session.py`）

- `SessionStore`：`{session_id: list[BaseMessage]}` 内存字典，配 `asyncio.Lock` 防并发写坏
- `get_history(session_id) -> list[BaseMessage]`
- `append_turn(session_id, user_text, assistant_text)`：**流式结束后**才写回
- `build_context(history, system_prompt) -> list[BaseMessage]`：**纯函数**，走 TDD

裁剪用 `langchain_core.messages.trim_messages`，签名已核实：

```python
trim_messages(
    messages,
    max_tokens=settings.max_history_tokens,
    token_counter="approximate",
    strategy="last",
    start_on="human",
    include_system=True,
    allow_partial=False,
)
```

两个**实测得出的关键点**（与直觉相反，容易踩）：

1. **`include_system` 默认是 `False`** —— 不显式传 `True`，System Message 会被裁掉，客服人设直接消失
2. **`token_counter` 用字面量 `"approximate"`** —— 传模型对象会走 tiktoken，而 tiktoken 对 DeepSeek / Claude / Ollama 的分词不适用，预算控制会失真。用 `"approximate"` 在四家上口径统一

`start_on="human"` 保证裁剪后首条是 HumanMessage：多数上游期望历史以 HumanMessage 开头，或 SystemMessage 后紧跟 HumanMessage，否则可能因 role 顺序报 400。

### 5.4 Prompt 管理（`prompts/cs.py`）

```python
ChatPromptTemplate.from_messages([
    ("system", CS_SYSTEM_PROMPT),
    MessagesPlaceholder("history"),
    ("human", "{input}"),
])
```

`CS_SYSTEM_PROMPT` 写死四类行为约束：

1. **不编造订单信息** —— 没有订单上下文时，不臆造订单号、物流状态、金额
2. **不擅自承诺赔付额度** —— 涉及退款/赔偿金额一律走流程话术，不给具体数字承诺
3. **超权限转人工** —— 明确列出转人工触发条件，不硬答
4. **语气与话术边界** —— 礼貌、简洁、不越界承诺、不人身评价

### 5.5 对话接口（`routers/chat.py`）

`POST /chat/stream`，请求体 `{session_id: str, message: str}`，响应 `text/event-stream`。

流程：
1. 取会话历史 → `build_context()` 裁剪并拼 System Prompt
2. `model.astream(messages)` 逐 chunk 推送
3. 流正常结束后推 `done` 事件，并把本轮 user + assistant 写回会话表
4. 上游异常推 `error` 事件，且**不写入本轮**（避免半截回复污染历史）

SSE 事件格式：

```
event: token
data: {"delta": "您好"}

event: done
data: {"session_id": "s1", "finish_reason": "stop"}

event: error
data: {"message": "上游返回 401"}
```

用 FastAPI `StreamingResponse(media_type="text/event-stream")`，配 `X-Accel-Buffering: no` 头避免反向代理缓冲。逐 chunk `yield`，不攒批。

### 5.6 结构化抽取（`routers/extract.py`）

`POST /extract`，请求体 `{text: str}`，响应 `AfterSalesTicket`：

```python
class IssueType(str, Enum):
    REFUND = "退款"
    EXCHANGE = "换货"
    REPAIR = "维修"
    LOGISTICS = "物流"
    INVOICE = "发票"
    OTHER = "其他"

class AfterSalesTicket(BaseModel):
    order_id: str | None          # 提取不到必须是 None，禁止编造
    issue_type: IssueType
    expected_resolution: str | None
    confidence: float             # 0-1，模型自报置信度
```

`confidence` 是**模型自报**值，已知不可靠。它在第一章的唯一用途是给评估集提供一个观察维度（看模型在哪些样例上「错得很自信」），**不参与任何业务判断、不做阈值分流**。若后续章节想用它做转人工阈值，必须先用评估集校准，不能默认它可信。

调用 `model.with_structured_output(AfterSalesTicket, method=<能力表>, include_raw=True)`，拿 `parsing_error` 做判断，失败重试一次。

## 6. 验证策略

按项目约定：**产出不是可单测代码的任务（纯 Prompt、数据类），用标注样例或评估集跑一遍替代 TDD 那一步；其余步骤照走。**

| 产出 | 验证方式 |
|---|---|
| `build_context` 裁剪逻辑 | **TDD**，纯函数，先写测试 |
| Pydantic 契约（`AfterSalesTicket` 等） | **TDD**，校验规则可单测 |
| SSE 事件格式（`token` / `done` / `error`） | **TDD**，格式是契约 |
| `CS_SYSTEM_PROMPT` 行为约束 | 评估集 `cases_prompt.jsonl`，规则判定 |
| `/extract` 抽取质量 | 评估集 `cases_extract.jsonl`，字段级准确率 |

### 6.1 抽取评估集（`evals/cases_extract.jsonl`）

约 20 条**真实口吻**的售后描述，覆盖边界：错别字、口语化、一条含多诉求、完全没提订单号、夹带无关闲聊、情绪化表达。每条标注期望的 `order_id` / `issue_type` / `expected_resolution`。

跑出的指标：`order_id` 准确率、`issue_type` 准确率、`expected_resolution` 命中率。

三个指标各自的判定口径（先定义再跑，避免事后凑标准）：

- **`order_id` 准确率**：字符串精确匹配（去空格、统一大小写）。输入未提及订单号时期望值为 `null`。
  **硬约束**：输入未提及但输出非 `null` = **编造，直接判错**，不得计入「提取不到算对」的口径。
- **`issue_type` 准确率**：与标注枚举值严格相等。标 `OTHER` 的样例若模型猜了具体类型，算错（宁缺毋滥，乱猜不算对）。
- **`expected_resolution` 命中率**：不做字符串匹配（措辞自由度太高）。改为**语义等价判定**，由评分模型按「是否表达了同一诉求」二值判定，判定 prompt 与评分用的模型一并记录进 dev-notes，保证可复现。

评估脚本输出逐条明细（输入 / 期望 / 实际 / 判定），不只输出汇总数字。汇总数字单看无法定位退化。

### 6.2 行为评估集（`evals/cases_prompt.jsonl`）

约 8 条行为样例，针对 System Prompt 的四条约束各设正反例，例如「追问一个不存在的订单号」「要求超出额度的赔付」「诱导模型承诺具体到账时间」「超范围问题」。

每条标注期望行为与禁止行为，规则判定通过/不通过。

### 6.3 评估结论的呈现

跑完**必须出准确率数字**，不写「效果不错」这类无信息量的结论。数字连同当次使用的 provider/model 记入 `dev-notes/ch01.md`。

## 7. 风险与待确认项

### 7.1 必须在实现前实测的（不凭记忆）

1. `ChatOpenAI.with_structured_output` 在 LangChain 1.4 的**默认 method** —— 用 introspect 打印实测值，结论写入 dev-notes，再决定能力表怎么填
2. Anthropic OpenAI 兼容层对 `function_calling` 的**实际行为**是否可用（文档说 `strict` 被忽略，但基础调用可用）
3. Ollama OpenAI 兼容端点对 `response_format` 的实际处理（社区报告与官方声明不一致）

### 7.2 环境风险

- Python 3.14.6，全局环境干净。**依赖装在项目内 `.venv`**，不污染全局
- pip 联网频繁出现 `SSL: UNEXPECTED_EOF_WHILE_READING` 重试。安装失败需按镜像重试，此步骤写进实施计划
- 已实测可解析的版本：fastapi 0.141.1 / langchain 1.4.0 / langchain-openai 1.6.2 / langchain-core 1.6.3

### 7.3 已知未决

- 「对话流内自动抽取」被本章排除，但真实客服工单流最终需要它，留在后续章节
- 会话持久化、多 worker 共享，本章显式不做

## 8. 验收标准

对应需求方给出的三条，逐条给可执行命令：

**验收 1 — 流式回复**
```
curl -N -X POST http://127.0.0.1:8000/chat/stream \
  -H "Content-Type: application/json" \
  -d '{"session_id":"demo","message":"你们什么时候发货?"}'
```
预期：逐 token 打印 `event: token`，最后 `event: done`。

**验收 2 — 两轮上下文**
```
curl -N -X POST http://127.0.0.1:8000/chat/stream -H "Content-Type: application/json" \
  -d '{"session_id":"demo","message":"我买的是机械键盘"}'
curl -N -X POST http://127.0.0.1:8000/chat/stream -H "Content-Type: application/json" \
  -d '{"session_id":"demo","message":"我刚才说我买的是什么?"}'
```
预期：第二轮答出「机械键盘」。

**验收 3 — 结构化 JSON**
```
curl -X POST http://127.0.0.1:8000/extract -H "Content-Type: application/json" \
  -d '{"text":"我上周买的那个键盘 A12345 到现在还没发货,要么赶紧发要么退款"}'
```
预期：返回含 `order_id` / `issue_type` / `expected_resolution` / `confidence` 的 JSON。

## 9. 交付物

1. 可运行的 FastAPI 服务与上述三条演示命令（写入 README）
2. 测试结果（TDD 部分）与评估集准确率数字
3. `dev-notes/ch01.md` —— 开发过程留痕
