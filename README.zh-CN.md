<div align="center">

# 🚀 Agent Made Easy

### 一套从零理解 AI Agent 的实践型学习资料

**从 LLM 基础，到 Agent Loop、Tool、Memory、RAG，再到可复用 Skill。**


[English README](README.en.md) | 🌏 **中文**

</div>

---

## ✨ 这是什么？

**Agent Made Easy** 是一套面向 AI Agent 初学者和工程实践者的系统化学习资料。

## 🌟 这套资料有什么不同？

- **先讲系统，不先讲玄学**  
  把 Agent 当成 Runtime 系统来拆，而不是包装成神秘聊天机器人。

- **边界讲得很清楚**  
  LLM 提出建议，Runtime 负责执行；Tool 返回 Observation；State 记录进度。

- **概念都配最小实验**  
  每个关键系统概念，都有一个能落地的最小代码实验。

- **适合小白，但不回避工程问题**  
  不讲多余数学，但会讲权限、失败、状态、记忆、检索和可复用能力。

- **中英文双版本**  
  GitHub 默认英文主页，也提供中文 README 和中文正文资料。


## 👀 适合谁看？

如果你符合下面任意一种情况，这套资料会很适合你：

- 你刚开始学 AI Agent，希望建立清晰心智模型。
- 你用过 LLM API，但还不理解 Agent 架构。
- 你不想一上来就被复杂框架绑住。
- 你分不清 Tool Calling、Context、State、Memory、RAG、Skill。
- 你希望先看最小可运行例子，再进入生产级框架。

这套资料可能不适合你，如果：

- 你只想找一个开箱即用的 Agent 框架。
- 你已经很熟悉 Agent Runtime 架构。
- 你想看的是偏研究论文或 benchmark 对比。

---

读完这套内容，你应该能判断：

- 一个系统只是普通 LLM 调用，还是已经具备 Agent 结构？
- LLM 负责判断什么，Runtime 又负责执行什么？
- Tool、Memory、RAG、Skill 到底怎么连成一个系统？
- Agent 的权限边界、失败处理和任务连续性应该放在哪里？

> 🧠 **核心判断：**  
> Agent 不是更聪明的聊天机器人。  
> Agent 是一个围绕目标运行的 **任务执行系统**。

---

## 🧭 学习路线

这套内容按从浅到深的路线设计：

1. **先看清系统形态**  
   理解 Agent 为什么不等于聊天机器人，也不等于一次 LLM 调用。

2. **再建立任务闭环**  
   掌握 Goal、Action、Observation、State、Feedback Loop 和 Stop Condition。

3. **理解模型输入控制**  
   搞清楚 Instruction、Context、Prompt、Messages 每一轮怎么组织。

4. **让 Agent 连接外部世界**  
   加入 Tool、Tool Description、Tool Call、Runtime 校验和 Observation。

5. **让任务能连续推进**  
   区分 State、Memory、Context，理解 Memory Write、Retrieval、Update 和 Forgetting。

6. **让回答基于外部资料**  
   用 RAG 做检索增强生成，让 Agent 的回答有资料依据。

7. **封装可复用能力**  
   用 Skill 把某一类任务能力组织成可选择、可加载、可管理的能力包。

---

## 📚 课程目录

### 核心章节

| # | 章节 | 你会学到什么 |
| --- | --- | --- |
| 01 | [Agent：从对话模型到任务执行系统](<outputs/1.Agent：从对话模型到任务执行系统_排版优化版.md>) | Agent 的基本定义，以及它和聊天机器人、自动化、Workflow、Copilot 的区别 |
| 02 | [LLM：Agent 的能力来源与边界](<outputs/2.LLM-Agent-capability-boundary-optimized.md>) | LLM 给 Agent 提供什么能力，以及为什么单靠 LLM 还不够 |
| 03 | [Agent Loop：Agent 如何推进任务](<outputs/3.Agent Loop：Agent 如何推进任务_排版优化版.md>) | Goal、Plan、Step、Action、Observation、State、Feedback Loop、Stop Condition 和 ReAct |
| 04 | [Model Input System：Agent 如何组织模型输入](<outputs/4.Model Input System：Agent 如何组织模型输入_排版优化版.md>) | Instruction、Prompt、Context、Messages、Context Assembly 和 Prompt Injection |
| 05 | [Tool System：Agent 如何连接外部世界](<outputs/5.Tool System：Agent 如何连接外部世界_排版优化版.md>) | Tool、Tool Description、Tool Call、Runtime、Tool Failure 和 Permission Boundary |
| 06 | [Memory System：State、Memory 与任务连续性](<outputs/6.Memory System：State、Memory 与任务连续性_排版优化版.md>) | State、Memory、Context、Memory Write、Retrieval、Update、Forgetting 和隐私风险 |
| 07 | [Knowledge System：RAG 如何让 Agent 基于外部资料工作](<outputs/7.Knowledge System：RAG 如何让 Agent 基于外部资料工作_排版优化版.md>) | RAG、Chunk、Embedding、Index、Retrieval、Rerank 和 Grounded Answer |
| 08 | [Skill System：Skill 如何复用 Agent 能力](<outputs/8.Skill System：Skill 如何复用 Agent 能力_排版优化版.md>) | Skill Registry、Progressive Loading、Skill Package 和可复用 Agent 能力 |

