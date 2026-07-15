# 第 8 章章末实践

**实验 6：把 Agent 能力封装成一个最小 SkillAgent**

*承接实验 5 的 API 与 Memory，加入 Skill Registry、Progressive Loading 与 Skill Package*

实验 5 已经把真实 API 调用、Memory Store、Memory Retrieval、Context Assembly 和本地 Tool 执行放进了一个最小 Agent。第 8 章继续往前走：当 Agent 能力开始变多时，不应该把所有规则、工具、知识来源和输出格式都塞进同一个 Prompt，而应该把某一类任务封装成 Skill。

这个实验的目标，是把第 8 章的 Skill System 做成一个可以运行的最小版本。正文里即使不放完整 Skill 示例，也不影响理解；完整的例子放在本实验里更合适，因为这里可以同时展示 Skill Registry、Skill Selection、Progressive Loading、Skill Package、Runtime、Tool、Memory 和输出约束如何连在一起。

> **阅读提示**
>
> 本实验不是做一个生产级 Skill 平台，而是跑通一个最小 SkillAgent。重点不是代码多复杂，而是让读者看到：Skill 不是高级 Prompt，而是一个可以被注册、选择、加载、执行和测试的能力包。
>
> 如果代码第一次看起来比较长，先不要急着逐行记住。先抓主线：Runtime 先看 Skill Registry 的 metadata，再选择 Skill，最后只加载被选中的 Skill Package。

## 1. 为什么第八章需要一个实战

Skill 如果只停留在概念层，很容易被读者理解成“高级 Prompt”。原因很直接：Skill 里面确实包含 Instruction、Procedure、Output Schema 这类文字说明，而且它们最终可能进入 Context，被 LLM 看见。表面看，它像是一个更长、更规范的 Prompt。

但从工程上看，Skill 的重点不是把 Prompt 写长，而是把一类任务的能力做成可复用单元。一个 Skill 需要有触发条件、权限边界、允许使用的 Tool、依赖的 Knowledge Source、输出结构、失败处理和测试评估。这个实验就是为了把这些东西放进代码，让读者看到 Skill 到底多了哪一层。

| **实验 5 已经有的能力** | **实验 6 新增的能力** |
| --- | --- |
| API 调用 LLM | 仍然使用 API，但不把所有任务规则直接塞给 LLM |
| Memory Store 与 Memory Retrieval | Memory 仍然存在，但会根据选中的 Skill 取回相关记忆 |
| 本地 Tool 执行 | Tool 不再默认全部可用，而是由 Skill 的 allowed\_tools 限定 |
| Context Assembly | Context Assembly 会加入选中 Skill 的 Instruction 与 Output Schema |
| Trace | Trace 会记录 Skill Selection 和 Skill Package 加载过程 |

## 2. 本实验要跑通的完整链路

本实验的链路比实验 5 多了一个 Skill 层。它不是替代 Runtime，也不是替代 Tool、Memory 或 Context Assembly，而是在这些系统之上增加一个能力组织方式。

- **Skill Registry：** Skill 注册表。这里只保存轻量 metadata，例如 id、name、description、triggers、version 和 permissions。

- **Skill Selection：** Skill 选择过程。Runtime 根据 Goal 和 Registry metadata 判断应该使用哪个 Skill。

- **Progressive Loading：** 渐进加载。系统先加载 Registry metadata，选中 Skill 后才加载完整 Skill Package。

- **Skill Package：** 完整能力包。包含 instruction、procedure\_topics、allowed\_tools、knowledge\_sources、output\_schema 和 failure\_handling。

- **SkillAgent：** 本实验里的最小 Agent。它不是新模型，而是 Runtime 加载 Skill 后形成的运行链路。

**图式 8P-1：本实验的最小链路**

```text
User Input
    -> Extract Goal
    -> Load Skill Registry metadata
    -> Skill Selection
    -> Load selected Skill Package
    -> Retrieve Memory by selected Skill
    -> Execute allowed Tools
    -> Assemble Final Context
    -> LLM generates answer under Skill constraints
```

