# Lab 5 - API Calls and Memory Agent

*Continue from Tool Calling: use the Responses API to run State, Memory, and Context*

Lab 4 made the Tool System visible: the model sees Tool Descriptions, outputs Tool Calls, and Runtime validates and executes tools. Lab 4 used a mock LLM so the focus could stay on Tool System.

This lab adds real API calls. The API call lets a Python program request generation from an LLM. At the same time, State, Memory, Tool execution, and permissions still belong to the Agent Runtime.

> **Reading Note**
>
> This lab is more detailed because it is the first one to connect to a real API. First understand where API calls sit in the system.

## 1. Why Connect an API Starting Here

Earlier labs avoided real LLM calls so readers would not get distracted by API keys, network errors, model output formats, or cost. Now that Tool System and Memory System have been introduced, we can replace the mock decision layer with a real LLM call.

| Lab 4 | Lab 5 | Why |
| --- | --- | --- |
| `mock_llm_decide(messages)` | `client.responses.create(...)` | Replace rule simulation with real LLM API call |
| Tool Description in Messages | Tool Description passed as `tools` | Use standard Function Calling style |
| Runtime parses JSON Action | Runtime parses `function_call` in `response.output` | Closer to real API output |
| Tool executes locally | Tool still executes locally | Model requests; system executes |
| State records done/missing | State still records task progress | State is not saved by the model |
| No long-term memory | Add `memory_store.json` | Show Memory write, retrieval, and Context entry |

> **Hold This First**
>
> Calling an API does not hand all Agent responsibility to the model. LLM generates and judges. Runtime owns State, Memory, Tool execution, permissions, and errors.

## 2. The Full Chain This Lab Runs

This lab adds Memory System on top of Tool Calling.

- **State:** current task ledger, including goal, done, missing, observations, trace, and final answer.
- **Memory:** reusable information across rounds or tasks.
- **Retrieved Memory:** the small relevant subset selected for this round.
- **Context:** model input material assembled from user input, state, memory, and observations.
- **Tool Call:** function-call request returned by the API.
- **Observation:** cleaned Tool result that enters State.

> **Figure 6P-1: State, Memory, and Tool Calling after API integration**
>
> The diagram should show User Input entering Agent Runtime. Runtime contains State, Memory Store, Memory Retrieval, Context Assembly, and Tool Registry. Context Assembly sends input and tools to the Responses API. The API returns `function_call`. Runtime executes the local Tool, creates Observation, updates State, optionally writes Memory, and sends Tool output back using `previous_response_id`. `previous_response_id` is API response continuation, not long-term Agent Memory.

## 3. Setup: API Key, Model Name, and Dependency

```powershell
# PowerShell
python -m pip install --upgrade openai
$env:OPENAI_API_KEY = "your API key"
$env:OPENAI_MODEL = "gpt-5.5"
python memory_api_agent.py
```

The code reads model name from `OPENAI_MODEL`. If the default model is not available for your account, set it to a model you can use.

> **Security Note**
>
> Do not write API keys into Word files, code files, or screenshots. Use environment variables.

## 4. Minimal API Call: Confirm the Program Can Call the LLM

```python
from openai import OpenAI

client = OpenAI()

response = client.responses.create(
    model="gpt-5.5",
    input="Explain Agent State in one sentence."
)

print(response.output_text)
```

This is not yet an Agent. It only proves the Python program can call the LLM. An Agent still needs Context Assembly, Tool System, State Update, Memory Retrieval, and Memory Write.

## 5. Complete Code: Memory Agent with API

Save the following as `memory_api_agent.py`.

**Code 6P-3A: Imports, config, Memory Store, and State**

```python
# memory_api_agent.py
# Setup:
#   1. python -m pip install --upgrade openai
#   2. Set OPENAI_API_KEY
#   3. If the default model is unavailable, set OPENAI_MODEL

import json
import os
from datetime import datetime, timezone
from pathlib import Path

try:
    from openai import OpenAI
except ImportError as exc:
    raise SystemExit("Run first: python -m pip install --upgrade openai") from exc


MODEL = os.getenv("OPENAI_MODEL", "gpt-5.5")
MEMORY_FILE = Path("memory_store.json")


def now_iso():
    return datetime.now(timezone.utc).isoformat()


def require_api_key():
    if not os.getenv("OPENAI_API_KEY"):
        raise SystemExit("Please set OPENAI_API_KEY first.")


def load_memory():
    if MEMORY_FILE.exists():
        return json.loads(MEMORY_FILE.read_text(encoding="utf-8"))

    return [
        {
            "id": "mem_user_report_style",
            "type": "user_preference",
            "scope": "global",
            "content": "User prefers industry reports to start with conclusions, then evidence, then recommendations.",
            "confidence": "high",
            "source": "seed",
            "updated_at": now_iso(),
        },
        {
            "id": "mem_nev_analysis_frame",
            "type": "procedural_memory",
            "scope": "industry_analysis",
            "content": "New-energy vehicle industry changes are usually analyzed through sales, pricing, policy, and battery cost.",
            "confidence": "high",
            "source": "seed",
            "updated_at": now_iso(),
        },
    ]


def save_memory(memory_store):
    MEMORY_FILE.write_text(json.dumps(memory_store, ensure_ascii=False, indent=2), encoding="utf-8")


def build_initial_state(goal):
    return {
        "goal": goal,
        "done": [],
        "missing": ["sales changes", "price changes", "policy changes", "battery cost changes"],
        "observations": [],
        "trace": [],
        "final_answer": None,
    }
```

