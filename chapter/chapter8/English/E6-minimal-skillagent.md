# Lab 6 - Minimal SkillAgent

*Package Agent capability with Skill Registry, Progressive Loading, and Skill Package*

Lab 5 placed real API calls, Memory Store, Memory Retrieval, Context Assembly, and local Tool execution into a minimal Agent. Chapter 8 adds the Skill layer: when Agent capabilities grow, do not put every rule, tool, knowledge source, and output format into one prompt. Package a class of tasks into a Skill.

This lab builds a minimal runnable SkillAgent. It shows how Skill Registry, Skill Selection, Progressive Loading, Skill Package, Runtime, Tool, Memory, and output constraints connect.

> **Reading Note**
>
> This is not a production Skill platform. The goal is to show that Skill is not a fancy prompt. It is a capability package that can be registered, selected, loaded, executed, and tested.

## 1. Why Chapter 8 Needs a Lab

If Skill remains only a concept, it is easy to misunderstand it as "a better prompt." A Skill does contain Instruction, Procedure, and Output Schema, and those may enter Context. But engineering-wise, the point of Skill is packaging reusable capability.

| Lab 5 already had | Lab 6 adds |
| --- | --- |
| API calls to LLM | Still uses API, but does not put every task rule directly into the LLM |
| Memory Store and Retrieval | Memory is retrieved according to selected Skill |
| Local Tool execution | Tool availability is limited by Skill `allowed_tools` |
| Context Assembly | Adds selected Skill instruction and output schema |
| Trace | Records Skill Selection and Skill Package loading |

## 2. Full Chain This Lab Runs

- **Skill Registry:** lightweight metadata such as id, name, description, triggers, version, and permissions.
- **Skill Selection:** Runtime selects a Skill from Goal and registry metadata.
- **Progressive Loading:** load metadata first; load full Skill Package only after selection.
- **Skill Package:** instruction, procedure topics, allowed tools, knowledge sources, output schema, and failure handling.
- **SkillAgent:** the runtime chain formed after loading a Skill.

**Diagram 8P-1: Minimal chain**

```text
User Input
    -> Extract Goal
    -> Load Skill Registry metadata
    -> Skill Selection
    -> Load selected Skill Package
    -> Retrieve Memory by selected Skill
    -> Execute allowed Tools
    -> Assemble Final Context
    -> LLM generates answer under Skill constraints
```

> **Figure 8P-1: Minimal SkillAgent runtime chain**
>
> The diagram should show User Input and Goal on the left, Skill Registry metadata and Skill Selection in the middle, Selected Skill Package after selection, Memory Store and Tool Registry below, Context Assembly and LLM on the right, and Trace recording the process.

## 3. Setup: API Key, Model Name, and Dependency

```powershell
# PowerShell
python -m pip install --upgrade openai
$env:OPENAI_API_KEY = "your API key"
$env:OPENAI_MODEL = "gpt-5.5"
python skill_agent.py
```

The code reads model name from `OPENAI_MODEL`. If the default model is not available, set it to a model your account can use.

## 4. What Objects This Lab Builds

| Object | Code location | Role |
| --- | --- | --- |
| Skill Registry | `SKILL_REGISTRY` | Stores metadata for Skill Selection |
| Skill Package | `SKILL_PACKAGES` | Stores full Skill details, loaded only after selection |
| Skill Selection | `select_skill()` | Lets the LLM select a Skill from Goal and metadata |
| Progressive Loading | `load_skill_registry_metadata()`, `load_skill_details()` | Loads lightly first, then loads only what is needed |
| Allowed Tools | `skill["allowed_tools"]` | Restricts tools for current Skill |
| Tool Registry | `TOOL_REGISTRY` | Local executable functions |
| Memory Retrieval | `retrieve_memory()` | Retrieves memory relevant to selected Skill |
| Agent Runtime | `run_agent()` | Connects Goal, Skill, Memory, Tool, Context, and final answer |

## 5. Complete Code: Minimal SkillAgent

Save the following as `skill_agent.py`.

**Code 8P-2A: Imports, API calls, and JSON parsing**