> **插图 8P-1：最小 SkillAgent 的运行链路**
>
> 图中要表达什么：展示实验 6 如何在实验 5 的 API、Memory、Tool 基础上增加 Skill Registry、Skill Selection 和 Progressive Loading。
>
> 图中包含哪些元素：左侧是 User Input 和 Goal；中间是 Skill Registry metadata、Skill Selection、Selected Skill Package；下方连接 Memory Store 与 Tool Registry；右侧是 Context Assembly、LLM 和 Final Answer；Trace 记录整个过程。
>
> 生图提示：一张简洁专业的 Agent 实战流程图，浅色背景。从 User Input 到 Goal，再到 Skill Registry metadata 和 Skill Selection，选中 Skill 后加载 Selected Skill Package。Selected Skill Package 连接 Memory Store、Tool Registry、Context Assembly 和 LLM，最后输出 Final Answer。旁边有 Trace 记录节点。技术书籍风格，不使用卡通机器人，不使用复杂科技背景。

## 3. 运行前准备：API Key、模型名与依赖

这个实验继续使用 OpenAI Python SDK 和 API Key。SDK 是 Software Development Kit 的缩写，可以理解为官方提供的一组开发工具，让 Python 程序更方便地调用 API。API Key 是访问 API 的凭证，不要写进代码，也不要提交到公开仓库。

**命令 8P-1：安装依赖并设置环境变量**

```powershell
# PowerShell
python -m pip install --upgrade openai
$env:OPENAI_API_KEY = "你的 API Key"
$env:OPENAI_MODEL = "gpt-5.5"
python skill_agent.py
```

代码默认从环境变量 OPENAI\_MODEL 读取模型名。如果默认模型在你的账号里不可用，把 OPENAI\_MODEL 改成你当前可用的模型即可。这个实验的重点是 Agent 架构，不是绑定某一个具体模型名。

## 4. 这个实验会构造哪些对象

在看完整代码前，先把对象边界说清楚。这里每个对象都对应第 8 章的一个概念，读者看代码时可以按这个表对照。

| **对象** | **代码里的位置** | **作用** |
| --- | --- | --- |
| Skill Registry | SKILL\_REGISTRY | 只保存 metadata，用于 Skill Selection |
| Skill Package | SKILL\_PACKAGES | 保存完整 Skill 细节，只有被选中后才加载 |
| Skill Selection | select\_skill() | 让 LLM 根据 Goal 和 Registry metadata 选择 Skill |
| Progressive Loading | load\_skill\_registry\_metadata(), load\_skill\_details() | 先轻量加载，再按需加载完整能力包 |
| Allowed Tools | skill\['allowed\_tools'\] | 限制当前 Skill 能调用哪些 Tool |
| Tool Registry | TOOL\_REGISTRY | Runtime 本地持有的实际工具函数 |
| Memory Retrieval | retrieve\_memory() | 根据选中 Skill 取回相关记忆 |
| Agent Runtime | run\_agent() | 串联 Goal、Skill、Memory、Tool、Context 和 Final Answer |

## 5. 完整代码：最小 SkillAgent

下面的代码分成四段展示，实际使用时放在同一个 skill\_agent.py 文件里。这个版本仍然是教学用最小实现，但已经能体现第 8 章最关键的工程思想：不要全量加载所有能力，只加载当前任务需要的 Skill。

**代码 8P-2A：导入依赖、API 调用与 JSON 解析**