**Code 6P-3B: Tool, Tool Description, Memory Retrieval, and State Update**

```python
def search_industry_news(topic, time_range):
    """Lab Tool: use local mock data instead of real search."""
    data = {
        "sales changes": "Sales growth slowed in some months, while leading brands stayed relatively stable.",
        "price changes": "Several automakers adjusted prices or increased discounts.",
        "policy changes": "Some regions updated subsidies, trade-in support, and license policies.",
        "battery cost changes": "Battery material prices changed, affecting pricing and margins.",
    }
    return {
        "topic": topic,
        "time_range": time_range,
        "source": "mock_public_search",
        "content": data.get(topic, "No related material found."),
    }


TOOL_REGISTRY = {"search_industry_news": search_industry_news}

TOOL_DESCRIPTIONS = [
    {
        "type": "function",
        "name": "search_industry_news",
        "description": "Search public material about recent new-energy vehicle industry changes.",
        "parameters": {
            "type": "object",
            "properties": {
                "topic": {"type": "string", "description": "sales changes, price changes, policy changes, or battery cost changes"},
                "time_range": {"type": "string", "description": "for example: last three months"},
            },
            "required": ["topic", "time_range"],
            "additionalProperties": False,
        },
    }
]


def retrieve_memory(user_input, memory_store):
    keywords = ["industry", "report", "new-energy vehicle", "analysis"]
    retrieved = []
    for item in memory_store:
        text = (item["content"] + " " + item.get("scope", "")).lower()
        if any(keyword in text for keyword in keywords):
            retrieved.append(item)
    return retrieved[:3]


def maybe_write_memory_from_user_input(user_input, memory_store):
    written = []
    lowered = user_input.lower()
    if "prefer" in lowered or "first give conclusions" in lowered:
        item = {
            "id": f"mem_user_pref_{len(memory_store) + 1}",
            "type": "user_preference",
            "scope": "global",
            "content": f"User preference inferred from input: {user_input}",
            "confidence": "medium",
            "source": "user_input",
            "updated_at": now_iso(),
        }
        memory_store.append(item)
        written.append(item)
    return written


def make_observation(tool_result):
    return {
        "topic": tool_result["topic"],
        "summary": tool_result["content"],
        "source": tool_result["source"],
    }


def update_state(state, observation):
    state["observations"].append(observation)
    topic = observation["topic"]
    if topic in state["missing"]:
        state["missing"].remove(topic)
    if topic not in state["done"]:
        state["done"].append(topic)
```

**Code 6P-3C: Context Assembly and Responses API Calls**