```python
# skill_agent.py
# Setup:
#   1. python -m pip install --upgrade openai
#   2. Set OPENAI_API_KEY
#   3. If the default model is unavailable, set OPENAI_MODEL

import copy
import json
import os
from datetime import datetime, timezone
from pathlib import Path

try:
    from openai import OpenAI
except ImportError as exc:
    raise SystemExit("Run first: python -m pip install --upgrade openai") from exc


MODEL = os.getenv("OPENAI_MODEL", "gpt-5.5")
MEMORY_FILE = Path("skill_memory_store.json")


def now_iso():
    return datetime.now(timezone.utc).isoformat()


def require_api_key():
    if not os.getenv("OPENAI_API_KEY"):
        raise SystemExit("Please set OPENAI_API_KEY first.")


def response_text(response):
    """Minimal text extraction compatible with multiple SDK versions."""
    if getattr(response, "output_text", None):
        return response.output_text

    chunks = []
    for item in getattr(response, "output", []) or []:
        for part in getattr(item, "content", []) or []:
            text = getattr(part, "text", None)
            if text:
                chunks.append(text)
    return "\n".join(chunks)


def call_llm_text(client, system_prompt, user_prompt):
    response = client.responses.create(
        model=MODEL,
        input=[
            {"role": "system", "content": system_prompt},
            {"role": "user", "content": user_prompt},
        ],
    )
    return response_text(response).strip()


def parse_json_object(text):
    """Extract a JSON object from model output. Production systems should use stricter structured output."""
    try:
        return json.loads(text)
    except json.JSONDecodeError:
        start = text.find("{")
        end = text.rfind("}")
        if start >= 0 and end > start:
            return json.loads(text[start : end + 1])
        raise


def call_llm_json(client, system_prompt, user_prompt):
    text = call_llm_text(client, system_prompt, user_prompt)
    return parse_json_object(text)
```

**Code 8P-2B: Memory, knowledge mock Tool, and Tool Registry**

```python
def load_memory():
    if MEMORY_FILE.exists():
        return json.loads(MEMORY_FILE.read_text(encoding="utf-8"))

    return [
        {
            "id": "mem_user_report_style",
            "type": "user_preference",
            "scope": "global",
            "content": "User prefers reports to start with conclusions, then evidence, then recommendations.",
            "updated_at": now_iso(),
        },
        {
            "id": "mem_ev_skill_scope",
            "type": "procedural_memory",
            "scope": "ev_market_analysis",
            "content": "For EV market analysis, compare sales, price, policy, and battery cost.",
            "updated_at": now_iso(),
        },
    ]


def save_memory(memory_store):
    MEMORY_FILE.write_text(json.dumps(memory_store, ensure_ascii=False, indent=2), encoding="utf-8")


def maybe_write_memory_from_user_input(user_input, memory_store):
    written = []
    if "prefer" in user_input.lower() or "first give conclusions" in user_input.lower():
        item = {
            "id": f"mem_user_pref_{len(memory_store) + 1}",
            "type": "user_preference",
            "scope": "global",
            "content": f"User preference inferred from input: {user_input}",
            "updated_at": now_iso(),
        }
        memory_store.append(item)
        written.append(item)
    return written


def retrieve_memory(user_input, skill_id, memory_store):
    return [
        item for item in memory_store
        if item.get("scope") in ["global", skill_id]
    ][:5]


def retrieve_market_knowledge(topic):
    data = {
        "sales changes": "Sales growth slowed in some months while leading brands stayed stable.",
        "price changes": "Automakers increased discounts and adjusted prices for selected models.",
        "policy changes": "Regional trade-in and subsidy policies affected short-term demand.",
        "battery cost changes": "Battery material prices affected margins and pricing decisions.",
    }
    return {"topic": topic, "content": data.get(topic, "No mock knowledge found."), "source": "mock_market_knowledge"}


TOOL_REGISTRY = {"retrieve_market_knowledge": retrieve_market_knowledge}
```

**Code 8P-2C: Skill Registry, Skill Package, and Skill Selection**