```python
# skill_agent.py
# 运行前准备：
#   1. python -m pip install --upgrade openai
#   2. 设置环境变量 OPENAI_API_KEY
#   3. 如果默认模型不可用，设置 OPENAI_MODEL 为你账号可用的模型
import copy
import json
import os
from datetime import datetime, timezone
from pathlib import Path

try:
    from openai import OpenAI
except ImportError as exc:
    raise SystemExit("请先运行：python -m pip install --upgrade openai") from exc


MODEL = os.getenv("OPENAI_MODEL", "gpt-5.5")
MEMORY_FILE = Path("skill_memory_store.json")


def now_iso():
    return datetime.now(timezone.utc).isoformat()


def require_api_key():
    if not os.getenv("OPENAI_API_KEY"):
        raise SystemExit("请先设置环境变量 OPENAI_API_KEY")


def response_text(response):
    """兼容不同 SDK 版本的最小文本提取函数。"""
    if getattr(response, "output_text", None):
        return response.output_text

    chunks = []
    for item in getattr(response, "output", []) or []:
        for part in getattr(item, "content", []) or []:
            text = getattr(part, "text", None)
            if text:
                chunks.append(text)
    return "\n".join(chunks)


def call_llm_text(client, system_prompt, user_prompt):
    response = client.responses.create(
        model=MODEL,
        input=[
            {"role": "system", "content": system_prompt},
            {"role": "user", "content": user_prompt},
        ],
    )
    return response_text(response).strip()


def parse_json_object(text):
    """从模型输出中提取 JSON 对象。生产系统应使用更严格的结构化输出约束。"""
    try:
        return json.loads(text)
    except json.JSONDecodeError:
        start = text.find("{")
        end = text.rfind("}")
        if start >= 0 and end > start:
            return json.loads(text[start : end + 1])
        raise


def call_llm_json(client, system_prompt, user_prompt):
    text = call_llm_text(client, system_prompt, user_prompt)
    return parse_json_object(text)
```

**代码 8P-2B：Memory、Knowledge Mock Tool 与 Tool Registry**

```python
def load_memory():
    if MEMORY_FILE.exists():
        return json.loads(MEMORY_FILE.read_text(encoding="utf-8"))

    return [
        {
            "id": "mem_user_report_style",
            "type": "user_preference",
            "scope": "global",
            "content": "用户偏好：行业分析报告默认先给结论，再给依据，最后给风险和建议。",
            "confidence": "high",
            "source": "seed",
            "updated_at": now_iso(),
        },
        {
            "id": "mem_ev_analysis_frame",
            "type": "procedural_memory",
            "scope": "ev_market_analysis",
            "content": "新能源汽车行业分析通常从销量、价格、政策、企业动态四个角度观察。",
            "confidence": "high",
            "source": "seed",
            "updated_at": now_iso(),
        },
    ]


def save_memory(memory_store):
    MEMORY_FILE.write_text(
        json.dumps(memory_store, ensure_ascii=False, indent=2),
        encoding="utf-8",
    )


def retrieve_memory(user_input, skill_id, memory_store):
    """最小 Memory Retrieval：只取 global 和当前 Skill 相关记忆。"""
    results = []
    for memory in memory_store:
        if memory["scope"] == "global":
            results.append(memory)
        elif memory["scope"] == skill_id:
            results.append(memory)
        elif "先给结论" in user_input and "先给结论" in memory["content"]:
            results.append(memory)
    return results[:5]


def maybe_write_memory_from_user_input(user_input, memory_store):
    """只写入明确、可复用的用户偏好。不要把所有对话都写入 Memory。"""
    if "先给结论" not in user_input:
        return []

    candidate = {
        "id": "mem_user_pref_conclusion_first",
        "type": "user_preference",
        "scope": "global",
        "content": "用户偏好：输出分析类结果时，优先采用先结论、后依据、再风险的结构。",
        "confidence": "high",
        "source": "explicit_user_input",
        "updated_at": now_iso(),
    }
    existing_ids = {item["id"] for item in memory_store}
    if candidate["id"] not in existing_ids:
        memory_store.append(candidate)
        return [candidate]
    return []


MARKET_KNOWLEDGE = {
    "销量变化": "最近三个月新能源汽车销量增速放缓，但头部品牌交付仍保持相对稳定。",
    "价格变化": "多家车企调整车型价格，部分品牌加大优惠力度，价格竞争仍然明显。",
    "政策变化": "部分地区继续推进以旧换新、购车补贴和充电基础设施支持政策。",
    "企业动态": "头部车企继续扩展智能驾驶、补能网络和海外市场，竞争焦点从单车价格转向综合能力。",
}


def retrieve_market_knowledge(topic):
    return {
        "source": "mock_ev_market_library",
        "topic": topic,
        "content": MARKET_KNOWLEDGE.get(topic, "没有找到相关资料。"),
    }


TOOL_REGISTRY = {
    "retrieve_market_knowledge": retrieve_market_knowledge,
}


TOOL_DESCRIPTIONS = {
    "retrieve_market_knowledge": "根据 topic 从新能源汽车行业资料库中取回一个资料片段。实验中用本地 mock 数据代替真实 RAG。",
}
```

