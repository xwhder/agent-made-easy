# 第 6 章章末实践

**接入 API 的 Memory Agent**

*从实验四的 Tool Calling 继续往前：用 Responses API 跑通 State、Memory 与 Context*

实验四已经把 Tool System 跑起来了：LLM 看到 Tool Description，输出 Tool Call；Agent Runtime 负责校验、执行 Tool，并把 Raw Result 整理成 Observation。实验四为了突出 Tool System，用的是 mock LLM，也就是用规则模拟模型判断。

从这个实验开始，我们正式加入 API 调用。API 是 Application Programming Interface 的缩写，可以理解为程序和外部服务之间约定好的调用接口。在这里，API 调用指我们的 Python 程序通过 OpenAI API 请求 LLM 生成结果。LLM 仍然指 Large Language Model，也就是大语言模型。

本实验要承接第六章：我们不只让模型调用工具，还要让 Agent Runtime 明确维护 State，保存 Memory，并在每一轮调用 LLM 前做 Context Assembly。Context Assembly 是把 User Input、State、Retrieved Memory、Observation 和 Tool Description 组装成这一轮模型输入的过程。

> **阅读提示**
>
> 这一章会比前几个实验更详细，因为它第一次接入真实 API。你不需要一口气记住所有代码，先看清楚 API 调用在系统里的位置。
>
> 先掌握到这里：API 负责让程序调用 LLM；State 仍然由 Agent Runtime 管；Memory 仍然由系统保存和取回；Tool 仍然由本地 Runtime 执行。

## 1. 为什么从这个实验开始接 API

前面的实验一直避免接真实 LLM，不是因为 API 不重要，而是因为过早接 API 会让读者把注意力放在模型返回格式、环境变量、网络错误和费用上，反而看不清 Agent 的运行机制。现在已经讲完 Tool System 和 Memory System，可以开始把 mock LLM 换成真实 LLM。

这个实验只替换一个关键位置：实验四里的 mock\_llm\_decide()。其他部分仍然保持 Agent 的系统边界：Tool Registry 仍然在本地，Tool 执行仍然由 Runtime 完成，Observation 仍然写回 State，Memory 仍然由系统筛选和保存。

| **实验四** | **实验五** | **为什么这样改** |
| --- | --- | --- |
| mock\_llm\_decide(messages) | client.responses.create(...) | 把规则模拟的模型判断换成真实 LLM API 调用。 |
| Tool Description 放入 Messages | Tool Description 作为 tools 参数传给 API | 让模型用标准 Function Calling 形式返回 Tool Call。 |
| Runtime 解析 JSON Action | Runtime 解析 response.output 里的 function\_call | 模型输出格式更接近真实 API。 |
| Tool 执行在本地 | Tool 仍然执行在本地 | 模型提出调用请求，系统决定是否执行。 |
| State 记录 done/missing | State 继续记录任务进度 | 第六章强调：任务状态不是模型自动保存的。 |
| 无长期 Memory | 加入 memory\_store.json | 让读者看到 Memory 如何被保存、取回并进入 Context。 |

> **先掌握到这里**
>
> 接入 API 不等于把 Agent 的全部责任交给模型。真实系统里，LLM 只负责根据输入做判断和生成；State、Memory、Tool 执行、权限边界和错误处理仍然属于 Agent Runtime。

## 2. 这个实验要跑通的完整链路

本实验的链路比实验四多了 Memory System。Memory System 是 Agent 的记忆管理系统，用来决定哪些信息保存、如何取回、什么时候放回 Context。这里的 Memory 不是模型自己记住，而是系统在模型外部保存的信息。

- **State：** 当前任务的运行账本。本实验里记录 goal、done、missing、observations、trace 和 final\_answer。

- **Memory：** 可跨轮次或跨任务复用的信息。本实验里用 memory\_store.json 保存用户偏好和分析框架。

- **Retrieved Memory：** 从 Memory Store 中取回的相关记忆。它不是全部 Memory，只是本轮相关的一小部分。

- **Context：** 本轮送给 LLM 的输入材料。本实验里由 user\_input、state、retrieved\_memory 和 recent\_observations 组成。

- **Tool Call：** 模型通过 API 返回的函数调用请求。它仍然不是工具已经执行。

- **Observation：** 工具结果被 Runtime 整理后的任务信息，会进入 State，并影响下一轮 Context。