```python
SKILL_REGISTRY = [
    {
        "id": "ev_market_analysis",
        "name": "New-Energy Vehicle Market Analysis Skill",
        "description": "Analyze recent sales, pricing, policy, and battery cost changes.",
        "triggers": ["new-energy vehicle", "EV", "industry analysis", "sales changes"],
        "version": "1.0.0",
        "permissions": {"tools": ["retrieve_market_knowledge"]},
    },
    {
        "id": "generic_summary",
        "name": "Generic Summary Skill",
        "description": "Summarize general text or notes.",
        "triggers": ["summarize", "summary", "notes"],
        "version": "1.0.0",
        "permissions": {"tools": []},
    },
]


SKILL_PACKAGES = {
    "ev_market_analysis": {
        "metadata": SKILL_REGISTRY[0],
        "instruction": "Produce a concise industry analysis grounded in available observations.",
        "procedure_topics": ["sales changes", "price changes", "policy changes", "battery cost changes"],
        "allowed_tools": ["retrieve_market_knowledge"],
        "knowledge_sources": ["mock_market_knowledge"],
        "output_schema": ["conclusion", "evidence", "risks", "recommendations"],
        "failure_handling": ["If a topic has no evidence, say so instead of inventing facts."],
    },
    "generic_summary": {
        "metadata": SKILL_REGISTRY[1],
        "instruction": "Summarize the given material clearly.",
        "procedure_topics": [],
        "allowed_tools": [],
        "knowledge_sources": [],
        "output_schema": ["summary", "key_points"],
        "failure_handling": ["Ask for material if input is insufficient."],
    },
}


def load_skill_registry_metadata():
    return copy.deepcopy(SKILL_REGISTRY)


def load_skill_details(skill_id):
    package = SKILL_PACKAGES.get(skill_id)
    return copy.deepcopy(package) if package else None


def select_skill(client, goal, registry_metadata):
    system = "You are a Skill selection module. Choose the best Skill. Output JSON only."
    user = json.dumps(
        {
            "goal": goal,
            "available_skills": registry_metadata,
            "output_format": {"skill_id": "id", "reason": "short reason"},
        },
        ensure_ascii=False,
        indent=2,
    )
    result = call_llm_json(client, system, user)
    if result.get("skill_id") not in SKILL_PACKAGES:
        return {"skill_id": "ev_market_analysis", "reason": "Fallback to the market analysis Skill."}
    return result
```

**Code 8P-2D: Runtime, Progressive Loading, and final output**

```python
def extract_goal(client, user_input):
    system = "Extract a clear executable task goal from user input. Output JSON only."
    user = json.dumps(
        {"user_input": user_input, "output_format": {"goal": "clear task goal", "assumptions": ["assumption"]}},
        ensure_ascii=False,
        indent=2,
    )
    result = call_llm_json(client, system, user)
    return result.get("goal", user_input), result.get("assumptions", [])


def build_initial_state(goal, skill):
    return {
        "goal": goal,
        "selected_skill": skill["metadata"]["id"] if skill else None,
        "loaded_skill_sections": [],
        "observations": [],
        "trace": [],
        "final_answer": None,
    }


def run_tool(tool_name, arguments, allowed_tools):
    if tool_name not in allowed_tools:
        raise PermissionError(f"Tool not allowed by selected Skill: {tool_name}")
    if tool_name not in TOOL_REGISTRY:
        raise KeyError(f"Unknown tool: {tool_name}")
    result = TOOL_REGISTRY[tool_name](**arguments)
    return {
        "tool_name": tool_name,
        "arguments": arguments,
        "source": result["source"],
        "summary": f"{result['topic']}: {result['content']}",
    }


def execute_skill_procedure(skill, state):
    allowed_tools = skill["allowed_tools"]
    for topic in skill["procedure_topics"]:
        observation = run_tool("retrieve_market_knowledge", {"topic": topic}, allowed_tools)
        state["observations"].append(observation)
        state["trace"].append(
            {"step": f"retrieve {topic}", "tool": observation["tool_name"], "source": observation["source"]}
        )


def build_final_context(user_input, goal, skill, state, retrieved_memories):
    memory_lines = [f"- [{m['type']}/{m['scope']}] {m['content']}" for m in retrieved_memories]
    observation_lines = [f"- [{o['source']}] {o['summary']}" for o in state["observations"]]
    return {
        "user_input": user_input,
        "goal": goal,
        "skill_instruction": skill["instruction"] if skill else "",
        "output_schema": skill["output_schema"] if skill else [],
        "failure_handling": skill["failure_handling"] if skill else [],
        "retrieved_memory": memory_lines,
        "observations": observation_lines,
    }


def generate_final_answer(client, context):
    system = (
        "You are an Agent using a selected Skill. "
        "Follow skill_instruction and output_schema. "
        "Base factual claims only on observations and retrieved_memory."
    )
    user = json.dumps(context, ensure_ascii=False, indent=2)
    return call_llm_text(client, system, user)


def run_agent(user_input):
    require_api_key()
    client = OpenAI()

    memory_store = load_memory()
    written_memories = maybe_write_memory_from_user_input(user_input, memory_store)

    goal, assumptions = extract_goal(client, user_input)
    registry_metadata = load_skill_registry_metadata()
    decision = select_skill(client, goal, registry_metadata)
    skill = load_skill_details(decision["skill_id"])
    if skill is None:
        raise SystemExit("No matching Skill was found.")

    state = build_initial_state(goal, skill)
    state["loaded_skill_sections"] = [
        "metadata",
        "instruction",
        "procedure_topics",
        "allowed_tools",
        "output_schema",
        "failure_handling",
    ]

    retrieved_memories = retrieve_memory(user_input, skill["metadata"]["id"], memory_store)
    execute_skill_procedure(skill, state)
    final_context = build_final_context(user_input, goal, skill, state, retrieved_memories)
    state["final_answer"] = generate_final_answer(client, final_context)
    save_memory(memory_store)

    print("\n=== Goal ===")
    print(goal)
    print("\n=== Skill Selection ===")
    print(json.dumps(decision, ensure_ascii=False, indent=2))
    print("\n=== Written Memories ===")
    print(json.dumps(written_memories, ensure_ascii=False, indent=2))
    print("\n=== Trace ===")
    print(json.dumps(state["trace"], ensure_ascii=False, indent=2))
    print("\n=== Final Answer ===")
    print(state["final_answer"])


if __name__ == "__main__":
    run_agent("Please first give conclusions and analyze the new-energy vehicle industry over the last three months.")
```

