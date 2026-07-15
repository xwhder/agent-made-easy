# Lab 3 - Model Input System

*Add model input assembly to the minimal ReAct Agent*

This lab follows Lab 2 and Chapter 4. Lab 2 showed the ReAct task loop: Goal gives direction, State records progress, Action moves the task, Observation updates State, and the Feedback Loop enters the next round.

This lab adds the Chapter 4 layer: before each judgment, how does the system assemble Instruction, Context, Messages, and output constraints so the LLM sees the right information?

> **Reading Note**
>
> Do not try to memorize every line first. Hold the main boundary: Lab 2 answered "how does the Agent loop move a task forward?" This lab answers "what should the LLM see before each round of decision?"

## 1. Where This Lab Extends Lab 2

Lab 2 simplified the LLM decision into:

```python
decide_next_action(state)
```

That was useful because it made the ReAct loop visible. But a real Agent should not simply throw raw State into the model. It should assemble Context first.

| Lab 2 part | New in this lab | Why |
| --- | --- | --- |
| Goal, State, Action, Observation, Feedback Loop | Kept unchanged | These are the task progression skeleton |
| `decide_next_action(state)` | `build_context(state)` + `build_messages(state)` + `mock_llm_decide(messages)` | State is organized into Context, then placed into Messages |
| `fake_search_tool()` | Kept for now | Full Tool System is Chapter 5 |
| `trace` | Records model output, Action, Observation, and State | Shows how model input affects the next step |

## 2. The Added Layer: Model Input System

The minimal ReAct loop was:

```text
State -> Decide -> Action -> Observation -> State Update
```

With a model input system:

```text
State -> Context Assembly -> Messages -> LLM Decide -> Action
```

Four terms matter:

- **Instruction:** rules for model behavior.
- **Context:** information the model should use for this round.
- **Messages:** common input structure for chat-style model APIs.
- **Output Constraint:** required output format so the runtime can continue.

> **Hold This First**
>
> Prompt is not just a sentence typed by the user. In an Agent, one prompt is often assembled by the system. Instruction controls behavior, Context gives evidence, Messages carry the input, and output constraints make the result executable.

## 3. A Runnable Minimal Version

Save the following as `minimal_context_agent.py` and run:

```powershell
python minimal_context_agent.py
```

**Code 4P-1: `minimal_context_agent.py`**

```python
# minimal_context_agent.py
# Run with: python minimal_context_agent.py
# This example uses only the Python standard library.

import json

SYSTEM_INSTRUCTION = """
You are the decision module of a research Agent.
Your job is not to write the final report directly.
Based on the current Context, decide the next Action.
Output JSON only. Do not include extra explanation.
"""

OUTPUT_INSTRUCTION = """
If information is still missing, output:
{"type": "tool_call", "tool_name": "fake_search_tool", "tool_input": "...", "reason": "..."}

If the information is enough, output:
{"type": "final_answer", "reason": "..."}
"""


def fake_search_tool(topic):
    data = {
        "sales changes": "Sales fluctuated over the last three months.",
        "policy changes": "Some regions adjusted subsidy and trade-in policies.",
        "price changes": "Several automakers changed model prices.",
        "battery cost changes": "Battery raw material prices affected margins.",
    }
    return {"topic": topic, "result": data.get(topic, "No related information found.")}


def build_initial_state(goal):
    return {
        "goal": goal,
        "done": [],
        "missing": ["sales changes", "policy changes", "price changes", "battery cost changes"],
        "observations": [],
        "final_answer": None,
    }


def build_context(state):
    observation_lines = [
        f"- {item['topic']}: {item['result']}" for item in state["observations"]
    ]
    return f"""
Goal: {state['goal']}
Done: {", ".join(state["done"]) if state["done"] else "none"}
Missing: {", ".join(state["missing"]) if state["missing"] else "none"}
Observations:
{chr(10).join(observation_lines) if observation_lines else "none yet"}
""".strip()


def build_messages(state):
    context = build_context(state)
    return [
        {"role": "system", "content": SYSTEM_INSTRUCTION.strip()},
        {
            "role": "user",
            "content": (
                "Please decide the next Action from the Context below.\n\n"
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
    """Simulate an LLM reading Messages and outputting an Action JSON."""
    user_content = messages[-1]["content"]
    missing_text = extract_context_field(user_content, "Missing")
    missing = [item.strip() for item in missing_text.split(",") if item.strip() and item != "none"]

    if missing:
        topic = missing[0]
        return json.dumps(
            {
                "type": "tool_call",
                "tool_name": "fake_search_tool",
                "tool_input": topic,
                "reason": f"The current Context lacks {topic}.",
            },
            ensure_ascii=False,
        )

    return json.dumps(
        {"type": "final_answer", "reason": "The key information is available."},
        ensure_ascii=False,
    )


def parse_action(model_output):
    action = json.loads(model_output)
    if action["type"] == "tool_call":
        if action.get("tool_name") != "fake_search_tool":
            raise ValueError("This lab only allows fake_search_tool.")
        if not action.get("tool_input"):
            raise ValueError("tool_call must include tool_input.")
    elif action["type"] != "final_answer":
        raise ValueError(f"Unknown Action type: {action['type']}")
    return action


def execute_action(action):
    if action["type"] == "tool_call":
        return fake_search_tool(action["tool_input"])
    if action["type"] == "final_answer":
        return {"result": "The available Observations are enough for the final analysis."}
    raise ValueError(f"Cannot execute Action: {action}")


def update_state(state, action, observation):
    if action["type"] == "tool_call":
        topic = observation["topic"]
        state["observations"].append(observation)
        if topic in state["missing"]:
            state["missing"].remove(topic)
        if topic not in state["done"]:
            state["done"].append(topic)

    if action["type"] == "final_answer":
        lines = ["New-energy vehicle industry changes:"]
        for item in state["observations"]:
            lines.append(f"- {item['topic']}: {item['result']}")
        state["final_answer"] = "\n".join(lines)

    return state


def run_agent():
    state = build_initial_state("Analyze recent changes in the new-energy vehicle industry.")
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
                "state": {"done": list(state["done"]), "missing": list(state["missing"])},
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

## 4. What the First-Round Messages Look Like

The first round shows what Chapter 4 is about: what does the LLM see?

```text
[system]
You are the decision module of a research Agent...

