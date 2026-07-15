# Lab 2 - Minimal ReAct Agent

*Run ReAct with only the Python standard library*

This lab corresponds to the end of Chapter 3. It does not connect to a real LLM or a real search API. Instead, it uses a rule function to simulate the LLM's next-step decision, and a local function to simulate a Tool.

The goal is to see the task progression structure: Goal gives direction, State records progress, Decide chooses the next Action, Action returns an Observation, Observation updates State, the Feedback Loop enters the next round, and the Stop Condition decides when to end.

## 1. What This Example Proves

If the concepts in Chapter 3 remain only words, they can feel abstract. This lab connects them through a small runnable program.

- **Goal:** what the task is trying to complete.
- **State:** what has been done, what is missing, and what has been observed.
- **Action:** what the current round will do.
- **Observation:** what the action or tool returns.
- **Feedback Loop:** the system judges again from new State.
- **Stop Condition:** the system knows when to finish.

## 2. A Minimal Runnable ReAct Agent

Save the following as `minimal_react_agent.py` and run:

```powershell
python minimal_react_agent.py
```

**Code 3P-1: `minimal_react_agent.py`**

```python
# minimal_react_agent.py
# Run with: python minimal_react_agent.py


def fake_search_tool(topic):
    """Simulate a search tool."""
    data = {
        "sales changes": "Sales fluctuated over the last three months, with growth slowing in some months.",
        "policy changes": "Some regions adjusted subsidies, trade-in policies, and license support.",
        "price changes": "Several automakers changed model prices or increased discounts.",
        "battery cost changes": "Battery raw material prices affected vehicle pricing and margins.",
    }
    return data.get(topic, "No related information found.")


def build_initial_state(goal):
    return {
        "goal": goal,
        "done": [],
        "missing": [
            "sales changes",
            "policy changes",
            "price changes",
            "battery cost changes",
        ],
        "observations": [],
        "final_answer": None,
    }


def decide_next_action(state):
    """Decide the next action from current State.

    In a real Agent, this is usually done by an LLM.
    Here we use simple rules so the ReAct structure is visible.
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
        return {"result": "The available Observations are enough to draft the final analysis."}

    return {"result": "Unknown Action."}


def update_state(state, action, observation):
    if action["type"] == "tool_call":
        topic = observation["topic"]
        state["observations"].append(observation)
        if topic in state["missing"]:
            state["missing"].remove(topic)
        if topic not in state["done"]:
            state["done"].append(topic)

    if action["type"] == "final_answer":
        lines = ["New-energy vehicle industry changes over the last three months:"]
        for item in state["observations"]:
            lines.append(f"- {item['topic']}: {item['result']}")
        state["final_answer"] = "\n".join(lines)

    return state


def run_agent():
    goal = "Analyze changes in the new-energy vehicle industry over the last three months."
    state = build_initial_state(goal)
    trace = []
    round_id = 1

    while True:
        action = decide_next_action(state)
        observation = execute_action(action)
        state = update_state(state, action, observation)

        trace.append(
            {
                "round": round_id,
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

        if not state["missing"]:
            final_action = {"type": "final_answer"}
            final_observation = execute_action(final_action)
            state = update_state(state, final_action, final_observation)
            trace.append(
                {
                    "round": round_id + 1,
                    "action": final_action,
                    "observation": final_observation,
                    "state": {
                        "done": list(state["done"]),
                        "missing": list(state["missing"]),
                    },
                }
            )
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

> **Why no real LLM here?**
>
> A real LLM call would introduce instructions, prompts, context assembly, message formatting, model output parsing, API keys, and error handling. Those are the focus of Chapter 4.
>
> Real Tool Calling would introduce Tool Description, Tool Call, argument validation, permissions, and Observation cleanup. Those are the focus of Chapter 5.

## 3. Reading Each Round from the Trace

The Trace records what the Agent did in each round. This matters because it lets developers review why the Agent selected an Action, what the Tool returned, how State changed, and why the task ended.

**Output 3P-2: Trace example**

```text
=== Trace ===

Round 1
Action: {'type': 'tool_call', 'tool_name': 'fake_search_tool', 'tool_input': 'sales changes'}
Observation: {'topic': 'sales changes', 'result': 'Sales fluctuated over the last three months...'}
State: {'done': ['sales changes'], 'missing': ['policy changes', 'price changes', 'battery cost changes']}

...

Round 5
Action: {'type': 'final_answer'}
State: {'done': ['sales changes', 'policy changes', 'price changes', 'battery cost changes'], 'missing': []}
```

The Agent does not answer in one step. It searches sales changes, then policy changes, then price changes, then battery cost changes. Each Observation updates State until no information is missing.

## 4. Code and Concept Mapping

| Code part | Concept | Role |
| --- | --- | --- |
| `goal` | Goal | Defines what the task should complete |
| `state` | State | Records progress, missing items, observations, and final answer |
| `decide_next_action()` | Decide | Chooses the next action; usually done by an LLM in real systems |
| `action` | Action | The current move, such as a tool call or final answer |
| `fake_search_tool()` | Tool | Simulates external capability |
| `observation` | Observation | Result returned by the action or tool |
| `update_state()` | State Update | Writes Observation back into State |
| `while` / `run_agent()` | Feedback Loop | Repeats judgment from updated State |
| `missing` empty | Stop Condition | Ends the task and produces final output |
| `trace` | Trace | Records each round for debugging and review |

## 5. This Example Is Not a Complete Agent

Strictly speaking, this example is not a production Agent. `decide_next_action()` is rule-based. It does not show a real LLM's judgment.

But that is the point. It is a teaching skeleton.

- `decide_next_action()` can later be replaced by a real LLM call.
- `fake_search_tool()` can later be replaced by a real Tool System.
- `state` can be expanded into a richer State and Memory System.
- `trace` can become an observability system.
- `final_answer` can add formatting, citations, evaluation, and human review.

## 6. How It Connects to Later Chapters

| Later chapter | What gets replaced or enhanced | What the reader should see |
| --- | --- | --- |
| Chapter 4: Model Input System | Replace `decide_next_action()` with a real LLM call | LLM judges from Instruction, Context, and Messages |
| Chapter 5: Tool System | Replace `fake_search_tool()` with registered tools | LLM outputs Tool Call; Runtime executes Tool |
| Chapter 6: Memory System | Expand `state` | Agent keeps task progress and reusable information |
| Engineering chapters | Strengthen Trace, Stop Condition, permission, and evaluation | Demo becomes controllable and debuggable |

## 7. Lab Summary

This lab turns Chapter 3's task progression structure into code. The key idea is not "Agent equals a few if statements." The rule function is only a stand-in for the LLM so the loop is visible.

The structure is:

```text
Decide -> Act -> Observe -> Update State -> Decide Again -> Stop
```