```python
def build_context(user_input, state, retrieved_memories):
    memory_lines = [
        f"- [{m['type']}/{m['scope']}] {m['content']}" for m in retrieved_memories
    ]
    observation_lines = [
        f"- {item['topic']}: {item['summary']} source={item['source']}"
        for item in state["observations"]
    ]
    return f"""
You are the decision module of a research Agent.
Do not invent industry facts.
If State.missing has items, call search_industry_news for the first missing topic.
If State.missing is empty, write the final English analysis.

User Input:
{user_input}

Goal:
{state["goal"]}

State:
- done: {state["done"]}
- missing: {state["missing"]}

Retrieved Memory:
{chr(10).join(memory_lines) if memory_lines else "- none"}

Recent Observations:
{chr(10).join(observation_lines) if observation_lines else "- none yet"}
""".strip()


def run_tool_call(function_call):
    tool_name = function_call.name
    if tool_name not in TOOL_REGISTRY:
        return {"ok": False, "error": f"Unknown tool: {tool_name}"}

    args = json.loads(function_call.arguments or "{}")
    result = TOOL_REGISTRY[tool_name](**args)
    return {"ok": True, "tool_name": tool_name, "arguments": args, "raw_result": result}


def get_function_calls(response):
    return [item for item in response.output if getattr(item, "type", None) == "function_call"]


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

**Code 6P-3D: Agent Loop, Tool Output Return, and Entrypoint**

```python
def run_agent(user_input):
    require_api_key()
    client = OpenAI()

    memory_store = load_memory()
    written = maybe_write_memory_from_user_input(user_input, memory_store)

    state = build_initial_state("Analyze recent changes in the new-energy vehicle industry.")

    for round_id in range(1, 6):
        retrieved = retrieve_memory(user_input, memory_store)
        context = build_context(user_input, state, retrieved)
        response = call_model_with_context(client, context)
        function_calls = get_function_calls(response)

        state["trace"].append(
            {
                "round": round_id,
                "retrieved_memory_count": len(retrieved),
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
                observation = make_observation(tool_result["raw_result"])
                update_state(state, observation)
                tool_outputs.append(
                    {
                        "type": "function_call_output",
                        "call_id": call.call_id,
                        "output": json.dumps(tool_result["raw_result"], ensure_ascii=False),
                    }
                )
            else:
                tool_outputs.append(
                    {
                        "type": "function_call_output",
                        "call_id": call.call_id,
                        "output": json.dumps(tool_result, ensure_ascii=False),
                    }
                )

        follow_up = send_tool_outputs(client, response.id, tool_outputs)
        state["trace"][-1]["tool_follow_up_response_id"] = follow_up.id

        if not state["missing"]:
            final_context = build_context(user_input, state, retrieve_memory(user_input, memory_store))
            final_response = client.responses.create(model=MODEL, input=final_context)
            state["final_answer"] = final_response.output_text
            break

    save_memory(memory_store)

    print("\n=== Written Memories ===")
    print(json.dumps(written, ensure_ascii=False, indent=2))
    print("\n=== Trace ===")
    print(json.dumps(state["trace"], ensure_ascii=False, indent=2))
    print("\n=== Final State ===")
    print(json.dumps({k: v for k, v in state.items() if k != "trace"}, ensure_ascii=False, indent=2))


if __name__ == "__main__":
    run_agent("Please first give conclusions, then analyze changes in the new-energy vehicle industry over the last three months.")
```

## 6. What API Function Calling Does Here

| Location | What it is | What it is not |
| --- | --- | --- |
| `TOOL_DESCRIPTIONS` | Tool description passed to the model | Not the real function |
| `response.output` function call | Model's request to call a function | Not executed yet |
| `TOOL_REGISTRY` | Local runtime's function registry | Not automatically executed by the model |
| `run_tool_call()` | Runtime execution of local function | Not LLM running code |
| `function_call_output` | Tool result returned to API | Not long-term Memory |

> **Key Boundary**
>
> API returns `function_call` as a suggestion/request. Whether to execute, how to execute, and what to do on failure still belong to Agent Runtime.

## 7. How State, Memory, and Context Map to Code

| Chapter 6 concept | Code location | Meaning |
| --- | --- | --- |
| State | `build_initial_state()`, `update_state()` | Records current progress |
| Memory Store | `memory_store.json`, `load_memory()`, `save_memory()` | Stores reusable information |
| Memory Retrieval | `retrieve_memory()` | Selects relevant memory |
| Memory Write | `maybe_write_memory_from_user_input()` | Saves explicit reusable preference |
| Context Assembly | `build_context()` | Combines user input, State, memory, and observations |
| Observation | `make_observation()` | Cleans Tool raw result |
| Trace | `state["trace"]` | Records response ids and tool calls |

## 8. What to Observe After Running

Because real LLM output is not always word-for-word identical, do not require exact output equality. Observe the chain:

- Does the model return `function_call`?
- Does Runtime execute the local Tool?
- Does Observation update State?
- Is Memory written and retrieved?
- Does `done` increase and `missing` shrink?
- Is the final answer generated from State and Observations?

## 9. This Is Not a Production Memory System

This lab uses a JSON file for Memory, simple keyword retrieval, local mock search data, and a single Agent loop. Production systems may use databases, vector stores, permission filters, expiration policies, richer Trace, evaluations, and human review.

## 10. Lab Summary

This lab connects Tool Calling with API calls and Memory:

```text
User Input -> Memory Retrieval -> Context Assembly -> Responses API -> Function Call -> Local Tool -> Observation -> State -> Memory
```

The key lesson is that API calls are only one part of an Agent. Runtime, State, Memory, Tool System, and Context Assembly remain system responsibilities.

## References

OpenAI API Reference: https://platform.openai.com/docs/api-reference/responses/create

OpenAI Function Calling Guide: https://platform.openai.com/docs/guides/function-calling

OpenAI Conversation State Guide: https://platform.openai.com/docs/guides/conversation-state
