<div align="center">

# 🚀 Agent Made Easy

### A practical learning guide for understanding AI Agents from zero

**From LLM fundamentals to Agent Loop, Tool, Memory, RAG, and reusable Skills.**


🌏 **English** | [Chinese README](README.zh-CN.md)

</div>

---

## ✨ What Is This?

**Agent Made Easy** is a structured learning resource for beginners and builders who want to understand AI Agents clearly.

## 🌟 What Makes This Different?

- **System first, hype later**  
  It explains Agent as a runtime system instead of wrapping it in vague magic.

- **Clear boundaries**  
  The LLM proposes. The Runtime executes. Tools return Observations. State records progress.

- **Every key concept has a minimal lab**  
  Each important system idea comes with a small implementation you can reason about.

- **Beginner-friendly, but still engineering-aware**  
  It avoids unnecessary math, but it does talk about permissions, failures, state, memory, retrieval, and reusable capability.

- **Bilingual by design**  
  The GitHub homepage can use English by default, while the Chinese README and Chinese learning materials remain available.

## 👀 Who This Is For

This guide is a good fit if:

- You are just starting to learn AI Agents and want a clear mental model.
- You have used LLM APIs but do not yet understand Agent architecture.
- You do not want to be locked into a complex framework from day one.
- You are confused by Tool Calling, Context, State, Memory, RAG, or Skill.
- You want minimal runnable examples before moving into production frameworks.

This guide may not be the best fit if:

- You only want a plug-and-play Agent framework.
- You already understand Agent Runtime architecture deeply.
- You are mainly looking for research papers or benchmark comparisons.

---

After reading this guide, you should be able to judge:

- Is a system just a normal LLM call, or does it really have an Agent structure?
- What should the LLM decide, and what should the Runtime execute?
- How do Tool, Memory, RAG, and Skill connect into one system?
- Where should permission boundaries, failure handling, and task continuity live?

> 🧠 **Core judgment:**  
> An Agent is not just a smarter chatbot.  
> An Agent is a **goal-driven task execution system**.

---

## 🧭 Learning Path

This material is organized from basic concepts to practical system design:

1. **First, understand the system shape**  
   Learn why an Agent is not just a chatbot and not just a single LLM call.

2. **Then build the task loop**  
   Understand Goal, Action, Observation, State, Feedback Loop, and Stop Condition.

3. **Understand model input control**  
   Learn how Instruction, Context, Prompt, and Messages are assembled each round.

4. **Let the Agent connect to the outside world**  
   Add Tool, Tool Description, Tool Call, Runtime validation, and Observation.

5. **Let tasks continue across rounds**  
   Separate State, Memory, and Context; understand Memory Write, Retrieval, Update, and Forgetting.

6. **Ground answers in external material**  
   Use RAG to retrieve knowledge and make Agent answers evidence-based.

7. **Package reusable capabilities**  
   Use Skill to organize task-specific capability into selectable, loadable, and manageable packages.

---

## 📚 Curriculum

### Core Chapters

| # | Chapter | What You Will Learn |
| --- | --- | --- |
| 01 | [Agent: From Dialogue Model to Task Execution System](<outputs/1.Agent：从对话模型到任务执行系统_排版优化版.md>) | The basic definition of Agent, and how it differs from chatbots, automation, Workflow, and Copilot |
| 02 | [LLM: The Capability Source and Boundary of Agents](<outputs/2.LLM-Agent-capability-boundary-optimized.md>) | What LLMs provide to Agents, and why LLM alone is not enough |
| 03 | [Agent Loop: How Agents Move Tasks Forward](<outputs/3.Agent Loop：Agent 如何推进任务_排版优化版.md>) | Goal, Plan, Step, Action, Observation, State, Feedback Loop, Stop Condition, and ReAct |
| 04 | [Model Input System: How Agents Organize Model Input](<outputs/4.Model Input System：Agent 如何组织模型输入_排版优化版.md>) | Instruction, Prompt, Context, Messages, Context Assembly, and Prompt Injection |
| 05 | [Tool System: How Agents Connect to the Outside World](<outputs/5.Tool System：Agent 如何连接外部世界_排版优化版.md>) | Tool, Tool Description, Tool Call, Runtime, Tool Failure, and Permission Boundary |
| 06 | [Memory System: State, Memory, and Task Continuity](<outputs/6.Memory System：State、Memory 与任务连续性_排版优化版.md>) | State, Memory, Context, Memory Write, Retrieval, Update, Forgetting, and privacy risks |
| 07 | [Knowledge System: How RAG Lets Agents Work with External Materials](<outputs/7.Knowledge System：RAG 如何让 Agent 基于外部资料工作_排版优化版.md>) | RAG, Chunk, Embedding, Index, Retrieval, Rerank, and Grounded Answer |
| 08 | [Skill System: How Skills Reuse Agent Capabilities](<outputs/8.Skill System：Skill 如何复用 Agent 能力_排版优化版.md>) | Skill Registry, Progressive Loading, Skill Package, and reusable Agent capability |

