# 第 5 章章末实践

**把 Tool System 跑起来：
一个最小 Tool Calling Agent**

*从 Tool Description 到 Tool Call、Runtime、Observation 与 Permission Boundary*

这个文档是第五章之后的独立实战，写法参考“实验3：模型输入系统”。实验3已经把 Instruction、Context、Messages 和输出约束放进最小 ReAct Agent 里；本实验继续沿着同一个例子往前走，把第五章的 Tool System 跑起来。

本实验仍然不接真实搜索 API，也不接真实 LLM。原因不是这些不重要，而是本章重点是工具调用链路：LLM 看到 Tool Description 后提出 Tool Call，Agent Runtime 校验这个请求，真正执行 Tool，把 Raw Result 整理成 Observation，再写回 State，进入下一轮 Context。

> **阅读提示**
>
> 先掌握到这里：实验3解决“LLM 每一轮看见什么”，本实验解决“LLM 判断要用工具后，系统怎么把请求变成可控执行”。不要把 Tool Call 理解成工具已经执行，它只是模型提出的调用请求。

## 1. 本实验承接实验3哪里

实验3里，Messages 已经能把 Goal、State、Observation 和输出约束交给 LLM。第五章之后，我们要继续补上 Tool System：让 Messages 里出现 Tool Description，让模型输出 Tool Call，再由 Runtime 决定是否执行。

| **实验3已有部分** | **本实验新增部分** | **对应第五章内容** |
| --- | --- | --- |
| build\_messages(state) | 把 Tool Description 放进 Available Tools | Tool Description 如何进入模型输入 |
| mock\_llm\_decide(messages) | 输出带工具名和参数的 Tool Call | Tool Call 是调用请求，不是执行结果 |
| execute\_action(action) | 升级为 Runtime 校验和执行 Tool Call | Runtime 负责解析、校验、执行或拒绝 |
| fake\_search\_tool() | 注册进 Tool Registry | Tool 是系统开放给 Agent 的外部能力接口 |
| Observation 更新 State | 区分 Raw Result 和 Observation | 工具结果如何回到下一轮 Context |
| 无权限演示 | 加入 send\_report\_email 的拒绝示例 | Permission Boundary 和 Tool Failure |

## 2. 本实验要突出第五章的五个点

第五章的重点不是“多写几个工具函数”，而是建立工具系统的边界。下面五个点，是本实验代码要让读者真正看见的部分。

- **Tool：** Agent 系统可以调用的外部能力接口。本实验里有 search\_industry\_news 和 send\_report\_email。

- **Tool Description：** 写给 LLM 看的工具说明，让模型知道工具能做什么、需要哪些参数、风险等级是什么。

- **Tool Call：** LLM 提出的调用请求，包含工具名、参数和调用理由。它不是工具已经执行。

- **Runtime：** Agent 的运行时执行层，负责解析、校验、执行或拒绝 Tool Call，并记录 Trace。

- **Observation：** 工具 Raw Result 被系统整理后的结果，会写回 State，影响下一轮 Context Assembly。

> **先掌握到这里**
>
> 读者现在不用掌握生产级工具框架。先把边界看清楚：LLM 负责提出 Tool Call，Runtime 负责决定能不能执行，Tool 负责真实外部能力，Observation 负责把结果带回任务循环。

## 3. 一个可运行的最小 Tool Calling Agent

下面代码可以直接运行，只使用 Python 标准库。它保留实验3的 Context 和 Messages 结构，但加入 Tool Registry、Tool Description、Tool Call 校验、Permission Boundary 和 Observation 整理。

这里的 mock\_llm\_decide(messages) 仍然用规则模拟 LLM。真实接入 LLM 时，优先替换这一层，而不是把 API 调用散落在 Runtime、Tool 或 State 更新逻辑里。

**代码 5P-1：minimal\_tool\_calling\_agent.py**