### 实战实验

| # | 实验 | 你会做出什么 |
| --- | --- | --- |
| E1 | [最小 Agent 闭环](<outputs/agent-made-easy_实验1_最小Agent闭环.md>) | 用最小伪代码看清 Goal、Context、Tool、Observation、State 和 Feedback Loop |
| E2 | [最小 ReAct Agent](<outputs/agent-made-easy_实验2_最小ReActAgent.md>) | 只用 Python 标准库跑通一个 ReAct 风格 Agent |
| E3 | [模型输入系统](<outputs/agent-made-easy_实验3_模型输入系统.md>) | 在每一轮模型判断前组装 Context 和 Messages |
| E4 | [最小 Tool Calling Agent](<outputs/agent-made-easy_实验4_最小ToolCallingAgent.md>) | 实现 Tool Registry、Tool Description、Tool Call、Runtime 校验和权限边界 |
| E5 | [API 调用与 Memory Agent](<outputs/agent-made-easy_实验5_API调用与MemoryAgent.md>) | 接入 Responses API、Function Calling、本地 Tool、State、Memory 和 Context |
| E6 | [最小 SkillAgent](<outputs/agent-made-easy_实验6_最小SkillAgent.md>) | 实现 Skill Registry、Skill Selection、Progressive Loading 和 Skill 约束输出 |

---

## 🧩 核心心智模型

很多 Agent 学习中的混乱，来自把所有东西都归结成“模型”。

这套资料会一直强调边界：

| 层级 | 负责什么 |
| --- | --- |
| 🧠 **LLM** | 理解、生成、总结、提出建议 |
| 🧾 **模型输入系统** | 决定每一轮 LLM 看见什么 |
| 🛠️ **Tool System** | 安全地连接外部能力 |
| 🧭 **Runtime** | 解析、校验、执行、拒绝、记录、停止 |
| 🗂️ **State** | 记录当前任务进度 |
| 🧠 **Memory** | 保存和取回可复用信息 |
| 📚 **Knowledge / RAG** | 检索外部资料，让回答有依据 |
| 🧩 **Skill** | 封装可复用 Agent 能力 |

```text
User Goal
   ↓
Context Assembly
   ↓
LLM Decision
   ↓
Action / Tool Call
   ↓
Runtime Validation
   ↓
Tool Execution
   ↓
Observation
   ↓
State Update
   ↓
Next Round or Stop
```

---

## 🧪 为什么要配实验？

实验不是为了堆代码，而是为了让你看清系统边界。

这套实验会从 fake LLM、fake Tool 开始，再逐步替换成真实系统组件：

- 实验 1：看见最小闭环。
- 实验 2：跑通最小 ReAct Agent。
- 实验 3：加入模型输入组装。
- 实验 4：加入受控 Tool Calling。
- 实验 5：接入真实 API 调用和 Memory。
- 实验 6：把能力封装成 Skill。

> 🔥 **重点不是背代码。**  
> 重点是知道 Agent 系统里每个模块到底负责什么。

---


## 🗺️ 仓库结构

```text
.
├── README.md                         # 英文主页
├── README.zh-CN.md                   # 中文主页
└── chapter/
    ├── chapter1/  
    │   ├── 中文版
    │   ├── English
    │   
    └──  chapter2/...                         # 中文章节和实验 Markdown
```

---

## 🚀 推荐阅读顺序

如果你完全是新手：

```text
第 1 章 -> 第 2 章 -> 实验 1 -> 第 3 章 -> 实验 2
```

如果你已经懂一点 LLM：

```text
第 3 章 -> 第 4 章 -> 实验 3 -> 第 5 章 -> 实验 4
```

如果你更关注工程化 Agent：

```text
第 5 章 -> 第 6 章 -> 第 7 章 -> 第 8 章 -> 实验 5-6
```

---

## 🙌 贡献方向

目前还在支持更新中。后续会有MCP和实际项目试炼！
欢迎补充：
- 更清晰的解释。
- 更贴近生产的工程版本。
- 实验代码测试。

---

## ⭐ 支持一下

如果这套资料帮你更清楚地理解 Agent，欢迎给仓库点个 Star。

这会让更多人找到一条不玄学、能落地的 Agent 学习路线。

---