**代码 8P-2C：Skill Registry、Skill Package 与 Skill Selection**

```python
SKILL_REGISTRY = [
    {
        "id": "ev_market_analysis",
        "name": "新能源汽车行业分析 Skill",
        "description": "用于分析新能源汽车行业在指定时间范围内的销量、价格、政策和企业动态变化。",
        "triggers": ["新能源汽车", "行业分析", "销量变化", "价格变化", "政策影响", "车企动态"],
        "version": "1.0.0",
        "permissions": ["retrieve_market_knowledge"],
    },
    {
        "id": "meeting_summary",
        "name": "会议纪要整理 Skill",
        "description": "用于把会议记录整理成议题、结论、待办事项和负责人。",
        "triggers": ["会议纪要", "会议记录", "待办事项", "行动项"],
        "version": "1.0.0",
        "permissions": [],
    },
]


SKILL_PACKAGES = {
    "ev_market_analysis": {
        "metadata": {
            "id": "ev_market_analysis",
            "name": "新能源汽车行业分析 Skill",
            "version": "1.0.0",
        },
        "instruction": (
            "你正在使用新能源汽车行业分析 Skill。必须围绕销量、价格、政策、企业动态组织分析。"
            "事实性结论必须来自 Observation 或 Retrieved Memory。资料不足时明确说明不足。"
        ),
        "required_inputs": ["time_range"],
        "procedure_topics": ["销量变化", "价格变化", "政策变化", "企业动态"],
        "allowed_tools": ["retrieve_market_knowledge"],
        "knowledge_sources": ["mock_ev_market_library"],
        "output_schema": [
            "结论摘要",
            "关键变化",
            "原因分析",
            "风险与不确定性",
            "资料来源",
        ],
        "failure_handling": [
            "如果用户没有提供时间范围，默认使用最近三个月并在输出中说明。",
            "如果资料不足，不要编造结论。",
            "如果工具不可用，说明当前无法完成资料检索。",
        ],
    },
    "meeting_summary": {
        "metadata": {
            "id": "meeting_summary",
            "name": "会议纪要整理 Skill",
            "version": "1.0.0",
        },
        "instruction": "你正在使用会议纪要整理 Skill。只根据用户提供的会议内容整理结论和待办。",
        "required_inputs": ["meeting_text"],
        "procedure_topics": [],
        "allowed_tools": [],
        "knowledge_sources": [],
        "output_schema": ["会议主题", "关键结论", "待办事项", "负责人", "风险"],
        "failure_handling": ["如果会议内容不足，先说明缺少哪些信息。"],
    },
}


def load_skill_registry_metadata():
    """第一阶段只加载 Skill Metadata，不加载完整 Skill Package。"""
    return copy.deepcopy(SKILL_REGISTRY)


def select_skill(client, goal, registry_metadata):
    system = (
        "你是 Agent Runtime 的 Skill Selection 模块。"
        "你只能根据 Skill Registry 的 metadata 选择一个 Skill。"
        "只输出 JSON，不要输出解释性文字。"
    )
    user = json.dumps(
        {
            "goal": goal,
            "available_skills": registry_metadata,
            "output_format": {"skill_id": "string or null", "reason": "short string"},
        },
        ensure_ascii=False,
        indent=2,
    )
    decision = call_llm_json(client, system, user)
    skill_id = decision.get("skill_id")
    if skill_id not in SKILL_PACKAGES:
        return {"skill_id": None, "reason": "没有匹配到可用 Skill"}
    return decision


def load_skill_details(skill_id):
    """第二阶段只加载被选中的 Skill Package。"""
    if skill_id is None:
        return None
    return copy.deepcopy(SKILL_PACKAGES[skill_id])
```

**代码 8P-2D：Runtime 主流程、Progressive Loading 与最终输出**