```python
# minimal_tool_calling_agent.py
# 运行方式：保存为 minimal_tool_calling_agent.py 后执行 python minimal_tool_calling_agent.py
# 这个示例只用 Python 标准库，重点是展示 Tool System 的执行链路。

import json


def search_industry_news(query, time_range):
    """低风险 Tool：模拟搜索公开资料。"""
    data = {
        "销量变化": "最近三个月新能源汽车销量出现波动，部分月份环比增长放缓。",
        "政策变化": "部分地区调整了购车补贴、以旧换新和牌照支持政策。",
        "价格变化": "多家车企调整车型价格，部分品牌加大优惠力度。",
        "电池成本变化": "电池原材料价格变化影响整车定价和毛利空间。",
    }
    for topic, result in data.items():
        if topic in query:
            return {
                "source": "public_search",
                "topic": topic,
                "raw_result": result,
            }
    return {
        "source": "public_search",
        "topic": "unknown",
        "raw_result": "",
    }


def send_report_email(to, subject, body):
    """高风险 Tool：模拟发送邮件。这个函数不会在没有确认时执行。"""
    return {
        "sent": True,
        "to": to,
        "subject": subject,
    }


TOOLS = {
    "search_industry_news": {
        "description": {
            "name": "search_industry_news",
            "purpose": "搜索新能源汽车行业最近三个月的公开资料。",
            "arguments": {
                "query": "要搜索的问题，例如：新能源汽车 最近三个月 销量变化",
                "time_range": "时间范围，例如：最近三个月",
            },
            "risk_level": "low",
            "returns": "公开资料摘要和来源类型。",
        },
        "handler": search_industry_news,
        "required_args": ["query", "time_range"],
        "requires_confirmation": False,
    },
    "send_report_email": {
        "description": {
            "name": "send_report_email",
            "purpose": "把最终报告发送给指定收件人。",
            "arguments": {
                "to": "收件人邮箱",
                "subject": "邮件标题",
                "body": "邮件正文",
            },
            "risk_level": "high",
            "returns": "发送结果。",
        },
        "handler": send_report_email,
        "required_args": ["to", "subject", "body"],
        "requires_confirmation": True,
    },
}


SYSTEM_INSTRUCTION = """
你是一个资料分析 Agent 的决策模块。
你可以根据 Context 请求调用 Tool，但不能声称自己已经执行了 Tool。
你只能输出 JSON，不要输出额外解释。
"""


def build_initial_state(goal):
    return {
        "goal": goal,
        "done": [],
        "missing": ["销量变化", "政策变化", "价格变化", "电池成本变化"],
        "observations": [],
        "final_answer": None,
    }


def build_tool_descriptions(tools):
    descriptions = []
    for tool in tools.values():
        descriptions.append(tool["description"])
    return json.dumps(descriptions, ensure_ascii=False, indent=2)


def build_context(state):
    observations = []
    for item in state["observations"]:
        status = "ok" if item["ok"] else "failed"
        observations.append(f"- [{status}] {item['summary']}")

    return f"""
Goal: {state['goal']}
Done: {"、".join(state["done"]) if state["done"] else "无"}
Missing: {"、".join(state["missing"]) if state["missing"] else "无"}

Observations:
{chr(10).join(observations) if observations else "暂无"}
""".strip()


def build_messages(state, tools):
    tool_descriptions = build_tool_descriptions(tools)
    context = build_context(state)
    return [
        {"role": "system", "content": SYSTEM_INSTRUCTION.strip()},
        {
            "role": "user",
            "content": f"""
请根据 Context 判断下一步 Action。

Context:
{context}

Available Tools:
{tool_descriptions}

如果需要调用工具，只能输出：
{{"type": "tool_call", "tool_name": "...", "arguments": {{...}}, "reason": "..."}}

如果资料已经足够，只能输出：
{{"type": "final_answer", "reason": "..."}}
""".strip(),
        },
    ]


def extract_context_field(text, field_name):
    prefix = field_name + ":"
    for line in text.splitlines():
        if line.startswith(prefix):
            return line.split(":", 1)[1].strip()
    return ""


def mock_llm_decide(messages):
    """
    模拟 LLM 根据 Messages 输出 Tool Call。
    真实接入时，把这个函数替换成 call_llm(messages)。
    """
    user_content = messages[-1]["content"]
    missing_text = extract_context_field(user_content, "Missing")
    missing = [item.strip() for item in missing_text.split("、") if item.strip() and item != "无"]

    if missing:
        topic = missing[0]
        return json.dumps(
            {
                "type": "tool_call",
                "tool_name": "search_industry_news",
                "arguments": {
                    "query": f"新能源汽车 最近三个月 {topic}",
                    "time_range": "最近三个月",
                },
                "reason": f"当前 Context 缺少{topic}，需要调用公开资料搜索工具。",
            },
            ensure_ascii=False,
        )

    return json.dumps(
        {
            "type": "final_answer",
            "reason": "关键资料已经补齐，可以生成最终分析。",
        },
        ensure_ascii=False,
    )


def parse_model_output(model_output):
    action = json.loads(model_output)
    if action["type"] not in ["tool_call", "final_answer"]:
        raise ValueError(f"未知 Action 类型: {action['type']}")
    return action


def validate_tool_call(action, tools):
    tool_name = action.get("tool_name")
    arguments = action.get("arguments", {})

    if tool_name not in tools:
        return False, f"Tool not found: {tool_name}"

    tool = tools[tool_name]
    missing_args = [arg for arg in tool["required_args"] if arg not in arguments]
    if missing_args:
        return False, f"Missing arguments: {missing_args}"

    if tool["requires_confirmation"] and not action.get("confirmed_by_user"):
        return False, f"Permission denied: {tool_name} requires user confirmation"

    return True, "ok"


def run_tool_call(action, tools):
    ok, message = validate_tool_call(action, tools)
    if not ok:
        return {
            "ok": False,
            "tool_name": action.get("tool_name"),
            "error": message,
        }

    tool = tools[action["tool_name"]]
    result = tool["handler"](**action["arguments"])
    return {
        "ok": True,
        "tool_name": action["tool_name"],
        "raw_result": result,
    }


def make_observation(tool_result):
    if not tool_result["ok"]:
        return {
            "ok": False,
            "summary": tool_result["error"],
            "topic": None,
        }

    raw = tool_result["raw_result"]
    if not raw.get("raw_result"):
        return {
            "ok": False,
            "summary": "工具返回为空，需要调整查询或换工具。",
            "topic": None,
        }

    return {
        "ok": True,
        "topic": raw["topic"],
        "summary": f"{raw['topic']}: {raw['raw_result']} 来源={raw['source']}",
    }


def update_state(state, action, observation):
    state["observations"].append(observation)
    if action["type"] == "tool_call" and observation["ok"]:
        topic = observation["topic"]
        if topic in state["missing"]:
            state["missing"].remove(topic)
        if topic not in state["done"]:
            state["done"].append(topic)

    if action["type"] == "final_answer":
        lines = ["新能源汽车行业最近三个月变化分析："]
        for item in state["observations"]:
            if item["ok"]:
                lines.append(f"- {item['summary']}")
        state["final_answer"] = "\n".join(lines)

    return state


def run_agent():
    state = build_initial_state("分析新能源汽车行业最近三个月的变化。")
    trace = []
    first_round_messages = None

    for round_id in range(1, 8):
        messages = build_messages(state, TOOLS)
        if first_round_messages is None:
            first_round_messages = messages

        model_output = mock_llm_decide(messages)
        action = parse_model_output(model_output)

        if action["type"] == "final_answer":
            observation = {"ok": True, "topic": None, "summary": "进入最终输出。"}
            state = update_state(state, action, observation)
            trace.append({"round": round_id, "action": action, "observation": observation})
            break

        tool_result = run_tool_call(action, TOOLS)
        observation = make_observation(tool_result)
        state = update_state(state, action, observation)

        trace.append(
            {
                "round": round_id,
                "action": action,
                "tool_result": tool_result,
                "observation": observation,
                "state": {
                    "done": list(state["done"]),
                    "missing": list(state["missing"]),
                },
            }
        )

    return state, trace, first_round_messages


def permission_boundary_demo():
    unsafe_action = {
        "type": "tool_call",
        "tool_name": "send_report_email",
        "arguments": {
            "to": "client@example.com",
            "subject": "新能源汽车行业分析",
            "body": "这里是报告正文。",
        },
        "reason": "尝试把报告发给客户。",
    }
    return run_tool_call(unsafe_action, TOOLS)


if __name__ == "__main__":
    final_state, trace, first_messages = run_agent()

    print("=== First Round Tool Descriptions ===")
    print(build_tool_descriptions(TOOLS))

    print("\n=== Trace ===")
    for item in trace:
        print(f"\nRound {item['round']}")
        print("Action:", item["action"])
        print("Observation:", item["observation"])
        if "state" in item:
            print("State:", item["state"])

    print("\n=== Permission Boundary Demo ===")
    print(permission_boundary_demo())

    print("\n=== Final Answer ===")
    print(final_state["final_answer"])
```

