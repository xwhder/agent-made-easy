# 第 4 章章末实践

**给最小 ReAct Agent 加上模型输入系统**

*从 decide\_next\_action() 到 Messages，让 LLM 每一轮看见该看的信息*

这个文档是一个独立实战，接在“实验2：最小 ReAct Agent”之后，也接在第四章“模型输入系统”之后。实验2已经跑通了 ReAct 的任务推进骨架：Goal 给出方向，State 记录进度，Action 推动任务，Observation 更新状态，Feedback Loop 进入下一轮。它故意没有接真实 LLM，而是用 decide\_next\_action() 模拟模型判断。

本实验要补上的，正是第四章讲的那一层：每一轮判断之前，系统如何把 Instruction、Context、Messages 和输出约束组装好，让 LLM（Large Language Model，大语言模型）能基于足够的信息做下一步判断。

> **阅读提示**
>
> 这一节不是要读者一次记住所有代码细节。先掌握到这里：实验2解决“Agent 怎么循环推进任务”，本实验解决“每一轮推进前，LLM 应该看见什么输入”。真正接入某个模型厂商的 SDK 不是本节重点，后面只需要替换一个函数。

## 1. 本实验接在实验2哪里

实验2里最关键的简化，是把 LLM 的判断写成了一个规则函数：decide\_next\_action(state)。这很好，因为它让读者先看懂 ReAct 循环。但进入第四章以后，我们已经知道，真实 Agent 不应该只把 state 直接丢给模型。系统需要先做 Context Assembly，也就是上下文组装。

Context Assembly 指系统在调用 LLM 之前，选择、整理并排列本轮模型需要看到的信息。Context 是模型判断时参考的信息环境；Context Assembly 是把这些信息装配出来的过程。两者不要混在一起。

| **实验2已有部分** | **本实验新增部分** | **为什么要新增** |
| --- | --- | --- |
| Goal、State、Action、Observation、Feedback Loop | 保留不变 | 这些是 ReAct 任务推进骨架，不需要推倒重写。 |
| decide\_next\_action(state) | build\_context(state) + build\_messages(state) + mock\_llm\_decide(messages) | 先把状态整理成 Context，再把 Instruction、Context 和输出约束放进 Messages。 |
| fake\_search\_tool() | 暂时保留 | Tool 指 Agent 可以调用的外部能力接口。本实验还不讲完整 Tool System，避免抢第五章内容。 |
| trace | 记录 model\_output、Action、Observation 和 State | 读者能看到模型输入如何影响下一步判断。 |

## 2. 本实验要多加的一层：模型输入系统

在最小 ReAct Agent 里，任务循环看起来是：State -> Decide -> Action -> Observation -> State Update。加入模型输入系统后，它会多出一个前置动作：State -> Context Assembly -> Messages -> LLM Decide。

这里有四个词需要先放稳。Instruction 是对模型行为的要求，例如只能输出 JSON、不要自由发挥。Context 是本轮判断需要参考的信息，例如 Goal、已完成项、缺失项和已有 Observation。Messages 是对话式 LLM 接口常见的输入结构，通常由不同 role 的消息组成，例如 system 和 user。Output Constraint 是输出约束，规定模型结果必须长成什么样，方便系统继续解析和执行。

> **先掌握到这里**
>
> 不要把 Prompt 理解成用户随手写的一句话。在 Agent 系统里，一轮 Prompt 往往是系统组装出来的模型输入，可能以 Messages 的形式出现。读者现在只要记住：Instruction 管行为，Context 给依据，Messages 承载输入，输出约束让系统能继续执行。

## 3. 一个可运行的最小版本

下面这段代码可以直接运行，只使用 Python 标准库。它仍然没有接真实模型 API，因为这一节要让读者看清输入系统本身：Context 如何从 State 来，Messages 如何被组装，模型输出如何被解析成 Action。

