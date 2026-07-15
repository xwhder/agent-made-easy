# 第 3 章章末实践

**把 ReAct 跑起来：一个最小可运行 Agent**

*用 Python 标准库看清 Goal、Action、Observation、State、Feedback Loop 与 Stop Condition*

本实践故意不接真实 LLM，也不接真实搜索 API。原因不是这些不重要，而是第三章的重点是任务推进结构。我们先用规则函数模拟 LLM 的下一步判断，用本地函数模拟 Tool，等第四章再把判断部分替换成真实 LLM，等第五章再接入真实 Tool System。

> **先掌握到这里**
>
> 这个例子的目标不是做出生产级 Agent，而是一条最小闭环：Goal 给出方向，State 记录进度，Decide 选择下一步 Action，Action 得到 Observation，Observation 更新 State，Feedback Loop 进入下一轮，Stop Condition 决定何时结束。

## 1. 这个例子要证明什么

第三章已经介绍了 Goal、Plan、Step、Action、Observation、State、Feedback Loop、Stop Condition 和 ReAct。如果这些概念只停留在文字层面，容易觉得它们是一堆名词。这个实践的作用，是用一个很小的程序把它们串起来。

- **Goal：** 任务目标，回答最终要完成什么。这里的 Goal 是分析新能源汽车行业最近三个月的变化。

- **State：** 任务当前状态，记录已经查到什么、还缺什么、是否已经形成最终答案。

- **Action：** 当前这一轮要执行的动作。它可以是调用工具，也可以是输出最终结果。

- **Observation：** Action 或 Tool 返回的结果，会被写回 State。

- **Feedback Loop：** 反馈循环，指系统根据新的 State 再次判断下一步。

- **Stop Condition：** 停止条件，指系统判断任务是否应该结束、暂停或交给人处理。

## 2. 一个最小可运行的 ReAct Agent

下面这段代码可以直接运行，只依赖 Python 标准库。为了让代码聚焦在 ReAct 骨架上，示例里没有真实 LLM 调用，也没有真实搜索接口。\`decide\_next\_action()\` 用简单规则模拟 LLM 的判断；\`fake\_search\_tool()\` 用本地数据模拟外部 Tool。

**代码 3P-1：minimal\_react\_agent.py**

```python
# minimal_react_agent.py
# 运行方式：保存为 minimal_react_agent.py 后执行 python minimal_react_agent.py

def fake_search_tool(topic):
    """模拟一个搜索工具。真实系统里，这里可以替换成搜索 API、数据库查询或文件读取。"""
    data = {
        "销量变化": "最近三个月新能源汽车销量出现波动，部分月份环比增长放缓。",
        "政策变化": "部分地区调整了购车补贴、以旧换新和牌照支持政策。",
        "价格变化": "多家车企调整车型价格，部分品牌加大优惠力度。",
        "电池成本变化": "电池原材料价格变化影响整车定价和毛利空间。",
    }
    return data.get(topic, "没有找到相关信息。")


def build_initial_state(goal):
    return {
        "goal": goal,
        "done": [],
        "missing": ["销量变化", "政策变化", "价格变化", "电池成本变化"],
        "observations": [],
        "final_answer": None,
    }


def decide_next_action(state):
    """根据当前 State 判断下一步 Action。

    在真实 Agent 中，这一步通常由 LLM 完成。
    这里先用简单规则模拟，是为了看清 ReAct 的任务推进结构。
    """
    if state["missing"]:
        next_topic = state["missing"][0]
        return {
            "type": "tool_call",
            "tool_name": "fake_search_tool",
            "tool_input": next_topic,
        }

    return {"type": "final_answer"}


def execute_action(action):
    if action["type"] == "tool_call":
        topic = action["tool_input"]
        result = fake_search_tool(topic)
        return {"topic": topic, "result": result}

    if action["type"] == "final_answer":
        return {"result": "可以基于已有 Observation 生成最终分析。"}

    return {"result": "未知 Action。"}


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
            lines.append(f"- {item['topic']}：{item['result']}")
        state["final_answer"] = "\n".join(lines)

    return state