```python
def extract_goal(client, user_input):
    system = "你是 Agent 的 Goal 识别模块。把用户输入整理成一个明确、可执行的任务目标。只输出 JSON。"
    user = json.dumps(
        {
            "user_input": user_input,
            "output_format": {"goal": "明确任务目标", "assumptions": ["必要假设"]},
        },
        ensure_ascii=False,
        indent=2,
    )
    result = call_llm_json(client, system, user)
    return result.get("goal", user_input), result.get("assumptions", [])


def build_initial_state(goal, skill):
    return {
        "goal": goal,
        "selected_skill": skill["metadata"]["id"] if skill else None,
        "loaded_skill_sections": [],
        "observations": [],
        "trace": [],
        "final_answer": None,
    }


def run_tool(tool_name, arguments, allowed_tools):
    if tool_name not in allowed_tools:
        raise PermissionError(f"Tool not allowed by selected Skill: {tool_name}")
    if tool_name not in TOOL_REGISTRY:
        raise KeyError(f"Unknown tool: {tool_name}")
    result = TOOL_REGISTRY[tool_name](**arguments)
    return {
        "tool_name": tool_name,
        "arguments": arguments,
        "source": result["source"],
        "summary": f"{result['topic']}: {result['content']}",
    }


def execute_skill_procedure(skill, state):
    allowed_tools = skill["allowed_tools"]
    for topic in skill["procedure_topics"]:
        observation = run_tool(
            "retrieve_market_knowledge",
            {"topic": topic},
            allowed_tools,
        )
        state["observations"].append(observation)
        state["trace"].append(
            {
                "step": f"retrieve {topic}",
                "tool": observation["tool_name"],
                "source": observation["source"],
            }
        )


def build_final_context(user_input, goal, skill, state, retrieved_memories):
    memory_lines = [
        f"- [{m['type']}/{m['scope']}] {m['content']}" for m in retrieved_memories
    ]
    observation_lines = [
        f"- [{o['source']}] {o['summary']}" for o in state["observations"]
    ]
    return {
        "user_input": user_input,
        "goal": goal,
        "skill_instruction": skill["instruction"] if skill else "",
        "output_schema": skill["output_schema"] if skill else [],
        "failure_handling": skill["failure_handling"] if skill else [],
        "retrieved_memory": memory_lines,
        "observations": observation_lines,
    }


def generate_final_answer(client, context):
    system = (
        "你是一个使用 Skill 的 Agent。"
        "必须遵守 skill_instruction 和 output_schema。"
        "只能基于 observations 和 retrieved_memory 生成事实性结论。"
    )
    user = json.dumps(context, ensure_ascii=False, indent=2)
    return call_llm_text(client, system, user)


def run_agent(user_input):
    require_api_key()
    client = OpenAI()

    memory_store = load_memory()
    written_memories = maybe_write_memory_from_user_input(user_input, memory_store)

    goal, assumptions = extract_goal(client, user_input)

    registry_metadata = load_skill_registry_metadata()
    decision = select_skill(client, goal, registry_metadata)
    skill = load_skill_details(decision["skill_id"])

    if skill is None:
        raise SystemExit("没有匹配到可用 Skill，本实验到此停止。")

    state = build_initial_state(goal, skill)
    state["loaded_skill_sections"] = [
        "metadata",
        "instruction",
        "procedure_topics",
        "allowed_tools",
        "output_schema",
        "failure_handling",
    ]

    retrieved_memories = retrieve_memory(user_input, skill["metadata"]["id"], memory_store)
    execute_skill_procedure(skill, state)

    final_context = build_final_context(
        user_input=user_input,
        goal=goal,
        skill=skill,
        state=state,
        retrieved_memories=retrieved_memories,
    )
    state["final_answer"] = generate_final_answer(client, final_context)

    save_memory(memory_store)

    print("\n=== Goal ===")
    print(goal)
    print("\n=== Skill Selection ===")
    print(json.dumps(decision, ensure_ascii=False, indent=2))
    print("\n=== Written Memories ===")
    print(json.dumps(written_memories, ensure_ascii=False, indent=2))
    print("\n=== Trace ===")
    print(json.dumps(state["trace"], ensure_ascii=False, indent=2))
    print("\n=== Final Answer ===")
    print(state["final_answer"])


if __name__ == "__main__":
    run_agent(
        "请先给结论，帮我分析新能源汽车行业最近三个月的变化。"
    )
```