## 4. Tool Description 长什么样

Tool Description 是写给 LLM 看的工具说明，不是工具函数本身。LLM 需要根据这些说明判断是否该调用工具、调用哪个工具、传哪些参数。

**运行输出 5P-2：第一轮 Tool Description**

```text
=== First Round Tool Descriptions ===
[
  {
    "name": "search_industry_news",
    "purpose": "搜索新能源汽车行业最近三个月的公开资料。",
    "arguments": {
      "query": "要搜索的问题，例如：新能源汽车 最近三个月 销量变化",
      "time_range": "时间范围，例如：最近三个月"
    },
    "risk_level": "low",
    "returns": "公开资料摘要和来源类型。"
  },
  {
    "name": "send_report_email",
    "purpose": "把最终报告发送给指定收件人。",
    "arguments": {
      "to": "收件人邮箱",
      "subject": "邮件标题",
      "body": "邮件正文"
    },
    "risk_level": "high",
    "returns": "发送结果。"
  }
]
```

这里有意放了两个 Tool：一个低风险搜索工具，一个高风险发邮件工具。这样读者能看到 Tool System 不只是“能调用”，还要知道工具风险和权限边界。

## 5. Tool Call 如何经过 Runtime

模型看到当前 Context 里缺少“销量变化”，又看到 Available Tools 里有 search\_industry\_news，于是输出一个 Tool Call。注意，这一步只是请求。真正执行前，Runtime 会检查工具是否存在、参数是否齐全、权限是否允许。