## 6. How Progressive Loading Appears in Code

| Stage | Code location | Loaded content | Why |
| --- | --- | --- | --- |
| Stage 1 | `load_skill_registry_metadata()` | id, description, triggers, version, permissions | Low-cost selection |
| Stage 2 | `select_skill()` | Goal + metadata | Choose Skill from light info |
| Stage 3 | `load_skill_details()` | Only selected Skill Package | Avoid loading every Skill |
| Stage 4 | `execute_skill_procedure()` | Allowed tools only | Keep permission boundary |
| Stage 5 | `build_final_context()` | Selected Skill, Memory, Observations | Reduce irrelevant rules |

> **Key Understanding**
>
> If every `SKILL_PACKAGE` is placed into Context from the start, the lab loses the point of Chapter 8. Skill System is valuable because the Runtime loads only the needed capability.

## 7. Chapter 8 Concepts in Code

| Concept | Code location | Meaning |
| --- | --- | --- |
| Skill | `SKILL_PACKAGES["ev_market_analysis"]` | Reusable capability package |
| Skill Registry | `SKILL_REGISTRY` | Lightweight selection metadata |
| Skill Selection | `select_skill()` | Chooses the matching Skill |
| Progressive Loading | `load_skill_registry_metadata()`, `load_skill_details()` | Metadata first, details later |
| Skill Invocation | `run_agent()` after selection | Select, load, and use Skill |
| Allowed Tools | `skill["allowed_tools"]` | Restricts executable tools |
| Permission Boundary | `run_tool()` | Rejects tools not allowed by Skill |
| Context Assembly | `build_final_context()` | Adds Skill, Memory, and Observations |
| Trace | `state["trace"]` | Records runtime execution |

Tool Registry and Skill Registry are not the same. Tool Registry stores executable functions. Skill Registry stores reusable capabilities. A Skill may reference Tools, but Tool is not Skill.

## 8. What to Observe After Running

Observe whether:

- Goal is extracted from User Input.
- Runtime first loads Skill Registry metadata.
- Skill Selection chooses `ev_market_analysis`.
- Runtime loads only the selected Skill Package.
- Allowed Tools restrict execution.
- Observations are written into State.
- Final Answer follows Skill Instruction and Output Schema.

## 9. This Is Not a Production Skill System

This lab uses Python dictionaries for Skill Registry and Skill Package, a mock Knowledge Tool, LLM-based selection, and simple memory filtering. A production system may use packaged skill directories, versioned configs, role-based permissions, RAG, automated evals, and observability.

## 10. Lab Summary

Lab 5 solved how an Agent calls a real API and uses Memory. Lab 6 adds Skill System:

```text
Goal -> Skill Registry -> Skill Selection -> Selected Skill Package -> Allowed Tools -> Context -> Final Answer
```

Skill is not a mysterious layer. It is a way to package, select, load, execute, and manage reusable Agent capability.

## References

OpenAI API Reference: https://platform.openai.com/docs/api-reference/responses/create

OpenAI Function Calling Guide: https://platform.openai.com/docs/guides/function-calling

OpenAI Agents SDK Guide: https://platform.openai.com/docs/guides/agents