代码里的 mock\_llm\_decide(messages) 用规则模拟 LLM 读完 Messages 后的判断。等读者真正接入模型时，只需要把这个函数替换成 call\_llm(messages)，并保持输出仍然是可解析的 Action JSON。

**代码 4P-1：minimal\_context\_agent.py**

```python
# minimal_context_agent.py
# 运行方式：保存为 minimal_context_agent.py 后执行 python minimal_context_agent.py
# 这个示例只使用 Python 标准库，重点是展示 Context Assembly 和 Messages。

import json


SYSTEM_INSTRUCTION = """
你是一个资料分析 Agent 的决策模块。
你的任务不是直接写完整报告，而是根据当前 Context 判断下一步 Action。
你只能输出 JSON，不要输出额外解释。
"""

OUTPUT_INSTRUCTION = """
如果还缺资料，输出：
{"type": "tool_call", "tool_name": "fake_search_tool", "tool_input": "...", "reason": "..."}

如果资料已经足够，输出：
{"type": "final_answer", "reason": "..."}
"""


def fake_search_tool(topic):
    """模拟一个外部 Tool。真实系统里，这里可以换成搜索、数据库或文件读取。"""
    data = {
        "销量变化": "最近三个月新能源汽车销量出现波动，部分月份环比增长放缓。",
        "政策变化": "部分地区调整了购车补贴、以旧换新和牌照支持政策。",
        "价格变化": "多家车企调整车型价格，部分品牌加大优惠力度。",
        "电池成本变化": "电池原材料价格变化影响整车定价和毛利空间。",
    }
    return {
        "topic": topic,
        "result": data.get(topic, "没有找到相关信息。"),
    }


def build_initial_state(goal):
    return {
        "goal": goal,
        "done": [],
        "missing": ["销量变化", "政策变化", "价格变化", "电池成本变化"],
        "observations": [],
        "final_answer": None,
    }


def build_context(state):
    """把 State 里和当前判断有关的信息整理成 Context。"""
    observations = []
    for item in state["observations"]:
        observations.append(f"- {item['topic']}: {item['result']}")

    return f"""
Goal: {state['goal']}
Done: {"、".join(state["done"]) if state["done"] else "无"}
Missing: {"、".join(state["missing"]) if state["missing"] else "无"}

Observations:
{chr(10).join(observations) if observations else "暂无"}
""".strip()


def build_messages(state):
    """把 Instruction、Context 和输出约束组装成 LLM 能接收的 Messages。"""
    context = build_context(state)
    return [
        {
            "role": "system",
            "content": SYSTEM_INSTRUCTION.strip(),
        },
        {
            "role": "user",
            "content": (
                "请根据下面 Context 判断下一步 Action。\n\n"
                f"{context}\n\n"
                f"{OUTPUT_INSTRUCTION.strip()}"
            ),
        },
    ]


def extract_context_field(context, field_name):
    prefix = field_name + ":"
    for line in context.splitlines():
        if line.startswith(prefix):
            return line.split(":", 1)[1].strip()
    return ""


def mock_llm_decide(messages):
    """
    这里模拟 LLM 阅读 Messages 后输出结构化 Action。
    真实接入时，只需要把这个函数替换成 call_llm(messages)。
    """
    user_content = messages[-1]["content"]
    missing_text = extract_context_field(user_content, "Missing")
    missing = [item.strip() for item in missing_text.split("、") if item.strip() and item != "无"]

    if missing:
        topic = missing[0]
        return json.dumps(
            {
                "type": "tool_call",
                "tool_name": "fake_search_tool",
                "tool_input": topic,
                "reason": f"当前 Context 还缺少{topic}，需要先补充资料。",
            },
            ensure_ascii=False,
        )

    return json.dumps(
        {
            "type": "final_answer",
            "reason": "当前 Context 中关键资料已经补齐，可以生成最终分析。",
        },
        ensure_ascii=False,
    )


def parse_action(model_output):
    """把模型输出解析成系统可以执行的 Action。真实系统里这里还要做更严格校验。"""
    action = json.loads(model_output)
    if action["type"] == "tool_call":
        if action.get("tool_name") != "fake_search_tool":
            raise ValueError("当前示例只允许调用 fake_search_tool")
        if not action.get("tool_input"):
            raise ValueError("tool_call 必须包含 tool_input")
    elif action["type"] != "final_answer":
        raise ValueError(f"未知 Action 类型: {action['type']}")
    return action


def execute_action(action):
    if action["type"] == "tool_call":
        return fake_search_tool(action["tool_input"])
    if action["type"] == "final_answer":
        return {"result": "可以基于已有 Observation 生成最终分析。"}
    raise ValueError(f"无法执行 Action: {action}")


def update_state(state, action, observation):
    if action["type"] == "tool_call":
        topic = observation["topic"]
        state["observations"].append(observation)
        if topic in state["missing"]:
            state["missing"].remove(topic)
        if topic not in state["done"]:
            state["done"].append(topic)

    if action["type"] == "final_answer":
        lines = ["新能源汽车行业最近三个月变化分析："]
        for item in state["observations"]:
            lines.append(f"- {item['topic']}: {item['result']}")
        state["final_answer"] = "\n".join(lines)

    return state


def run_agent():
    goal = "分析新能源汽车行业最近三个月的变化。"
    state = build_initial_state(goal)
    trace = []
    first_round_messages = None

    for round_id in range(1, 8):
        messages = build_messages(state)
        if first_round_messages is None:
            first_round_messages = messages

        model_output = mock_llm_decide(messages)
        action = parse_action(model_output)
        observation = execute_action(action)
        state = update_state(state, action, observation)

        trace.append(
            {
                "round": round_id,
                "model_output": model_output,
                "action": action,
                "observation": observation,
                "state": {
                    "done": list(state["done"]),
                    "missing": list(state["missing"]),
                },
            }
        )

        if action["type"] == "final_answer":
            break

    return state, trace, first_round_messages


if __name__ == "__main__":
    final_state, trace, first_messages = run_agent()

    print("=== First Round Messages ===")
    for message in first_messages:
        print(f"\n[{message['role']}]")
        print(message["content"])

    print("\n=== Trace ===")
    for item in trace:
        print(f"\nRound {item['round']}")
        print("Model Output:", item["model_output"])
        print("Action:", item["action"])
        print("Observation:", item["observation"])
        print("State:", item["state"])

    print("\n=== Final Answer ===")
    print(final_state["final_answer"])
```