**运行输出 5P-3：Trace 摘要**

```text
=== Trace ===

Round 1
Action: {'type': 'tool_call', 'tool_name': 'search_industry_news', 'arguments': {'query': '新能源汽车 最近三个月 销量变化', 'time_range': '最近三个月'}, 'reason': '当前 Context 缺少销量变化，需要调用公开资料搜索工具。'}
Observation: {'ok': True, 'topic': '销量变化', 'summary': '销量变化: 最近三个月新能源汽车销量出现波动，部分月份环比增长放缓。 来源=public_search'}
State: {'done': ['销量变化'], 'missing': ['政策变化', '价格变化', '电池成本变化']}

Round 2
Action: {'type': 'tool_call', 'tool_name': 'search_industry_news', 'arguments': {'query': '新能源汽车 最近三个月 政策变化', 'time_range': '最近三个月'}, 'reason': '当前 Context 缺少政策变化，需要调用公开资料搜索工具。'}
State: {'done': ['销量变化', '政策变化'], 'missing': ['价格变化', '电池成本变化']}

Round 3
Action: {'type': 'tool_call', 'tool_name': 'search_industry_news', 'arguments': {'query': '新能源汽车 最近三个月 价格变化', 'time_range': '最近三个月'}, 'reason': '当前 Context 缺少价格变化，需要调用公开资料搜索工具。'}
State: {'done': ['销量变化', '政策变化', '价格变化'], 'missing': ['电池成本变化']}

Round 4
Action: {'type': 'tool_call', 'tool_name': 'search_industry_news', 'arguments': {'query': '新能源汽车 最近三个月 电池成本变化', 'time_range': '最近三个月'}, 'reason': '当前 Context 缺少电池成本变化，需要调用公开资料搜索工具。'}
State: {'done': ['销量变化', '政策变化', '价格变化', '电池成本变化'], 'missing': []}

Round 5
Action: {'type': 'final_answer', 'reason': '关键资料已经补齐，可以生成最终分析。'}
```

这个 Trace 展示了第五章的完整链路：Tool Description 进入 Messages，模型输出 Tool Call，Runtime 校验并执行 Tool，Tool 返回 Raw Result，系统整理成 Observation，Observation 更新 State，下一轮 Context 再变化。

## 6. Tool Failure 与 Permission Boundary

一个可靠 Agent 不能假设工具永远成功，也不能让模型想做什么系统就做什么。本实验里，send\_report\_email 被标记为高风险 Tool，必须经过用户确认。模型即使提出发送邮件的 Tool Call，Runtime 也应该拦截。

**运行输出 5P-4：权限边界演示**

```text
=== Permission Boundary Demo ===
{
  "ok": false,
  "tool_name": "send_report_email",
  "error": "Permission denied: send_report_email requires user confirmation"
}
```

- **Tool Failure：** 工具不存在、参数错误、返回为空、网络超时、权限不足，都可以形成失败 Observation。

- **Permission Boundary：** 权限边界规定哪些 Tool 可以自动执行，哪些必须确认，哪些永远不能执行。

- **Human-in-the-loop：** 人在关键节点参与确认或接管。本实验没有真正弹出确认，只用 Runtime 拒绝演示边界。

