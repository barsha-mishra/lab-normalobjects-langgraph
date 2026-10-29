# Lab Proof: NormalObjects Complaint Processor

## 1. What I built

A **strict complaint processor** for NormalObjects, built with **LangGraph** and `gpt-4o-mini` (`temperature=0`). Every complaint follows a fixed path through six nodes:

```
START → intake → validate ──(valid)──→ investigate → resolve → close → END
                     │
                     └──(invalid)──→ reject → END
```

| Node | What it does | Uses the LLM? |
|---|---|---|
| `intake` | Categorizes the complaint (portal, monster, psychic, environmental, other) | Yes |
| `validate` | Checks the category against `VALID_CATEGORIES` and saves the reason | No (rule) |
| `investigate` | Answers **only** from `LORE[category]` and saves that lore text as `evidence` | Yes |
| `resolve` | Writes a short fix based **only** on the investigation | Yes |
| `close` | Final message, status `closed` | No |
| `reject` | Rejection message with the reason, status `rejected` | No |

The routing after `validate` is a **conditional edge** (`route_after_validation`): valid → `investigate`, invalid → `reject`.

Every node adds its name to `workflow_path` and prints a log line (`[INTAKE]`, `[VALIDATE]`, ...). The state (`ComplaintState`) also stores `is_valid`, `reason`, `evidence`, `investigation`, `resolution` and `closure`, so every step can be traced.

**Code:** [`Lab_normalobjects_langgraph.ipynb`](Lab_normalobjects_langgraph.ipynb) (notebook with outputs) and [`normalobjects_langgraph.py`](normalobjects_langgraph.py) (exported script)

**Test result:** 5 complaints → **4 closed, 1 rejected** (as expected).

## 2. Query

> "El can move things with her mind but can't lift heavy rocks. Why?"

## 3. Evidence

**Workflow path:**

```
[CLOSED] intake → validate → investigate → resolve → close
```

**Log lines:**

```
[INTAKE] Processing complaint...
[INTAKE] Categorized as: psychic

[VALIDATE] Checking category...
[VALIDATE] Valid: True | Reason: Category 'psychic' is in the valid list ['portal', 'monster', 'psychic', 'environmental'].
[ROUTE] Valid → investigate

[INVESTIGATE] Looking up lore...
[INVESTIGATE] Lore used (psychic): Psychic power drains the user's energy. Bigger or heavier objects need more energy, and nosebleeds are a warning sign.
[INVESTIGATE] Findings: El's inability to lift heavy rocks with her mind is likely due to the energy drain associated with using psychic powers. Heavier objects require significantly more energy to move, which can lead to fatigue and even nosebleeds as a warning sign.

[RESOLVE] Writing resolution...
[RESOLVE] Resolution: El's difficulty in lifting heavy rocks with her mind is likely due to the energy drain from using her psychic powers, as heavier objects require much more energy, leading to fatigue and potential nosebleeds.

[CLOSE] Closing complaint...
[CLOSE] Status: closed
```

**Lore used (saved in `evidence`):**

> Psychic power drains the user's energy. Bigger or heavier objects need more energy, and nosebleeds are a warning sign.

## 4. Final output

```
Category: psychic
Status:   closed
Closure:  Complaint closed. Category: psychic. Resolution: El's difficulty in lifting heavy rocks with her mind is likely due to the energy drain from using her psychic powers, as heavier objects require much more energy, leading to fatigue and potential nosebleeds.
```

## 5. Trace

| Claim in the resolution | Lore line | Grounded? |
|---|---|---|
| Difficulty is due to the energy drain from psychic powers | "Psychic power drains the user's energy." | ✅ |
| Heavier objects require much more energy | "Bigger or heavier objects need more energy" | ✅ |
| Potential nosebleeds | "nosebleeds are a warning sign" | ✅ (but the lore calls them a *warning sign*, the resolution makes them a *result*) |
| Leading to **fatigue** | Not in the lore | ❌ Added by the model |

**Result:** the main claims come from the lore, but the model added one word ("fatigue") and slightly changed the meaning of "nosebleeds". The instruction "use ONLY the lore" reduces invention, but does not prevent it fully.

## 6. Failure or limitation

**Same complaint, different answers, and errors travel down the line.**

I ran the monster complaint ("Demogorgons sometimes work together and sometimes fight. What's their deal?") three times. The lore only says: *"Demogorgons act alone when hunting, but follow a hive mind when a larger entity controls them."* It says **nothing about fighting**.

| Run | What the investigator said about fighting |
|---|---|
| 1 (nodes by hand) | "The lore does not cover the specifics of them fighting each other." ✅ honest |
| 2 (nodes by hand) | "The fighting among themselves could be a result of their independent nature." ❌ invented |
| 3 (full graph) | "The fighting among them could stem from individual instincts or competition." ❌ invented |

So:
1. **The graph makes the path stable, but not the content.** Every run took the same path (`intake → validate → investigate → resolve → close`) and the same category, but the LLM's wording and reasoning changed, even with `temperature=0`.
2. **Errors travel down the line.** The `resolve` node only reads the investigation, not the lore. In runs 2 and 3 it repeated the invented reason as if it were a fact ("Their fighting likely arises from individual instincts or competition").
3. **A "closed" status looks trustworthy.** The ticket ends as `closed` with a confident resolution, so a customer could not tell that part of it is unsupported.

**Other limitations:**
- `validate` only checks the **category**, not whether the lore can actually answer the question. The monster complaint passed validation although the lore only covers half of it.
- `intake` depends on the LLM's category choice. A wrong category would send the complaint to the wrong lore, and the evidence would look valid but be irrelevant.

**Possible fixes:** let `resolve` also see the lore, add a check node that compares the resolution with the lore before `close`, or route "lore does not cover this" answers to a human.

## 7. LangGraph vs. one LLM call

I sent the monster complaint and the invalid complaint to `llm.invoke(...)` directly (no graph, no lore).

- **Monster complaint:** the answer talked about "Stranger Things" and the "Upside Down" (model memory, not the lore), and said Demogorgons cooperate **when hunting**, which **contradicts** the lore ("act alone when hunting").
- **Invalid complaint:** it was **not rejected**. The model answered anyway and invented a funny complaint about missing socks.

| | LangGraph | One LLM call |
|---|---|---|
| Same answer every run? | Same path and category every run. Wording can change (monster: 3 runs, 3 different reasons for fighting) | No structure to keep it stable; answer depends only on the model |
| Invalid complaint rejected? | Yes: `intake → validate → reject`, with the reason | No: it answered anyway and invented a complaint about socks |
| Path visible? | Yes: `workflow_path` and log lines for every station | No: one black-box answer |
| Source of each claim? | The lore text, saved in `evidence` | Model memory ("Stranger Things", "Upside Down"); one claim contradicts the lore |
| Best for which client case? | Support/complaint processes that need rules, rejection and an audit trail | Quick brainstorming or creative answers where traceability doesn't matter |