## 4. 第一轮 Messages 长什么样

读代码时，不要一上来盯着所有函数。先看第一轮 Messages，因为它正好对应第四章的核心问题：LLM 每一轮到底看见什么。

**运行输出 4P-2：第一轮 Messages**

```text
=== First Round Messages ===

[system]
你是一个资料分析 Agent 的决策模块。
你的任务不是直接写完整报告，而是根据当前 Context 判断下一步 Action。
你只能输出 JSON，不要输出额外解释。

[user]
请根据下面 Context 判断下一步 Action。

Goal: 分析新能源汽车行业最近三个月的变化。
Done: 无
Missing: 销量变化、政策变化、价格变化、电池成本变化

Observations:
暂无

如果还缺资料，输出：
{"type": "tool_call", "tool_name": "fake_search_tool", "tool_input": "...", "reason": "..."}

如果资料已经足够，输出：
{"type": "final_answer", "reason": "..."}
```

- **system message：** 承载 Instruction，告诉模型它的角色、任务边界和输出规则。

- **user message：** 承载本轮 Context 和更具体的判断要求。这里的 user 不一定等于真实用户原话，而是系统组装后的模型输入。

- **Missing：** 来自 State，告诉模型当前还缺哪些资料。模型据此判断下一步 Action。

- **Observations：** 来自前几轮 Tool 或 Action 的结果。第一轮还没有 Observation，所以显示“暂无”。