> **插图 6P-1：接入 API 后的 State、Memory 与 Tool Calling 链路**
>
> 图中要表达什么：展示实验五如何把实验四的 Tool Calling 链路和第六章的 Memory System 接起来。重点表达：LLM API 负责判断，Runtime 负责状态、记忆、工具执行和权限边界。
>
> 图中包含哪些元素：左侧是 User Input，进入 Agent Runtime。Runtime 内部包含 State、Memory Store、Memory Retrieval、Context Assembly、Tool Registry。Context Assembly 输出 input 和 tools 给 OpenAI Responses API。API 返回 function\_call。Runtime 执行本地 Tool，得到 Raw Result，整理成 Observation，更新 State，并按策略写入 Memory。Observation 和 Retrieved Memory 再进入下一轮 Context。图中还要标出 previous\_response\_id 只用于把 Tool Output 交还给同一次 API 推理链，不等于长期 Memory。
>
> 生图提示：一张简洁专业的 Agent 实验流程图，浅色背景，技术书籍风格。左侧 User Input 指向 Agent Runtime，Runtime 中包含 State、Memory Store、Memory Retrieval、Context Assembly、Tool Registry。Context Assembly 连接 OpenAI Responses API，API 返回 function\_call 到 Runtime。Runtime 执行本地 Tool，生成 Observation，更新 State，并可写入 Memory Store。Observation 和 Retrieved Memory 回到下一轮 Context Assembly。用一个小注释标出 previous\_response\_id 只表示 API 响应接续，不是 Agent Memory。不要卡通机器人，不要复杂科技背景。

## 3. 运行前准备：API Key、模型名与依赖

这个实验需要 OpenAI Python SDK 和 API Key。SDK 是 Software Development Kit 的缩写，可以理解为官方提供的一组开发工具，让 Python 程序更方便地调用 API。API Key 是访问 API 的凭证，不要写进代码，也不要提交到公开仓库。

**命令 6P-1：安装依赖并设置环境变量**

```powershell
# PowerShell
python -m pip install --upgrade openai
$env:OPENAI_API_KEY = "你的 API Key"
$env:OPENAI_MODEL = "gpt-5.5"
python memory_api_agent.py
```

代码里默认从环境变量 OPENAI\_MODEL 读取模型名，如果没有设置，就使用 gpt-5.5。模型名称会随着平台更新而变化；如果你的账号暂时不能使用这个模型，把 OPENAI\_MODEL 改成你当前可用的模型即可。

> **安全提醒**
>
> 不要把 API Key 写在 Word、代码文件或截图里。最简单的做法是通过环境变量传入。对读者来说，先掌握“程序用 API Key 调用 LLM 服务”这一点就够了。

## 4. 最小 API 调用：先确认 LLM 能被程序调用

在进入完整 Agent 前，先理解最小 API 调用。client.responses.create(...) 可以理解为发起一次模型请求：程序把 input 发给 LLM，LLM 返回 response。这个 response 可能直接包含文本，也可能包含 function\_call。

**代码 6P-2：最小 Responses API 调用**

```python
from openai import OpenAI

client = OpenAI()

response = client.responses.create(
    model="gpt-5.5",
    input="用一句话解释 Agent 的 State 是什么。"
)

print(response.output_text)
```

这段代码还不是 Agent。它只是证明 Python 程序可以通过 API 调用 LLM。Agent 需要在 API 调用前后补上 Context Assembly、Tool System、State Update、Memory Retrieval 和 Memory Write。

## 5. 完整代码：接入 API 的 Memory Agent

下面的代码分成四段展示，实际使用时放在同一个 memory\_api\_agent.py 文件里。它不是生产级 Agent，但已经把实验四和第六章的关键链路放到一起：真实 API 调用、Function Calling、本地 Tool 执行、State、Memory Retrieval、Memory Write 和 Context Assembly。

**代码 6P-3A：导入依赖、配置、Memory Store 与 State**

