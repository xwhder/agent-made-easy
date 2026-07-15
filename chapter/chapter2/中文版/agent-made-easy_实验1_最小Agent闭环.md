# 实验 1：最小 Agent 闭环

*第二章之后的过渡代码：从一次 LLM 调用到任务推进结构*

前两章已经说明：Agent 不是一次普通的 LLM 调用，也不是单纯会调用 Tool 的模型。为了让这个判断更具体，本实验用一段很小的 Python 风格伪代码，把 Goal、Context、Tool、Observation、State 和 Feedback Loop 串起来。

这不是生产级 Agent，也不需要 API Key、外部网络或任何框架。它的目标只有一个：让你看见 Agent 的最小闭环长什么样，以及它和一次普通 LLM 调用有什么区别。

> **阅读提示**
>
> 如果你暂时不想看代码，可以先跳过这一节。它不会影响第三章阅读。但是推荐看完，提前建立。

## 1. 实验目标

这个实验只回答一个问题：一个最小 Agent 和一次普通 LLM 调用有什么区别？普通 LLM 调用通常是输入一次 Context，得到一次 Response，然后流程结束。最小 Agent 闭环则会围绕 Goal 组织 Context，让 LLM 判断 Action；如果 Action 需要外部能力，系统执行 Tool，得到 Observation，更新 State，再进入下一轮判断。

| **对比项** | **普通 LLM 调用** | **最小 Agent 闭环** |
| --- | --- | --- |
| 核心结构 | 一次输入，一次输出。 | 围绕 Goal 多轮推进。 |
| Tool | 通常不调用外部能力。 | 可以在需要时调用 Tool。 |
| Observation | 没有明确的行动结果回流。 | Tool 或 Action 的结果会进入下一轮判断。 |
| State | 通常不保存任务进度。 | 保存已经做过什么、获得了什么、还缺什么。 |
| 停止方式 | 输出后自然结束。 | 由 Stop Condition 判断是否结束。 |

## 2. 一次普通 LLM 调用

先看普通 LLM 调用。这里的 LLM 是 Large Language Model，也就是大语言模型；Context 是模型当前能看到的信息；Response 是模型生成的输出。

**伪代码 E1-1：一次普通 LLM 调用**

```python
user_input = "帮我分析最近三个月新能源汽车行业变化"
context = build_context(user_input)
answer = llm_generate(context)
print(answer)
```

> **先掌握到这里**
>
> 这段代码表达的是一次性流程：系统把用户输入组织成 Context，交给 LLM，LLM 生成回答，流程就结束。它还没有 Tool、Observation、State，也没有持续推进任务的 Feedback Loop。

## 3. 最小 Agent 闭环

下面的代码开始接近 Agent。Goal 是任务目标；Tool 是系统开放给 Agent 的外部能力；Observation 是 Tool 或 Action 执行后返回的结果；State 是任务推进过程中的当前状态；Feedback Loop 是系统根据结果继续判断下一步的循环结构。

> **先掌握到这里**
>
> 这里出现的 mock\_llm\_decide 不是一个真实 LLM，而是用来模拟 LLM 根据 Context 判断下一步。这样做是为了让你先看懂结构，而不是卡在模型 API、网络搜索或框架配置上。

**伪代码 E1-2：最小 Agent 闭环**

```python
state = {
    "goal": "整理新能源汽车行业最近三个月变化",
    "notes": [],
    "step": 0,
    "done": False
}

MAX_STEPS = 3

def search_tool(query):
    return [
        "销量资料：最近三个月销量变化明显",
        "政策资料：部分地区补贴政策调整",
        "价格资料：多家车企调整车型价格",
        "成本资料：电池成本变化影响整车价格"
    ]

def build_context(state):
    return {
        "goal": state["goal"],
        "notes": state["notes"],
        "step": state["step"]
    }

def mock_llm_decide(context):
    if len(context["notes"]) == 0:
        return {
            "type": "tool_call",
            "tool": "search_tool",
            "query": "新能源汽车 最近三个月 行业变化"
        }

    return {
        "type": "final_answer",
        "content": (
            "可以从销量、政策、价格和电池成本四个"
            "角度整理一份结构化报告。"
        )
    }

while not state["done"] and state["step"] < MAX_STEPS:
    context = build_context(state)
    action = mock_llm_decide(context)

    if action["type"] == "tool_call":
        observation = search_tool(action["query"])
        state["notes"].extend(observation)
        state["step"] += 1
        continue

    if action["type"] == "final_answer":
        print(action["content"])
        state["done"] = True
```

> **先掌握到这里**
>
> 这段代码的重点不是语法，而是结构：Agent 不是只调用一次 LLM。它会根据 Goal 组织 Context，让 LLM 判断 Action；如果需要 Tool，系统执行 Tool，得到 Observation，写入 State，再进入下一轮判断。

## 4. 这段代码应该怎么看

这段代码可以从下表理解。读代码时不要先纠结具体语法，而要看每个模块在 Agent 闭环里解决什么问题。

| **代码元素** | **对应概念** | **它解决的问题** |
| --- | --- | --- |
| state | State | 保存任务目标、已获得资料、当前步骤和是否结束。 |
| goal | Goal | 规定 Agent 要推进的任务方向。 |
| build\_context | Context 组织 | 把当前 Goal 和 State 组织成模型可见信息。 |
| mock\_llm\_decide | LLM 判断 | 模拟 LLM 根据 Context 判断下一步 Action。 |
| search\_tool | Tool | 模拟外部能力，用来获得资料。 |
| observation | Observation | Tool 返回的结果，会写回 State。 |
| while 循环 | Feedback Loop | 让 Agent 根据新状态继续判断下一步。 |
| MAX\_STEPS / done | Stop Condition | 控制 Agent 什么时候停止，避免无限循环。 |

## 5. 它还不是完整 Agent

这个实验只展示最小闭环。真实 Agent 会比它复杂得多，但复杂度并不是凭空增加的，而是从这个闭环继续扩展出来的。

**mock\_llm\_decide：**会替换为真实 LLM 调用，并带有 Instruction、Context 和输出格式约束。

**search\_tool：**会替换为真实搜索、网页读取、文件读取、数据库查询或业务系统接口。

**简单 state：**会扩展为更完整的 State / Memory 系统，用来管理任务进度和长期信息。

**简单 MAX\_STEPS：**会扩展为更完整的 Stop Condition，包括权限限制、成本限制、失败处理和人工确认。

**直接 print：**会扩展为正式输出、报告生成、引用检查、Trace 和 Evaluation。

> **关键边界**
>
> 这段代码不是在教你“用几十行代码做出生产级 Agent”。它只是把最小结构显出来：Goal 让任务有方向，Context 让模型有依据，Tool 连接外部能力，Observation 带回结果，State 保持连续性，Feedback Loop 让任务继续推进。

## 6. 本实验小结

通过这个小实验，可以把前两章的判断落到代码结构上：LLM 提供理解和生成能力，但 Agent 的关键不只是 LLM，而是围绕 Goal 建立一个能够持续推进任务的循环系统。

普通 LLM 调用通常是一次输入、一次输出；最小 Agent 闭环则会在 Action、Observation、State 和 Feedback Loop 之间持续推进。后面的章节会把这个闭环继续拆开：第三章讲任务推进系统，第四章讲模型输入系统，第五章讲工具系统，第六章讲记忆系统。