> **关键边界**
>
> Observation 不会自动变成 Prompt。更准确的链路是：Observation 先更新 State，下一轮 Context Assembly 再决定哪些 State 信息进入 Context，最后被放进 Messages。这样设计，系统才能筛选、压缩、标注来源和处理风险内容。

## 5. 从输出看模型输入如何影响 Action

因为第一轮 Context 里 Missing 的第一项是“销量变化”，所以模型判断下一步 Action 是调用 fake\_search\_tool 查询销量变化。Tool 在这里指 Agent 系统可以调用的外部能力接口；本实验只是用本地函数模拟它。完整的 Tool Description、Tool Call、参数校验和 Permission Boundary 会在第五章展开。

**运行输出 4P-3：Trace 摘要**

```text
=== Trace ===

Round 1
Model Output: {"type": "tool_call", "tool_name": "fake_search_tool", "tool_input": "销量变化", "reason": "当前 Context 还缺少销量变化，需要先补充资料。"}
Action: {'type': 'tool_call', 'tool_name': 'fake_search_tool', 'tool_input': '销量变化', 'reason': '当前 Context 还缺少销量变化，需要先补充资料。'}
Observation: {'topic': '销量变化', 'result': '最近三个月新能源汽车销量出现波动，部分月份环比增长放缓。'}
State: {'done': ['销量变化'], 'missing': ['政策变化', '价格变化', '电池成本变化']}

Round 2
Action: {'type': 'tool_call', 'tool_name': 'fake_search_tool', 'tool_input': '政策变化', 'reason': '当前 Context 还缺少政策变化，需要先补充资料。'}
State: {'done': ['销量变化', '政策变化'], 'missing': ['价格变化', '电池成本变化']}

Round 3
Action: {'type': 'tool_call', 'tool_name': 'fake_search_tool', 'tool_input': '价格变化', 'reason': '当前 Context 还缺少价格变化，需要先补充资料。'}
State: {'done': ['销量变化', '政策变化', '价格变化'], 'missing': ['电池成本变化']}

Round 4
Action: {'type': 'tool_call', 'tool_name': 'fake_search_tool', 'tool_input': '电池成本变化', 'reason': '当前 Context 还缺少电池成本变化，需要先补充资料。'}
State: {'done': ['销量变化', '政策变化', '价格变化', '电池成本变化'], 'missing': []}

Round 5
Action: {'type': 'final_answer', 'reason': '当前 Context 中关键资料已经补齐，可以生成最终分析。'}
State: {'done': ['销量变化', '政策变化', '价格变化', '电池成本变化'], 'missing': []}
```

这个 Trace 的重点不是“规则函数多聪明”，而是让读者看见输入系统的作用：每一轮 State 改变后，Context 就会改变；Context 改变后，Messages 也会改变；LLM 看到的内容不同，下一步 Action 也随之变化。

## 6. 代码里的概念对应关系

下面这张表是本实验最重要的阅读地图。小白读者可以先看它建立整体感；懂技术的读者可以用它检查代码边界是否清楚。

| **代码位置** | **对应概念** | **它承担什么职责** |
| --- | --- | --- |
| SYSTEM\_INSTRUCTION | Instruction | 约束模型角色和行为：不要直接写报告，只判断下一步 Action。 |
| OUTPUT\_INSTRUCTION | Output Constraint | 规定模型只能输出两类 JSON：tool\_call 或 final\_answer。 |
| build\_context(state) | Context Assembly 的一部分 | 从 State 中选择 Goal、done、missing 和 observations，整理成本轮 Context。 |
| build\_messages(state) | Messages | 把 system message 和 user message 组装成模型输入结构。 |
| mock\_llm\_decide(messages) | LLM Decide 的模拟层 | 模拟 LLM 根据 Messages 输出下一步 Action。真实接入时替换这里。 |
| parse\_action(model\_output) | 输出解析与校验 | 把模型文本解析成系统可执行的 Action，并做最小校验。 |
| execute\_action(action) | Action 执行 | 根据 Action 调用 Tool 或进入最终输出。 |
| update\_state(...) | State Update | 把 Observation 写回 State，让下一轮 Context 发生变化。 |
| trace | Trace | 记录每轮模型输出、Action、Observation 和 State，方便复盘。 |