```python
# memory_api_agent.py
# 运行前准备：
#   1. python -m pip install --upgrade openai
#   2. 设置环境变量 OPENAI_API_KEY
#   3. 如果默认模型不可用，设置 OPENAI_MODEL 为你账号可用的模型

import json
import os
from datetime import datetime, timezone
from pathlib import Path

try:
    from openai import OpenAI
except ImportError as exc:
    raise SystemExit("请先运行：python -m pip install --upgrade openai") from exc


MODEL = os.getenv("OPENAI_MODEL", "gpt-5.5")
MEMORY_FILE = Path("memory_store.json")


def now_iso():
    return datetime.now(timezone.utc).isoformat()


def require_api_key():
    if not os.getenv("OPENAI_API_KEY"):
        raise SystemExit("请先设置环境变量 OPENAI_API_KEY")


def load_memory():
    if MEMORY_FILE.exists():
        return json.loads(MEMORY_FILE.read_text(encoding="utf-8"))

    return [
        {
            "id": "mem_user_report_style",
            "type": "user_preference",
            "scope": "global",
            "content": "用户偏好：行业分析报告默认先给结论，再给证据，最后给建议。",
            "confidence": "high",
            "source": "seed",
            "updated_at": now_iso(),
        },
        {
            "id": "mem_nev_analysis_frame",
            "type": "procedural_memory",
            "scope": "industry_analysis",
            "content": "新能源汽车行业变化通常从销量、价格、政策、电池成本四个角度观察。",
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


def build_initial_state(goal):
    return {
        "goal": goal,
        "done": [],
        "missing": ["销量变化", "价格变化", "政策变化", "电池成本变化"],
        "observations": [],
        "trace": [],
        "final_answer": None,
    }
```

**代码 6P-3B：Tool、Tool Description、Memory Retrieval 与 State Update**

```python
def search_industry_news(topic, time_range):
    """实验用 Tool：用本地模拟数据代替真实搜索 API。"""
    data = {
        "销量变化": "最近三个月新能源汽车销量增速放缓，但头部品牌交付仍相对稳定。",
        "价格变化": "多家车企调整车型价格，部分品牌加大优惠力度。",
        "政策变化": "部分地区继续推进以旧换新、购车补贴和充电基础设施支持政策。",
        "电池成本变化": "电池原材料价格波动影响整车定价和车企毛利空间。",
    }
    return {
        "source": "mock_public_search",
        "topic": topic,
        "time_range": time_range,
        "raw_result": data.get(topic, "没有找到相关信息。"),
    }


TOOL_DESCRIPTIONS = [
    {
        "type": "function",
        "name": "search_industry_news",
        "description": "查询新能源汽车行业某个主题的公开资料摘要。实验中用本地模拟数据代替真实搜索。",
        "parameters": {
            "type": "object",
            "properties": {
                "topic": {
                    "type": "string",
                    "enum": ["销量变化", "价格变化", "政策变化", "电池成本变化"],
                    "description": "要查询的行业变化主题。",
                },
                "time_range": {
                    "type": "string",
                    "description": "查询时间范围，例如：最近三个月。",
                },
            },
            "required": ["topic", "time_range"],
            "additionalProperties": False,
        },
    }
]


TOOL_REGISTRY = {
    "search_industry_news": search_industry_news,
}


def retrieve_memory(user_input, state, memory_store):
    """最小 Retrieval：按 scope 和关键词筛选相关记忆。"""
    text = user_input + "\n" + state["goal"]
    results = []
    for memory in memory_store:
        if memory["scope"] == "global":
            results.append(memory)
            continue
        if memory["scope"] == "industry_analysis" and "新能源" in text:
            results.append(memory)
            continue
        if any(word in memory["content"] for word in state["missing"]):
            results.append(memory)
    return results[:5]


def maybe_write_memory_from_user_input(user_input, memory_store):
    """只演示明确偏好的写入。不是所有 User Input 都应该写入 Memory。"""
    if "先给结论" not in user_input:
        return []

    candidate = {
        "id": "mem_user_pref_conclusion_first",
        "type": "user_preference",
        "scope": "global",
        "content": "用户偏好：报告优先采用先结论、后证据、再建议的结构。",
        "confidence": "high",
        "source": "explicit_user_input",
        "updated_at": now_iso(),
    }

    existing_ids = {item["id"] for item in memory_store}
    if candidate["id"] not in existing_ids:
        memory_store.append(candidate)
        return [candidate]
    return []


def make_observation(tool_name, tool_args, raw_result):
    return {
        "ok": True,
        "tool_name": tool_name,
        "topic": raw_result["topic"],
        "summary": f"{raw_result['topic']}: {raw_result['raw_result']}",
        "source": raw_result["source"],
        "arguments": tool_args,
    }


def update_state(state, observation):
    state["observations"].append(observation)
    topic = observation.get("topic")
    if topic in state["missing"]:
        state["missing"].remove(topic)
    if topic and topic not in state["done"]:
        state["done"].append(topic)
    return state
```