### Hands-On Labs

| # | Lab | What You Will Build |
| --- | --- | --- |
| E1 | [Minimal Agent Loop](<outputs/agent-made-easy_实验1_最小Agent闭环.md>) | Use minimal pseudocode to understand Goal, Context, Tool, Observation, State, and Feedback Loop |
| E2 | [Minimal ReAct Agent](<outputs/agent-made-easy_实验2_最小ReActAgent.md>) | Run a ReAct-style Agent using only the Python standard library |
| E3 | [Model Input System](<outputs/agent-made-easy_实验3_模型输入系统.md>) | Assemble Context and Messages before each model decision |
| E4 | [Minimal Tool Calling Agent](<outputs/agent-made-easy_实验4_最小ToolCallingAgent.md>) | Implement Tool Registry, Tool Description, Tool Call, Runtime validation, and permission boundary |
| E5 | [API Calls and Memory Agent](<outputs/agent-made-easy_实验5_API调用与MemoryAgent.md>) | Connect Responses API, Function Calling, local Tools, State, Memory, and Context |
| E6 | [Minimal SkillAgent](<outputs/agent-made-easy_实验6_最小SkillAgent.md>) | Implement Skill Registry, Skill Selection, Progressive Loading, and Skill-constrained output |

---

## 🧩 Core Mental Model

Many confusions in Agent learning come from treating everything as "the model."

This guide keeps the boundaries explicit:

| Layer | Responsibility |
| --- | --- |
| 🧠 **LLM** | Understand, generate, summarize, and suggest |
| 🧾 **Model Input System** | Decide what the LLM sees each round |
| 🛠️ **Tool System** | Safely connect external capabilities |
| 🧭 **Runtime** | Parse, validate, execute, reject, log, and stop |
| 🗂️ **State** | Record current task progress |
| 🧠 **Memory** | Store and retrieve reusable information |
| 📚 **Knowledge / RAG** | Retrieve external material and ground answers |
| 🧩 **Skill** | Package reusable Agent capability |

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

## 🧪 Why Include Labs?

The labs are not here to add code for its own sake. They are here to make the system boundaries visible.

The labs start with a fake LLM and fake Tool, then gradually replace them with real system components:

- Lab 1: See the minimal loop.
- Lab 2: Run a minimal ReAct Agent.
- Lab 3: Add model input assembly.
- Lab 4: Add controlled Tool Calling.
- Lab 5: Add real API calls and Memory.
- Lab 6: Package capability as a Skill.

> 🔥 **The point is not to memorize code.**  
> The point is to know what each module in an Agent system is responsible for.

---

## 🗺️ Repository Structure

```text
.
├── README.md                         # Existing project README
├── README.en.md                      # English Agent Made Easy homepage
├── README.zh-CN.md                   # Chinese Agent Made Easy homepage
└── chapter/
    ├── chapter1/
    │   ├── Chinese version
    │   ├── English version
    │
    └── chapter2/...                  # Chinese and English chapters / labs
```

---

## 🚀 Suggested Reading Order

If you are completely new:

```text
Chapter 1 -> Chapter 2 -> Lab 1 -> Chapter 3 -> Lab 2
```

If you already understand some LLM basics:

```text
Chapter 3 -> Chapter 4 -> Lab 3 -> Chapter 5 -> Lab 4
```

If you care more about engineering-oriented Agents:

```text
Chapter 5 -> Chapter 6 -> Chapter 7 -> Chapter 8 -> Labs 5-6
```

---

## 🙌 Contribution Direction

This project is still being updated. More MCP content and real project trials will be added later.

Contributions are welcome:

- Clearer explanations.
- More production-oriented engineering versions.
- Tests for lab code.

---

## ⭐ Support

If this guide helps you understand Agents more clearly, consider giving the repository a Star.

It helps more people find a grounded, practical path into Agent learning.