[user]
Please decide the next Action from the Context below.

Goal: Analyze recent changes...
Done: none
Missing: sales changes, policy changes, price changes, battery cost changes
Observations:
none yet
```

- **system message:** carries Instruction.
- **user message:** carries this round's Context and request.
- **Missing:** comes from State and tells the model what is still needed.
- **Observations:** comes from previous tool results; first round has none.

> **Key Boundary**
>
> Observation does not automatically become Prompt. Observation first updates State. The next Context Assembly step decides what State information enters Messages.

## 5. How Input Affects Action

Because the first Missing item is `sales changes`, the model output proposes a tool call for `sales changes`. After State changes, the next Context changes, and the next Action changes as well.

This is the value of the input system: it makes the model's next judgment depend on the current task state, not on vague memory.

## 6. Concept Mapping

| Code location | Concept | Responsibility |
| --- | --- | --- |
| `SYSTEM_INSTRUCTION` | Instruction | Constrains model role and behavior |
| `OUTPUT_INSTRUCTION` | Output Constraint | Requires executable JSON |
| `build_context(state)` | Context Assembly | Turns State into useful Context |
| `build_messages(state)` | Messages | Builds model input structure |
| `mock_llm_decide(messages)` | LLM Decide simulation | Replace this with a real LLM later |
| `parse_action(model_output)` | Output parsing | Converts model text into executable Action |
| `execute_action(action)` | Action execution | Calls Tool or finalizes |
| `update_state(...)` | State Update | Writes Observation back into State |
| `trace` | Trace | Records each round |

## 7. Why This Is Not Full Tool Calling Yet

The code already contains `tool_call`, but it is not a full Tool System. It lacks:

- Tool Description.
- Formal argument schema.
- Permission boundary.
- Tool registry.
- Tool failure handling.
- Runtime-level validation beyond the simplest checks.

Those are added in Lab 4 and Chapter 5.

## 8. Where to Replace with a Real LLM

Keep the boundaries stable. Replace only the simulated decision layer:

```python
def call_llm(messages):
    """Call your selected LLM SDK or HTTP API here.

    Input remains messages.
    Output must still be parseable as Action JSON.
    """
    model_output = your_llm_client.generate(messages=messages)
    return model_output


# Old:
# model_output = mock_llm_decide(messages)

# New:
# model_output = call_llm(messages)
```

## 9. Lab Summary

Lab 2 showed how an Agent moves a task forward. This lab shows what the model should see before each move.

The core chain is:

```text
State -> Context Assembly -> Messages -> LLM Decide -> Action
```