**代码 6P-3C：Context Assembly 与 Responses API 调用**

```python
def build_context(user_input, state, retrieved_memories):
    memory_lines = [
        f"- [{m['type']}/{m['scope']}] {m['content']}"
        for m in retrieved_memories
    ]
    observation_lines = [
        f"- {item['summary']} 来源={item['source']}"
        for item in state["observations"]
    ]

    return f"""
你是一个资料分析 Agent 的决策模块。

你的任务不是直接编造行业事实，而是根据 State 判断下一步是否需要调用 Tool。
每一轮最多调用一个 Tool。
如果 State.missing 还有内容，请优先调用 search_industry_news 查询第一个 missing 主题。
如果 State.missing 已经为空，请直接输出最终中文分析，不要再调用工具。

User Input:
{user_input}

Goal:
{state["goal"]}

State:
- done: {state["done"]}
- missing: {state["missing"]}

Retrieved Memory:
{chr(10).join(memory_lines) if memory_lines else "- 无相关记忆"}

Recent Observations:
{chr(10).join(observation_lines) if observation_lines else "- 暂无"}
""".strip()


def run_tool_call(function_call):
    tool_name = function_call.name
    if tool_name not in TOOL_REGISTRY:
        return {
            "ok": False,
            "error": f"Unknown tool: {tool_name}",
        }

    args = json.loads(function_call.arguments or "{}")
    result = TOOL_REGISTRY[tool_name](**args)
    return {
        "ok": True,
        "tool_name": tool_name,
        "arguments": args,
        "raw_result": result,
    }


def get_function_calls(response):
    return [
        item
        for item in response.output
        if getattr(item, "type", None) == "function_call"
    ]


def call_model_with_context(client, context):
    return client.responses.create(
        model=MODEL,
        input=[{"role": "user", "content": context}],
        tools=TOOL_DESCRIPTIONS,
    )


def send_tool_outputs(client, previous_response_id, tool_outputs):
    return client.responses.create(
        model=MODEL,
        previous_response_id=previous_response_id,
        input=tool_outputs,
    )
```

**代码 6P-3D：Agent 主循环、Tool Output 回传与运行入口**

```python
def run_agent(user_input):
    require_api_key()
    client = OpenAI()

    memory_store = load_memory()
    written = maybe_write_memory_from_user_input(user_input, memory_store)

    state = build_initial_state(
        "分析新能源汽车行业最近三个月的主要变化，并形成结构化总结。"
    )

    for round_id in range(1, 8):
        retrieved_memories = retrieve_memory(user_input, state, memory_store)
        context = build_context(user_input, state, retrieved_memories)

        response = call_model_with_context(client, context)
        function_calls = get_function_calls(response)

        state["trace"].append(
            {
                "round": round_id,
                "retrieved_memory_count": len(retrieved_memories),
                "model_response_id": response.id,
                "function_call_count": len(function_calls),
            }
        )

        if not function_calls:
            state["final_answer"] = response.output_text
            break

        tool_outputs = []
        for call in function_calls:
            tool_result = run_tool_call(call)

            if tool_result["ok"]:
                observation = make_observation(
                    tool_result["tool_name"],
                    tool_result["arguments"],
                    tool_result["raw_result"],
                )
                update_state(state, observation)
            else:
                observation = {
                    "ok": False,
                    "summary": tool_result["error"],
                }

            tool_outputs.append(
                {
                    "type": "function_call_output",
                    "call_id": call.call_id,
                    "output": json.dumps(observation, ensure_ascii=False),
                }
            )

        # 这一步把 Tool 的执行结果交还给同一次 API 推理链。
        # 注意：previous_response_id 不是 Agent State，也不是长期 Memory。
        follow_up = send_tool_outputs(client, response.id, tool_outputs)
        state["trace"][-1]["tool_follow_up_response_id"] = follow_up.id

        if not state["missing"]:
            final_context = build_context(user_input, state, retrieve_memory(user_input, state, memory_store))
            final_response = client.responses.create(
                model=MODEL,
                input=[{"role": "user", "content": final_context}],
            )
            state["final_answer"] = final_response.output_text
            break

    save_memory(memory_store)
    return state, memory_store, written


if __name__ == "__main__":
    user_input = (
        "我偏好报告先给结论，再给证据，最后给建议。"
        "请分析新能源汽车行业最近三个月的变化。"
    )
    final_state, memory_store, written_memories = run_agent(user_input)

    print("\n=== Written Memories ===")
    print(json.dumps(written_memories, ensure_ascii=False, indent=2))

    print("\n=== Trace ===")
    print(json.dumps(final_state["trace"], ensure_ascii=False, indent=2))

    print("\n=== Final State ===")
    print(json.dumps(
        {
            "done": final_state["done"],
            "missing": final_state["missing"],
            "observations": final_state["observations"],
        },
        ensure_ascii=False,
        indent=2,
    ))

    print("\n=== Final Answer ===")
    print(final_state["final_answer"])
```