> **关键边界**
>
> Tool Description 可以提醒模型什么时候不要调用某个工具，但不能只靠模型自觉。真正的安全控制必须由 Runtime 执行。即使模型输出了 Tool Call，只要权限不足、参数不合法或风险过高，Runtime 都应该拒绝。

## 7. 代码里的概念对应关系

下面这张表是本实验的阅读地图。读者不需要一次背下所有函数，但应该能看出第五章概念在代码里各自落在哪里。

| **代码位置** | **对应概念** | **职责边界** |
| --- | --- | --- |
| TOOLS | Tool Registry | 记录系统允许使用哪些 Tool，以及描述、参数、权限和执行函数。 |
| description | Tool Description | 写给 LLM 看的工具说明，不是工具本身。 |
| handler | Tool | 真正执行外部能力的函数，由 Runtime 调用。 |
| mock\_llm\_decide() | LLM Decide | 模拟模型根据 Messages 和 Tool Description 输出 Tool Call。 |
| parse\_model\_output() | 输出解析 | 把模型文本解析成系统可处理的 Action。 |
| validate\_tool\_call() | Runtime 校验 | 检查工具是否存在、参数是否齐全、权限是否允许。 |
| run\_tool\_call() | Runtime 执行层 | 校验通过后执行 Tool；校验失败则返回失败结果。 |
| make\_observation() | Observation 整理 | 把 Raw Result 或失败信息整理成可写回 State 的 Observation。 |
| update\_state() | State Update | 让 Observation 改变 done、missing 和最终答案。 |

> **插图 5P-1：最小 Tool Calling Agent 的执行链路**
>
> 图中要表达什么：展示第五章的核心链路：模型不是直接执行工具，而是输出 Tool Call，再由 Runtime 校验、执行、整理结果并回写状态。
>
> 图中包含哪些元素：左侧是 Context Assembly 和 Messages，其中包含 Goal、State、Observation 和 Tool Description。中间是 LLM Decide，输出 Tool Call。Tool Call 进入 Agent Runtime，Runtime 做 Tool Exists、Schema Check、Permission Check。通过后执行 Tool，得到 Raw Result，再整理成 Observation，更新 State，并进入下一轮 Context。Runtime 旁边有拒绝分支，标注 Tool Failure 和 Permission Boundary。底部有 Trace 记录每一轮 Tool Call、Runtime Decision、Observation 和 State Change。
>
> 生图提示：一张简洁专业的 Agent Tool Calling 流程图，浅色背景，技术书籍风格。左侧为 Context Assembly 和 Messages，Messages 中标注 Tool Description。中间为 LLM Decide，输出 Tool Call。Tool Call 指向 Agent Runtime，Runtime 内包含 Tool Exists、Schema Check、Permission Check 三个检查节点。检查通过后连接 Tool，Tool 返回 Raw Result，Raw Result 变成 Observation，Observation 更新 State，再回到下一轮 Context Assembly。Runtime 另有拒绝分支，标注 Tool Failure 和 Permission Boundary。底部放置 Trace 记录条。不使用卡通机器人，不使用复杂科技背景，不使用夸张渐变。

## 8. 实践小结：第五章真正落地的部分

实验3让读者看到模型输入系统如何给 LLM 组织信息；本实验让读者看到，当 LLM 判断需要外部能力时，系统如何把这个判断变成受控执行。到这里，读者只需要把一条链路看清楚：Tool Description 让 LLM 知道工具，Tool Call 表达调用请求，Runtime 校验并执行 Tool，Raw Result 被整理成 Observation，Observation 更新 State，并影响下一轮 Context。

| **本实验结论** | **一句话记忆** |
| --- | --- |
| Tool 不等于 Tool Description | Tool 是真实能力，Tool Description 是写给 LLM 看的说明书。 |
| Tool Call 不等于工具已执行 | Tool Call 只是模型提出的调用请求。 |
| Runtime 是执行边界 | Runtime 负责校验、执行、拒绝、记录和失败处理。 |
| Observation 不等于 Raw Result | Observation 是系统整理后的结果，会进入下一轮 Context。 |
| 权限不能只靠模型自觉 | Permission Boundary 必须由系统层强制执行。 |

下一步如果继续写第六章，就可以把实验3和本实验合并成更完整的 Tool Calling Agent：真实 LLM 读取 Messages，基于 Tool Description 输出 Tool Call，再由 Runtime 执行 Tool，并把 Observation 放回下一轮 Context。
