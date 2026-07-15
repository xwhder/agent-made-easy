# Lab 1 - Minimal Agent Loop

*Transition code after Chapter 2: from one LLM call to a task progression structure*

The first two chapters explained that an Agent is not a normal one-shot LLM call, and it is not merely a model that can call tools. This lab uses a small Python-style pseudocode example to connect Goal, Context, Tool, Observation, State, and Feedback Loop.

This is not a production Agent. It does not need an API key, external network access, or any framework. It has one purpose: to show what the smallest Agent loop looks like and how it differs from a normal LLM call.

> **Reading Note**
>
> If you do not want to read code yet, you can skip this lab and return later. It will not block Chapter 3, but reading it helps build intuition early.

## 1. Lab Goal

This lab answers one question:

> What is the difference between a minimal Agent and a normal LLM call?

A normal LLM call usually takes one Context, receives one Response, and stops. A minimal Agent loop organizes Context around a Goal, lets the LLM decide an Action, executes a Tool if needed, receives an Observation, updates State, and then decides again.

| Comparison item | Normal LLM call | Minimal Agent loop |
| --- | --- | --- |
| Core structure | One input, one output | Multiple rounds around a Goal |
| Tool | Usually no external capability | Can call tools when needed |
| Observation | No explicit action result loop | Tool or action result enters the next round |
| State | Usually no task progress ledger | Records what is done, what is known, and what is missing |
| Stop style | Ends after output | Stop Condition decides whether to end |

## 2. A Normal LLM Call

First, look at a normal LLM call. Here, Context means the information the model can see in this call, and Response means the generated output.

**Pseudocode E1-1: One normal LLM call**

```python
user_input = "Help me analyze the new-energy vehicle industry over the last three months."
context = build_context(user_input)
answer = llm_generate(context)
print(answer)
```

> **Hold This First**
>
> This code describes a one-shot flow: the system turns user input into Context, sends it to the LLM, receives an answer, and stops. There is no Tool, Observation, State, or Feedback Loop yet.

## 3. Minimal Agent Loop

The next pseudocode starts to look like an Agent. Goal is the task target. Tool is an external capability exposed by the system. Observation is the result returned after an Action. State is the current task status. Feedback Loop lets the system judge the next step from the latest result.

> **Hold This First**
>
> `mock_llm_decide` is not a real LLM. It simulates how an LLM might decide the next step from Context. This keeps the focus on structure, not API setup or search integration.

**Pseudocode E1-2: Minimal Agent loop**

```python
state = {
    "goal": "Summarize changes in the new-energy vehicle industry over the last three months",
    "notes": [],
    "step": 0,
    "done": False,
}

MAX_STEPS = 3


def search_tool(query):
    return [
        "Sales data: sales changed noticeably over the last three months.",
        "Policy data: some regions adjusted subsidy policies.",
        "Pricing data: several automakers changed model prices.",
        "Cost data: battery cost changes affected vehicle pricing.",
    ]


def build_context(state):
    return {
        "goal": state["goal"],
        "notes": state["notes"],
        "step": state["step"],
    }


def mock_llm_decide(context):
    if len(context["notes"]) == 0:
        return {
            "type": "tool_call",
            "tool": "search_tool",
            "query": "new-energy vehicle industry changes last three months",
        }

    return {
        "type": "final_answer",
        "content": (
            "The report can be organized around sales, policy, pricing, "
            "and battery cost changes."
        ),
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

> **Hold This First**
>
> The point of this code is not syntax. The point is structure: the Agent organizes Context around a Goal, decides an Action, executes a Tool when needed, receives an Observation, writes it into State, and then decides again.

## 4. How to Read This Code

| Code element | Agent concept | What it solves |
| --- | --- | --- |
| `state` | State | Stores task goal, collected notes, current step, and completion status |
| `goal` | Goal | Gives the Agent a task direction |
| `build_context` | Context assembly | Turns current Goal and State into model-visible information |
| `mock_llm_decide` | Decision layer | Simulates the LLM deciding the next Action |
| `search_tool` | Tool | Provides external information |
| `observation` | Observation | Captures the result of the Tool |
| `while` loop | Feedback Loop | Lets the system continue until done |
| `MAX_STEPS` | Stop Condition | Prevents endless execution |

## 5. This Is Not a Complete Agent Yet

This lab deliberately leaves out many production concerns:

- No real LLM API call.
- No tool schema.
- No permission boundary.
- No source citations.
- No error handling.
- No memory system.
- No evaluation.

That is intentional. Before adding those layers, you should first see the smallest loop.

## 6. Lab Summary

The minimal Agent loop is:

```text
Goal -> Context -> Action -> Tool -> Observation -> State -> next round
```

The key difference from a normal LLM call is continuity. A normal call answers once. An Agent loop can keep moving a task forward.