## 6. API Function Calling 在这里到底做了什么

Function Calling 是一种让模型按工具调用格式提出请求的机制。模型不会直接执行 Python 函数，它只会返回类似“我要调用 search\_industry\_news，并传入 topic=销量变化”的结构化请求。真正执行 search\_industry\_news 的仍然是本地 Runtime。

| **位置** | **它是什么** | **它不是什么** |
| --- | --- | --- |
| TOOL\_DESCRIPTIONS | 写给模型看的工具说明，会通过 tools 参数传给 API。 | 不是实际工具函数。 |
| response.output 中的 function\_call | 模型返回的工具调用请求。 | 不是工具已经执行。 |
| TOOL\_REGISTRY | 本地 Runtime 持有的工具函数注册表。 | 不会自动暴露给模型执行。 |
| run\_tool\_call() | Runtime 根据 function\_call 执行本地函数。 | 不是 LLM 自己执行代码。 |
| function\_call\_output | Runtime 把工具执行结果交还给 API 的格式。 | 不是长期 Memory。 |

> **关键边界**
>
> API 返回 function\_call，只表示模型建议调用某个工具。是否执行、如何执行、执行失败怎么办、是否需要用户确认，仍然由 Agent Runtime 决定。这一点和实验四完全一致。

## 7. 第六章的 State、Memory 与 Context 在代码里如何对应

这一节是本实验最重要的部分。不要只盯着 API 调用本身，因为 API 只是调用 LLM 的方式。第六章真正要让读者理解的是：任务连续性来自系统设计，不是模型自己记住一切。

| **第六章概念** | **代码位置** | **说明** |
| --- | --- | --- |
| State | build\_initial\_state(), update\_state() | 记录当前任务进度，例如 done、missing、observations 和 trace。 |
| Memory Store | memory\_store.json, load\_memory(), save\_memory() | 保存可复用信息，例如用户偏好和分析框架。 |
| Memory Retrieval | retrieve\_memory() | 从 Memory Store 中取回本轮相关的信息，不是取回全部历史。 |
| Memory Write | maybe\_write\_memory\_from\_user\_input() | 只把明确、可复用的用户偏好写入 Memory。 |
| Context Assembly | build\_context() | 把 User Input、State、Retrieved Memory 和 Recent Observations 组装给 LLM。 |
| Observation | make\_observation() | 把 Tool Raw Result 整理成可写入 State 的任务信息。 |
| Trace | state\['trace'\] | 记录每一轮 API 响应、函数调用数量和后续响应 id，方便排查。 |

这里尤其要区分 previous\_response\_id 和 Memory。previous\_response\_id 是 Responses API 用来把工具输出交还给同一次模型推理链的机制，它解决的是“这次 function\_call 的结果如何返回给模型继续生成”。Memory 解决的是“哪些信息未来任务还要复用”。二者关系很近，但不是一回事。

> **先掌握到这里**
>
> previous\_response\_id 可以帮助 API 接续一次响应；State 让 Agent 知道当前任务做到哪里；Memory 让 Agent 在未来轮次或未来任务中复用信息。不要把它们混成一个概念。

## 8. 运行后应该观察什么

因为真实 LLM 的输出具有一定不确定性，运行结果不一定每次逐字一样。你不需要要求每次输出完全一致，只要观察链路是否正确：模型是否返回 function\_call，Runtime 是否执行本地 Tool，Observation 是否更新 State，Memory 是否被保存和取回。