## 6. Progressive Loading 在代码里如何体现

本实验最重要的不是输出了什么报告，而是加载顺序。Progressive Loading 不是一句概念，它在代码里有明确对应关系。

| **阶段** | **代码位置** | **加载内容** | **为什么这样做** |
| --- | --- | --- | --- |
| 第一阶段 | load\_skill\_registry\_metadata() | 只加载 Skill 的 id、description、triggers、version、permissions | 让 Runtime 先低成本判断可能用哪个 Skill |
| 第二阶段 | select\_skill() | 把 Goal 和 Registry metadata 给 LLM | 只让模型基于轻量信息做 Skill Selection |
| 第三阶段 | load\_skill\_details() | 只加载被选中的 Skill Package | 避免所有 Skill 的完整 Instruction、Procedure、Output Schema 都进入 Context |
| 第四阶段 | execute\_skill\_procedure() | 只执行 selected Skill 允许的 Tool | 用 allowed\_tools 保持权限边界 |
| 第五阶段 | build\_final\_context() | 只把当前 Skill、Memory、Observation 组装进最终输入 | 减少无关规则干扰 |

> **关键理解**
>
> 如果把所有 SKILL\_PACKAGES 一开始都放进 Context，这个实验就失去了第 8 章的重点。Skill System 的价值不是让 Prompt 更长，而是让 Runtime 知道何时只加载必要能力。

## 7. 第八章概念在代码里的对应关系

这一节是读代码时最重要的对照表。第 8 章讲的是概念，本实验把它们落到了最小实现中。

| **第八章概念** | **代码位置** | **说明** |
| --- | --- | --- |
| Skill | SKILL\_PACKAGES\['ev\_market\_analysis'\] | 面向新能源汽车行业分析任务的可复用能力包 |
| Skill Registry | SKILL\_REGISTRY | 只保存用于选择 Skill 的轻量信息 |
| Skill Selection | select\_skill() | 根据 Goal 从多个 Skill 中选择当前任务最匹配的一个 |
| Progressive Loading | load\_skill\_registry\_metadata(), load\_skill\_details() | 先加载 metadata，选中后再加载完整 Skill Package |
| Skill Invocation | run\_agent() 中 select\_skill 后加载并执行 skill | 选择、加载并启用 Skill 的过程 |
| Allowed Tools | skill\['allowed\_tools'\] | 限制当前 Skill 可以使用 retrieve\_market\_knowledge |
| Permission Boundary | run\_tool() | 如果 Tool 不在 allowed\_tools 中，直接拒绝执行 |
| Context Assembly | build\_final\_context() | 把 Skill Instruction、Output Schema、Memory 和 Observation 组装给 LLM |
| Trace | state\['trace'\] | 记录 Runtime 执行了哪些工具和步骤 |

注意，Tool Registry 和 Skill Registry 不是一回事。Tool Registry 记录本地有哪些可执行工具函数；Skill Registry 记录系统有哪些可复用能力。Skill 可以引用 Tool，但 Tool 不是 Skill。

## 8. 运行后应该观察什么

真实 LLM 的输出可能每次略有差异，所以不要用逐字一致作为判断标准。这个实验要观察的是运行链路是否符合 Skill System 的设计。

**运行观察 8P-3：Trace 应该体现的变化**

```text
一次正常运行中，Trace 应该能看见这些变化：
1. Goal 被整理出来，不再只是原始 User Input。
2. Runtime 先加载 Skill Registry metadata。
3. Skill Selection 选择 ev_market_analysis，而不是加载所有 Skill。
4. Runtime 只加载 ev_market_analysis 的完整 Skill Package。
5. loaded_skill_sections 显示本次实际加载了 instruction、procedure_topics、allowed_tools、output_schema 等内容。
6. execute_skill_procedure() 按 Skill 的 procedure_topics 调用 retrieve_market_knowledge。
7. Observation 写入 State，Final Answer 基于 Observation、Memory 和 Skill Output Schema 生成。

你不需要要求模型每次输出逐字一致。重点观察链路是否正确：
User Input -> Goal -> Skill Registry -> Skill Selection -> Selected Skill Package -> Runtime 执行 -> Observation -> Final Answer
```

