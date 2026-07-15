# Lab 4 - Minimal Tool Calling Agent

*Run the Tool System: from Tool Description to Tool Call, Runtime, Observation, and Permission Boundary*

This lab follows Lab 3 and Chapter 5. Lab 3 added Instruction, Context, Messages, and output constraints to the minimal ReAct Agent. This lab keeps the same example and adds a minimal Tool System.

The lab still does not call a real LLM or a real search API. The focus is the tool execution chain: the model sees Tool Descriptions, outputs a Tool Call, Runtime validates the request, executes or rejects the Tool, turns Raw Result into Observation, writes it back to State, and enters the next Context.

> **Reading Note**
>
> Lab 3 answered: what should the LLM see each round? This lab answers: after the LLM says it wants a tool, how does the system turn that request into controlled execution?

## 1. Where This Lab Extends Lab 3

| Lab 3 part | New in this lab | Chapter 5 concept |
| --- | --- | --- |
| `build_messages(state)` | Adds Tool Descriptions to available tools | Tool Description enters model input |
| `mock_llm_decide(messages)` | Outputs a Tool Call with tool name and arguments | Tool Call is a request |
| `execute_action(action)` | Becomes Runtime validation and execution | Runtime parses, validates, executes, or rejects |
| `fake_search_tool()` | Registered in Tool Registry | Tool is system capability |
| Observation update | Separates Raw Result and Observation | Tool result returns to context |
| No permission demo | Adds blocked email example | Permission Boundary and Tool Failure |

## 2. Five Points This Lab Should Make Visible

- **Tool:** external capability exposed by the system.
- **Tool Description:** model-facing description of a Tool.
- **Tool Call:** structured request proposed by the model.
- **Runtime:** execution layer that validates, executes, rejects, and records.
- **Observation:** cleaned result written back into State.

> **Hold This First**
>
> LLM proposes Tool Call. Runtime decides whether it can run. Tool performs the real operation. Observation carries the result back into the task loop.

## 3. A Runnable Minimal Tool Calling Agent

Save the following as `minimal_tool_calling_agent.py` and run:

```powershell
python minimal_tool_calling_agent.py
```

**Code 5P-1: `minimal_tool_calling_agent.py`**