**运行观察 6P-4：Trace 应该体现的变化**

```text
一次正常运行中，Trace 应该能看见这些信息：
Round 1:
  retrieved_memory_count: 2 或 3
  function_call_count: 1
  Tool Call: search_industry_news(topic="销量变化", time_range="最近三个月")
  State.done: ["销量变化"]

Round 2:
  Tool Call: search_industry_news(topic="价格变化", time_range="最近三个月")
  State.done: ["销量变化", "价格变化"]

Round 3:
  Tool Call: search_industry_news(topic="政策变化", time_range="最近三个月")

Round 4:
  Tool Call: search_industry_news(topic="电池成本变化", time_range="最近三个月")

Final:
  State.missing: []
  final_answer: 由真实 LLM 根据 State、Observation 和 Retrieved Memory 生成
```

- **Written Memories：** 如果 user\_input 中包含明确偏好，Runtime 会写入一条用户偏好 Memory。

- **Trace：** 每一轮会记录 model\_response\_id、function\_call\_count 和 tool\_follow\_up\_response\_id。

- **Final State：** done 应该逐步增加，missing 应该逐步减少。

- **Final Answer：** 最后答案由真实 LLM 根据 State、Observation 和 Retrieved Memory 生成。

## 9. 这个实验还不是生产级 Memory System

这个实验有意保持简单。它用 JSON 文件保存 Memory，用关键词和 scope 做 Retrieval，用本地模拟数据代替真实搜索 Tool。这些设计不是为了生产可用，而是为了让读者第一次接 API 时还能看清楚系统边界。

| **当前实验** | **生产系统可能会升级为** |
| --- | --- |
| memory\_store.json | 数据库、向量数据库、对象存储或专用记忆服务。 |
| 关键词 Retrieval | 语义检索、向量相似度、时间过滤、权限过滤和重排序。 |
| 本地模拟 search\_industry\_news | 真实搜索 API、数据库查询、企业知识库或浏览器工具。 |
| 简单用户偏好写入 | 带权限、来源、置信度、过期时间和用户确认的 Memory Write 策略。 |
| 单 Agent 循环 | 多 Agent、任务队列、异步工具执行、完整 Trace 和可观测性系统。 |

> **工程判断**
>
> 不要一接 API 就把所有历史都塞进 Prompt。这个实验保留 Memory Retrieval 的步骤，就是为了让读者看到：Memory 必须经过筛选后进入 Context。取回太少，模型缺背景；取回太多，模型会被噪音拖偏。

## 10. 实验小结：从实验四走到第六章

实验四证明了 Tool System 的边界：模型提出 Tool Call，Runtime 执行 Tool，Observation 更新 State。实验五在这个基础上加入真实 API 调用，并把第六章的 Memory System 放进循环：Memory Store 保存可复用信息，Retrieval 取回相关记忆，Context Assembly 把 State 和 Retrieved Memory 放进本轮输入。

| **你应该带走的结论** | **一句话解释** |
| --- | --- |
| API 调用不是 Agent 的全部 | API 只是让程序调用 LLM，Agent 还需要 Runtime、State、Memory 和 Tool System。 |
| Function Calling 不是模型执行函数 | 模型返回 function\_call，本地 Runtime 执行真正的函数。 |
| State 不是 previous\_response\_id | State 是你的任务账本，previous\_response\_id 只是 API 响应接续机制。 |
| Memory 不是聊天记录全量回放 | Memory 要经过写入策略、检索、筛选和 Context Assembly。 |
| 第六章的关键是任务连续性 | Agent 能继续做事，是因为系统保存并取回了正确的信息。 |

到这里，读者已经看到一个更接近真实 Agent 的最小形态：真实 LLM 负责判断，Tool System 负责连接外部能力，State 负责当前任务进度，Memory System 负责信息复用，Context Assembly 负责把正确的信息交给模型。后续再升级向量检索、真实搜索工具、多 Agent 或复杂权限系统，才不会失去主线。

## 参考资料

OpenAI API Reference: https://platform.openai.com/docs/api-reference/responses/create

OpenAI Function Calling Guide: https://platform.openai.com/docs/guides/function-calling

OpenAI Conversation State Guide: https://platform.openai.com/docs/guides/conversation-state