def run_agent():
    goal = "分析新能源汽车行业最近三个月的变化。"
    state = build_initial_state(goal)
    trace = []
    round_id = 1

    while True:
        action = decide_next_action(state)
        observation = execute_action(action)
        state = update_state(state, action, observation)

        trace.append({
            "round": round_id,
            "action": action,
            "observation": observation,
            "state": {
                "done": list(state["done"]),
                "missing": list(state["missing"]),
            },
        })

        if action["type"] == "final_answer":
            break

        if not state["missing"]:
            final_action = {"type": "final_answer"}
            final_observation = execute_action(final_action)
            state = update_state(state, final_action, final_observation)

            trace.append({
                "round": round_id + 1,
                "action": final_action,
                "observation": final_observation,
                "state": {
                    "done": list(state["done"]),
                    "missing": list(state["missing"]),
                },
            })
            break

        round_id += 1

    return state, trace


if __name__ == "__main__":
    final_state, trace = run_agent()

    print("=== Trace ===")
    for item in trace:
        print(f"\nRound {item['round']}")
        print("Action:", item["action"])
        print("Observation:", item["observation"])
        print("State:", item["state"])

    print("\n=== Final Answer ===")
    print(final_state["final_answer"])
```

> **为什么这里不用真实 LLM**
>
> 真实 LLM 接入会引入 Instruction、Prompt、Context、Messages、模型输出解析、API Key 和错误处理，这些是第四章的重点。
>
> 真实 Tool Calling 会引入 Tool Description、Tool Call、参数校验、权限边界和 Observation 整理，这些是第五章的重点。
>
> 本节先把骨架跑通，后面的章节再逐步把模拟部分替换成真实系统能力。

## 3. 从 Trace 看每一轮如何推进

Trace 是运行轨迹，记录 Agent 每一轮做了什么。这个词在 Agent 系统里很重要，因为它让开发者可以复盘：模型为什么选择这个 Action，工具返回了什么，State 如何变化，任务为什么结束或继续。

**运行输出 3P-2：Trace 示例**

```text
=== Trace ===

Round 1
Action: {'type': 'tool_call', 'tool_name': 'fake_search_tool', 'tool_input': '销量变化'}
Observation: {'topic': '销量变化', 'result': '最近三个月新能源汽车销量出现波动，部分月份环比增长放缓。'}
State: {'done': ['销量变化'], 'missing': ['政策变化', '价格变化', '电池成本变化']}

Round 2
Action: {'type': 'tool_call', 'tool_name': 'fake_search_tool', 'tool_input': '政策变化'}
Observation: {'topic': '政策变化', 'result': '部分地区调整了购车补贴、以旧换新和牌照支持政策。'}
State: {'done': ['销量变化', '政策变化'], 'missing': ['价格变化', '电池成本变化']}

Round 3
Action: {'type': 'tool_call', 'tool_name': 'fake_search_tool', 'tool_input': '价格变化'}
Observation: {'topic': '价格变化', 'result': '多家车企调整车型价格，部分品牌加大优惠力度。'}
State: {'done': ['销量变化', '政策变化', '价格变化'], 'missing': ['电池成本变化']}

Round 4
Action: {'type': 'tool_call', 'tool_name': 'fake_search_tool', 'tool_input': '电池成本变化'}
Observation: {'topic': '电池成本变化', 'result': '电池原材料价格变化影响整车定价和毛利空间。'}
State: {'done': ['销量变化', '政策变化', '价格变化', '电池成本变化'], 'missing': []}

