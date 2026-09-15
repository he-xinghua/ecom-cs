# ecom-cs — 电商智能客服系统

Python + FastAPI + LangChain。第一章：纯对话链路（不做工具调用 / Agent 循环）。

## 硬规则（用户明确定死，不得自行更改）

1. **全程走 Superpowers 流程**，技能自动触发。产出不是可单测代码的任务（纯 Prompt、数据类），**把 TDD 那步换成拿标注样例或评估集跑一遍验证**，其余步骤照走。
   - 例外：聊天页面用 Vibe Coding 直接做，不套 brainstorm / TDD / code review。
2. **过程留痕**：每个阶段（brainstorm 定稿、计划评审通过、每个任务完成、code review 结论、finish）完成后**立即**追记 `dev-notes/chNN.md`，记四样：用户关键原话、关键产出、被拒绝/纠偏的内容、翻车与返工。**不许收尾时一次性补记。**
3. **涉及具体库/框架/API 的用法，一律先用 Context7 MCP 查最新官方文档和接口定义再动手**（FastAPI、SQLAlchemy、LangChain、LangGraph、Milvus、Langfuse 等）。版本对不上的 API 是返工重灾区，禁止凭记忆写。
4. **技术选型定死，发现矛盾或走不通时停下来问用户，不要自行换方案。**
5. 完结交付要含：功能演示命令、测试结果、dev-notes 路径。

## 执行模式

- 实施阶段用 `superpowers:subagent-driven-development`（SDD）：每任务派独立子代理实现 + 任务级评审 + 最终整分支终审。**控制器不写业务代码**（会绕过评审）。
- **冲突优先级**：用户的指令 > 技能默认。SDD 默认的 "Rulings, not stalls"（不停下来问、自行裁决）**被用户明确否决**——遇到矛盾或走不通就停下问用户。

## 关键技术决策

- 模型接入：应用侧**统一说 OpenAI 协议**，`ChatOpenAI` + `base_url`，切换 provider 只改 `.env`（GPT / Claude / DeepSeek / Ollama）
- 会话状态：服务端内存表（接受重启丢失、多 worker 不共享）
- 结构化抽取：**独立端点** `POST /extract`，不做对话流旁路
- 结构化输出 method **按 provider 配置化**：`.env` 的 `STRUCTURED_OUTPUT_METHOD` 优先，否则按 `base_url` 匹配能力表给默认值

## 已核实的 API 事实（防止返工，来源为 Context7 官方文档）

- **LangChain 版本是 1.x**（实测 langchain 1.4.0 / langchain-core 1.6.3）。0.x 的 `ConversationBufferMemory` 等已不在主线。写法用 `init_chat_model` / `create_agent` / middleware 体系。
- **`trim_messages` 的 `include_system` 默认是 `False`** —— 不显式传 `True`，System Prompt 会被**静默**裁掉，不报错。
- **`trim_messages` 的 `token_counter` 用字面量 `"approximate"`** —— 传模型对象会走 tiktoken，而 tiktoken 对 DeepSeek / Claude / Ollama 的分词不适用，预算控制会失真。
- **跨 provider 的结构化输出是静默失败重灾区**：Claude 的 OpenAI 兼容层**静默忽略 `response_format`**；DeepSeek 不支持 `json_schema`；Ollama 的 OpenAI 兼容端点对 `response_format` 处理与官方声明不一致（有记录的 issue）。**风险在于不报错、只给错 JSON。**

## 待实测确认（不凭记忆断言，实施前必须验证）

1. `ChatOpenAI.with_structured_output` 在 LangChain 1.4 的**默认 `method`** —— 用 introspect 打印实测值
2. Anthropic OpenAI 兼容层对 `function_calling` 的实际可用性
3. Ollama OpenAI 兼容端点对 `response_format` 的实际处理

## 环境

- Python **3.14.6**（`C:\Users\Hxh\AppData\Local\Python\pythoncore-3.14-64`）。依赖装项目内 `.venv`，不污染全局。
- 已实测可解析：fastapi 0.141.1 / langchain 1.4.0 / langchain-openai 1.6.2 / langchain-core 1.6.3
- **pip 联网频繁 `SSL: UNEXPECTED_EOF_WHILE_READING` 重试**，装依赖需按镜像重试
- 远程：`git@github.com:he-xinghua/ecom-cs.git`，默认分支 `master`

## 关键路径

- Spec（binding authority）：`docs/superpowers/specs/2026-09-15-ecom-cs-ch01-design.md`
- 过程留痕：`dev-notes/ch01.md`
- 计划（待生成）：`docs/superpowers/plans/`