```python
# minimal_tool_calling_agent.py
# Run with: python minimal_tool_calling_agent.py
# Uses only the Python standard library.

import json


def search_industry_news(query, time_range):
    """Low-risk Tool: simulate searching public material."""
    data = {
        "sales changes": "Sales fluctuated over the last three months.",
        "policy changes": "Some regions adjusted subsidy and trade-in policies.",
        "price changes": "Several automakers changed model prices.",
        "battery cost changes": "Battery raw material prices affected margins.",
    }
    for topic, result in data.items():
        if topic in query:
            return {"source": "public_search", "topic": topic, "raw_result": result}
    return {"source": "public_search", "topic": "unknown", "raw_result": ""}


def send_report_email(to, subject, body):
    """High-risk Tool: simulated email sending."""
    return {"sent": True, "to": to, "subject": subject}


TOOLS = {
    "search_industry_news": {
        "description": {
            "name": "search_industry_news",
            "purpose": "Search public material about the new-energy vehicle industry.",
            "arguments": {
                "query": "Question to search, such as: sales changes",
                "time_range": "Time range, such as: last three months",
            },
            "risk_level": "low",
            "returns": "Public information summary and source type.",
        },
        "handler": search_industry_news,
        "required_args": ["query", "time_range"],
        "requires_confirmation": False,
    },
    "send_report_email": {
        "description": {
            "name": "send_report_email",
            "purpose": "Send the final report to a recipient.",
            "arguments": {"to": "email", "subject": "email subject", "body": "email body"},
            "risk_level": "high",
            "returns": "Send result.",
        },
        "handler": send_report_email,
        "required_args": ["to", "subject", "body"],
        "requires_confirmation": True,
    },
}


SYSTEM_INSTRUCTION = """
You are the decision module of a research Agent.
You may request Tool Calls based on Context, but you must not claim the Tool has already run.
Output JSON only.
"""


def build_initial_state(goal):
    return {
        "goal": goal,
        "done": [],
        "missing": ["sales changes", "policy changes", "price changes", "battery cost changes"],
        "observations": [],
        "final_answer": None,
    }


def build_tool_descriptions(tools):
    return json.dumps([tool["description"] for tool in tools.values()], ensure_ascii=False, indent=2)


def build_context(state):
    observations = []
    for item in state["observations"]:
        status = "ok" if item["ok"] else "failed"
        observations.append(f"- [{status}] {item['summary']}")

    return f"""
Goal: {state['goal']}
Done: {", ".join(state["done"]) if state["done"] else "none"}
Missing: {", ".join(state["missing"]) if state["missing"] else "none"}
Observations:
{chr(10).join(observations) if observations else "none yet"}
""".strip()


def build_messages(state, tools):
    return [
        {"role": "system", "content": SYSTEM_INSTRUCTION.strip()},
        {
            "role": "user",
            "content": f"""
Please decide the next Action.

Context:
{build_context(state)}

Available Tools:
{build_tool_descriptions(tools)}

If a tool is needed, output:
{{"type": "tool_call", "tool_name": "...", "arguments": {{...}}, "reason": "..."}}

If enough information is available, output:
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
    user_content = messages[-1]["content"]
    missing_text = extract_context_field(user_content, "Missing")
    missing = [item.strip() for item in missing_text.split(",") if item.strip() and item != "none"]

    if missing:
        topic = missing[0]
        return json.dumps(
            {
                "type": "tool_call",
                "tool_name": "search_industry_news",
                "arguments": {"query": topic, "time_range": "last three months"},
                "reason": f"The Context lacks {topic}.",
            },
            ensure_ascii=False,
        )

    return json.dumps({"type": "final_answer", "reason": "The key information is available."})


def parse_model_output(model_output):
    action = json.loads(model_output)
    if action["type"] not in ["tool_call", "final_answer"]:
        raise ValueError(f"Unknown Action type: {action['type']}")
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
        return {"ok": False, "tool_name": action.get("tool_name"), "error": message}

    tool = tools[action["tool_name"]]
    result = tool["handler"](**action["arguments"])
    return {"ok": True, "tool_name": action["tool_name"], "raw_result": result}


def make_observation(tool_result):
    if not tool_result["ok"]:
        return {"ok": False, "summary": tool_result["error"], "topic": None}

    raw = tool_result["raw_result"]
    if not raw.get("raw_result"):
        return {"ok": False, "summary": "Tool returned empty result.", "topic": None}

    return {
        "ok": True,
        "topic": raw["topic"],
        "summary": f"{raw['topic']}: {raw['raw_result']} source={raw['source']}",
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
        lines = ["New-energy vehicle industry changes:"]
        for item in state["observations"]:
            if item["ok"]:
                lines.append(f"- {item['summary']}")
        state["final_answer"] = "\n".join(lines)

    return state


def run_agent():
    state = build_initial_state("Analyze recent changes in the new-energy vehicle industry.")
    trace = []
    first_round_messages = None

    for round_id in range(1, 8):
        messages = build_messages(state, TOOLS)
        if first_round_messages is None:
            first_round_messages = messages

        action = parse_model_output(mock_llm_decide(messages))

        if action["type"] == "final_answer":
            observation = {"ok": True, "topic": None, "summary": "Enter final output."}
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
                "state": {"done": list(state["done"]), "missing": list(state["missing"])},
            }
        )

    return state, trace, first_round_messages


def permission_boundary_demo():
    unsafe_action = {
        "type": "tool_call",
        "tool_name": "send_report_email",
        "arguments": {
            "to": "client@example.com",
            "subject": "Industry analysis",
            "body": "Report body.",
        },
        "reason": "Try to send report to client.",
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

## 4. What Tool Description Looks Like

Tool Description is written for the model. It is not the tool function itself. It tells the model what tools exist, what arguments they require, and what risks they carry.

In this lab, one tool is low-risk search and one tool is high-risk email sending. This makes the permission boundary visible.

## 5. How Tool Call Passes Through Runtime

The model sees that `sales changes` is missing and that `search_industry_news` is available. It outputs a Tool Call. Runtime then checks:

- Does the tool exist?
- Are required arguments present?
- Is permission sufficient?
- Can the tool be executed?

Only after these checks does the Tool run.

## 6. Tool Failure and Permission Boundary

The email tool requires confirmation. If the model proposes the call without confirmation, Runtime rejects it:

```text
Permission denied: send_report_email requires user confirmation
```

> **Key Boundary**
>
> The model can request. The runtime must enforce safety.

## 7. Concept Mapping

| Code location | Concept | Boundary |
| --- | --- | --- |
| `TOOLS` | Tool Registry | Records tools, descriptions, args, permissions, handlers |
| `description` | Tool Description | Model-facing description |
| `handler` | Tool | Real executable function |
| `mock_llm_decide()` | LLM Decide | Simulates model tool request |
| `parse_model_output()` | Output parsing | Converts model text to Action |
| `validate_tool_call()` | Runtime validation | Checks tool, args, and permissions |
| `run_tool_call()` | Runtime execution | Executes or rejects |
| `make_observation()` | Observation cleanup | Turns raw result into State-friendly information |
| `update_state()` | State Update | Updates done and missing items |

## 8. Lab Summary

This lab turns Tool Calling into an execution chain:

```text
Tool Description -> Tool Call -> Runtime Validation -> Tool Execution -> Observation -> State
```

Tool System is the Agent's execution layer, and Runtime is its safety boundary.