## 7. 为什么这还不是完整 Tool Calling

读者可能会注意到，代码里已经出现了 tool\_call 和 fake\_search\_tool。那为什么这还不是第五章要讲的 Tool Calling？原因是：本实验只关心模型输入如何组织，并没有建立完整 Tool System。

- **Tool Description：** 还没有正式出现。真实系统需要把工具说明写给 LLM 看，让模型知道工具能做什么、参数是什么、什么时候该用。

- **Tool Call：** 这里只是一个简化 JSON。真实系统里还要校验工具名、参数 Schema、权限、成本和风险。

- **Runtime：** 这里由 execute\_action() 简化模拟。真实 Runtime 要负责解析模型输出、执行或拒绝 Tool Call、记录 Trace、处理失败。

- **Observation：** 这里直接把工具返回结果写回 State。真实系统通常要摘要、去噪、标注来源，并防止外部内容污染 Instruction。

> **先掌握到这里**
>
> 本实验的价值是把第四章落地：LLM 不是凭空判断下一步，而是根据系统组装好的 Messages 判断下一步。至于模型怎么知道有哪些 Tool、Tool Call 谁来执行、失败怎么办、权限怎么控，这些正好进入第五章。

## 8. 如果要接真实 LLM，替换哪里

真实接入 LLM 时，不建议把 API 调用散落在 Agent 循环的各个位置。更稳的做法是只替换 mock\_llm\_decide(messages) 这一层，保留 build\_context()、build\_messages()、parse\_action() 和 update\_state() 的边界。

**伪代码 4P-4：真实 LLM 的替换点**

```python
def call_llm(messages):
    """
    这里接入你选择的 LLM SDK 或 HTTP API。
    输入仍然是 messages。
    输出必须仍然能被 parse_action() 解析成 Action JSON。
    """
    model_output = your_llm_client.generate(messages=messages)
    return model_output


# 原来：
# model_output = mock_llm_decide(messages)

# 替换为：
# model_output = call_llm(messages)
```

这样做的好处是边界清楚：模型输入系统负责“给 LLM 看什么”，LLM 负责“判断下一步”，解析层负责“把模型输出变成可执行结构”，Runtime 决定“能不能执行”。即使后面更换模型、增加 Tool 或调整权限，也不需要重写整个 ReAct 循环。

## 9. 实践小结：第四章真正落地的部分

实验2让读者看见 Agent 如何一轮轮推进任务；本实验让读者看见，每一轮推进之前，系统如何把模型输入准备好。到这里，读者不需要把所有实现细节背下来，只要把链路看清楚：State 不是直接等于 Prompt，Observation 也不是直接塞回 LLM；系统会先做 Context Assembly，再形成 Messages，让 LLM 基于明确的 Instruction、Context 和输出约束做判断。

| **本实验结论** | **一句话记忆** |
| --- | --- |
| 实验2的 ReAct 骨架没有被推翻。 | 只是把 Decide 前面补上模型输入系统。 |
| Context 不是越多越好。 | 它应该是当前 Step 需要的有效信息。 |
| Messages 是模型输入的承载形式。 | Instruction、Context 和输出约束可以通过 Messages 进入模型。 |
| 真实 LLM 接入点应该隔离。 | 优先替换 mock\_llm\_decide(messages)，不要把 API 调用散落在循环里。 |
| 下一章进入 Tool System。 | 当模型判断需要外部能力时，系统还要解决工具说明、工具调用、执行、权限和失败处理。 |
