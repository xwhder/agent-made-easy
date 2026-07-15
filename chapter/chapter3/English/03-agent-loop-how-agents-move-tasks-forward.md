# Chapter 3 - Task Progression System: How an Agent Moves from Goal to Result

*From generating answers to moving tasks forward*

The previous chapters established two boundaries: Agent is not merely a chatbot, and Agent is not equal to LLM. This chapter focuses on the core mechanism that makes an Agent different from a one-shot model call: **task progression**.

An Agent does not only produce an answer. It keeps asking: What is the Goal? What do we know now? What is missing? What Action should come next? What did the last Action return? Should we continue, stop, or ask the user?

> **Chapter Route**
>
> We separate User Input, Goal, Prompt, and Context; then explain how Goal becomes Plan and Step; then define Action, Observation, State, Feedback Loop, and Stop Condition. After that, we look at common progression patterns and the classic ReAct style.

## 3.1 From "Generating an Answer" to "Moving a Task Forward"

A normal LLM application often works like this:

```text
User Input -> Build Prompt -> LLM -> Answer
```

This is useful, but it is not enough for many tasks. A task may require several steps: search for information, inspect files, run tests, ask for missing input, compare results, revise output, and stop only when the goal is satisfied.

An Agent needs a progression structure:

```text
Goal -> State -> Decide Action -> Execute Action -> Observation -> Update State -> Decide Again
```

The difference is not cosmetic. The second structure can continue. It can correct course. It can wait for missing information. It can stop when done.

![Figure 3-1: Agent task progression loop](assets/figure-3-1.jpeg)

## 3.2 User Input, Goal, Prompt, and Context Are Not the Same

These four terms are often mixed together, but they solve different problems.

| Term | What it means | Example |
| --- | --- | --- |
| User Input | What the user actually typed | "Analyze the new-energy vehicle industry in the last three months." |
| Goal | The task target the system extracts or clarifies | "Produce a structured evidence-based report about recent industry changes." |
| Prompt | The model input for one call | A formatted request sent to the LLM |
| Context | The information visible to the model in that call | Goal, state, observations, tool descriptions, source snippets |

The user input may be incomplete or ambiguous. The Agent should turn it into a clearer Goal. The Prompt is only one call's input. Context is the working information placed inside that input.

> **Key Boundary**
>
> User Input is not automatically Goal. Observation is not automatically Prompt. State is not automatically Context. The Agent runtime must organize them.

## 3.3 How Goal Becomes Plan and Step

A Goal describes the target. A Plan describes a possible route. A Step is the next unit of execution.

For example:

> Goal: analyze the main changes in the new-energy vehicle industry in the last three months and produce a structured report.

A Plan may be:

1. Collect sales-change information.
2. Collect policy-change information.
3. Collect price-change information.
4. Collect battery-cost information.
5. Synthesize the findings into a report.

The current Step might be:

> Search for recent sales-change information.

The Agent does not always need a fully written plan at the beginning. Sometimes it plans only the next step. Sometimes it revises the plan after seeing new Observations. The important point is that Goal, Plan, and Step should not collapse into one vague instruction.

## 3.4 Action: What Exactly Is the Agent's Next Move?

Action is the next thing the Agent decides to do. Common Actions include:

- Generate a direct answer.
- Call a tool.
- Read a file.
- Search for material.
- Run code or tests.
- Ask the user for clarification.
- Request permission.
- Stop and return the final result.

An Action should be explicit enough for the runtime to execute or reject. "Think about the problem" is not a useful runtime Action. "Call `search_tool` with query X" is.

Action design matters because it creates the interface between model judgment and system execution.

## 3.5 Observation: Results Must Be Collected After Action

After an Action, the system receives an Observation.

| Action | Possible Observation |
| --- | --- |
| Search web | Search results, snippets, source links |
| Read file | File contents or parse error |
| Run tests | Passing tests or failure logs |
| Ask user | User answer or refusal |
| Call database | Query result or permission error |

Observation is not automatically "truth." It may be incomplete, noisy, stale, or malicious. The Agent system may need to summarize, validate, filter, cite, or reject it before using it in the next model call.

## 3.6 State: Why the Task Can Continue

State is the Agent's working ledger. It records progress and lets the task continue across rounds.

A minimal State might contain:

```json
{
  "goal": "Analyze recent new-energy vehicle industry changes",
  "done": ["sales changes", "policy changes"],
  "missing": ["price changes", "battery cost changes"],
  "observations": [],
  "final_answer": null
}
```

Without State, every model call starts from scratch. With State, the Agent can know what has been done, what remains, and whether the next action should continue or stop.

State is not the same as Memory. State is about the current task. Memory is information that may be reused across future turns or tasks. Chapter 6 expands this distinction.

## 3.7 Feedback Loop and Stop Condition: How the Loop Continues and Ends

The Feedback Loop is the structure that lets the Agent use new Observations to update State and make the next decision.

The loop must also have a Stop Condition. Otherwise, the Agent may repeat tool calls, drift from the Goal, or run until cost limits are hit.

Common Stop Conditions include:

- The Goal is complete.
- Required information is collected.
- Maximum steps or cost limit reached.
- Required permission is missing.
- The task is unsafe or out of scope.
- The user needs to clarify something.
- The system encountered an unrecoverable error.

> **Engineering Judgment**
>
> A reliable Agent does not only know how to continue. It knows when to stop, ask, refuse, or hand off.

## 3.8 Common Task Progression Patterns: Agent Is Not Only ReAct

ReAct is famous, but it is not the only task progression style. Agents may use several patterns.

| Pattern | Shape | Good for |
| --- | --- | --- |
| Plan-and-execute | Build a plan, then execute steps | Tasks with clear stages |
| ReAct | Reason, act, observe, repeat | Tool-heavy tasks with uncertainty |
| Reflect-and-revise | Produce, critique, improve | Writing, code review, report polishing |
| Router | Classify task and choose a handler | Systems with multiple skills |
| Human-in-the-loop | Pause for confirmation or input | Risky or ambiguous actions |
| Workflow + Agent | Workflow controls stages, Agent handles flexible parts | Production systems |

The right pattern depends on the task, risk, cost, and expected reliability.

## 3.9 ReAct: A Classic Reasoning + Acting Style

ReAct stands for Reasoning and Acting. It is often described as:

```text
Thought -> Action -> Observation -> Thought -> Action -> Observation -> ...
```

The important idea is that the model does not only answer. It reasons about the next action, the runtime executes that action, the system receives an Observation, and the next reasoning step uses that Observation.

In production systems, we usually do not expose raw "Thought" text. Instead, we store structured traces: what action was proposed, what tool was called, what observation returned, and how state changed.

![Figure 3-2: ReAct style task progression](assets/figure-3-2.png)

## 3.10 Chapter Summary: The Core of Agent Is the Task Progression Loop

This chapter's main point is simple:

**An Agent is not only a model that generates answers. It is a system that moves tasks forward.**

The progression loop contains:

- User Input, which must be interpreted.
- Goal, which defines the target.
- Context, which carries current working information.
- Action, which is the next executable move.
- Observation, which returns results from the world.
- State, which records progress.
- Feedback Loop, which enables continuation.
- Stop Condition, which prevents endless or unsafe execution.

Once you understand this loop, later systems become easier to place: model input systems decide what each round sees; tool systems execute actions; memory systems preserve continuity; knowledge systems provide evidence; skill systems package reusable capabilities.