Round 5
Action: {'type': 'final_answer'}
Observation: {'result': '可以基于已有 Observation 生成最终分析。'}
State: {'done': ['销量变化', '政策变化', '价格变化', '电池成本变化'], 'missing': []}
```

从 Trace 可以看出，Agent 不是一次性生成答案。它先查销量变化，再查政策变化，再查价格变化，再查电池成本变化。每一轮的 Observation 都会更新 State，State 中的 missing 列表不断减少，直到缺失信息为空，再进入最终输出。

## 4. 代码与概念对照

下面这张表是本实践最重要的阅读方式。不要把注意力放在代码长不长，而是看每个函数在 Agent 任务推进里承担什么角色。

| **代码里的部分** | **对应概念** | **作用** |
| --- | --- | --- |
| goal | Goal | 定义任务最终要完成什么。 |
| state | State | 记录已完成、缺失信息、观察结果和最终答案。 |
| decide\_next\_action() | Decide | 判断下一步做什么。真实系统里通常由 LLM 完成。 |
| action | Action | 当前轮要执行的动作，例如调用工具或输出最终答案。 |
| fake\_search\_tool() | Tool | 模拟外部工具能力。真实系统里可替换成搜索、数据库、文件读取等。 |
| observation | Observation | 工具或动作返回的结果。 |
| update\_state() | State Update | 把 Observation 写回 State。 |
| while / run\_agent() | Feedback Loop | 根据新状态进入下一轮判断。 |
| missing 为空后 final\_answer | Stop Condition / Final Output | 缺失信息补齐后结束任务并生成结果。 |
| trace | Trace | 记录每一轮执行过程，方便调试、审计和复盘。 |

## 5. 这个例子还不是完整 Agent

技术读者看到这里可能会问：如果 \`decide\_next\_action()\` 只是规则判断，那它算不算 Agent？这个问题很好。严格说，这个例子不是生产级 Agent，也没有展示 LLM 的真实推理能力。它是一个教学骨架，目的是把第三章的任务推进结构先跑起来。

- **\`decide\_next\_action()\`：** 后面可以替换成真实 LLM 调用，让模型根据 Instruction、Context 和 Messages 判断下一步。

- **\`fake\_search\_tool()\`：** 后面可以替换成真实 Tool System，例如搜索、文件读取、数据库查询或业务 API。

- **\`state\`：** 后面可以扩展成更完整的 State / Memory System，区分当前任务状态和长期记忆。

- **\`trace\`：** 后面可以扩展成可观测性系统，用于调试、审计、失败复盘和效果评估。

- **\`final\_answer\`：** 后面可以加入输出格式约束、引用检查、质量评估和 Human-in-the-loop。

> **先掌握到这里**
>
> 不要把这个例子理解成“Agent 就是几个 if else”。这里用规则模拟 Decide，是为了把任务推进骨架讲清楚。真正的 Agent 会把 Decide 交给 LLM，把 Tool 替换成真实外部能力，并加入权限、错误处理、上下文组装和观测记录。

## 6. 它如何连接后面的章节

这个小程序可以作为后续章节的连续改造对象。看到同一个 Agent 如何一步步变完整。

| **后续章节** | **替换或增强的位置** | **读者要看到的变化** |
| --- | --- | --- |
| 第 4 章：模型输入系统 | 把 \`decide\_next\_action()\` 替换成真实 LLM 调用。 | LLM 根据 Instruction、Context 和 Messages 判断下一步 Action。 |
| 第 5 章：Tool System | 把 \`fake\_search\_tool()\` 替换成真实工具注册和调用链路。 | LLM 看到 Tool Description，输出 Tool Call，Runtime 执行 Tool，结果变成 Observation。 |
| 第 6 章：Memory System | 扩展 \`state\`，区分短期任务状态和长期记忆。 | Agent 不再每轮都像重新开始，而是能保留进度和偏好。 |
| 后续工程化章节 | 增强 \`trace\`、Stop Condition、权限边界和评估。 | Agent 从能跑的 demo 逐渐变成可调试、可控、可评估的系统。 |

## 7. 实践小结

第三章的核心不是让读者马上搭出完整 Agent，而是让读者看懂任务如何一轮轮推进。这个独立实践把 ReAct 的骨架落到代码里：先判断下一步，再执行动作，拿到结果，更新状态，然后决定继续还是停止。

到这里，读者只需要掌握一个边界：这个例子展示的是 Agent 的任务推进结构，不是完整生产系统。下一步进入第四章时，可以把 \`decide\_next\_action()\` 替换成真实 LLM；进入第五章时，可以把 \`fake\_search\_tool()\` 替换成真实 Tool System。这样，Agent 会从一个可理解的骨架，逐步变成一个可工作的系统。