- **Goal：** 应该从用户输入中整理出明确目标，而不是直接把原句当成系统状态。

- **Skill Selection：** 应该选择 ev\_market\_analysis，并给出简短 reason。

- **Loaded Skill Sections：** 应该只加载选中 Skill 的 metadata、instruction、procedure\_topics、allowed\_tools、output\_schema 等内容。

- **Tool Permission：** Runtime 只允许执行 selected Skill 的 allowed\_tools。

- **Observation：** retrieve\_market\_knowledge 的结果会被整理成 Observation 写入 State。

- **Final Answer：** 最终答案应该受 Skill Instruction 和 Output Schema 约束，而不是自由发挥。

## 9. 这个实验还不是生产级 Skill System

这个实验有意保持简单。Skill Registry 和 Skill Package 都写在 Python 字典里；Knowledge Tool 用本地 mock 数据；Skill Selection 用 LLM 直接选择；Memory Retrieval 用 scope 做简单筛选。这些设计足够教学，但还不是正式生产系统。

| **当前实验** | **生产系统可能会升级为** |
| --- | --- |
| Python 字典里的 Skill Registry | 数据库、配置中心、插件系统或专门的 Skill 管理服务 |
| Python 字典里的 Skill Package | 目录化能力包、版本化配置、权限策略和发布流程 |
| LLM 直接选择 Skill | 规则预筛、向量检索、分类模型、LLM 复核的混合选择机制 |
| 本地 mock Knowledge Tool | 真实 RAG、搜索 API、数据库查询或企业知识库 |
| 简单 allowed\_tools 检查 | 角色权限、租户权限、用户确认、高风险操作审批 |
| 手动观察 Trace | 可观测性平台、审计日志、自动 Evals 和回归测试 |

> **工程判断**
>
> 不要一开始就做复杂 Skill 平台。先把最小链路跑通：Registry 能被读取，Skill 能被选择，细节能按需加载，Tool 能被权限限制，输出能被 Schema 约束。这个链路稳定后，再考虑版本管理、权限系统、Evals 和可观测性。

## 10. 实验小结：从 MemoryAgent 到 SkillAgent

实验 5 解决的是 Agent 如何接入真实 API，并在任务循环中使用 Memory。实验 6 在这个基础上增加 Skill System：Agent 不再把所有规则一次性塞进 Prompt，而是先根据 Goal 选择 Skill，再加载该 Skill 的 Instruction、Procedure、Allowed Tools、Output Schema 和 Failure Handling。

| **你应该带走的结论** | **一句话解释** |
| --- | --- |
| Skill 不是高级 Prompt | Prompt 是模型输入，Skill 是可复用能力包 |
| Skill 需要 Registry | Runtime 需要先知道系统有哪些可用能力 |
| Skill 需要 Progressive Loading | 先加载 metadata，选中后再加载细节 |
| Skill 需要 Runtime | Skill 本身不执行任务，真正执行的是 Agent Runtime |
| Skill 需要权限边界 | allowed\_tools 决定当前 Skill 能调用哪些 Tool |
| Skill 需要可观察 | Trace 应记录选择了哪个 Skill、加载了什么、执行了哪些 Tool |

到这里，读者已经可以把第 8 章的抽象概念放进一个最小程序里理解：Skill System 不是让 Agent 变得更神秘，而是让能力可以被封装、选择、加载、执行和管理。后续如果继续扩展，可以把这个 SkillAgent 接入真实 RAG、更多 Tool、更严格的 Permission Boundary 和自动 Evals。

## 参考资料

OpenAI API Reference: https://platform.openai.com/docs/api-reference/responses/create

OpenAI Function Calling Guide: https://platform.openai.com/docs/guides/function-calling

OpenAI Agents SDK Guide: https://platform.openai.com/docs/guides/agents
