# ActGuard

**ActGuard validates agent behavior over time.**

Your code validates what is being done.  
ActGuard validates whether it *should* be done, given **how we got here**.

Agent failures usually happen **across calls**, not inside a single function: wrong IDs carried between steps, retry storms, and budget drift over a session.

---

## Why agents break (and what ActGuard prevents)

| Real-world problem | What actually happens | ActGuard |
|--------------------|----------------------|----------|
| Made-up data | Agent uses an ID it never fetched | ✅ |
| Lost context | Correct ID fetched → wrong one used later | ✅ |
| Endless retries | Same tool called over and over with tiny changes | ✅ |
| Runaway costs | Agent keeps exploring and silently spends | ✅ |
| Skipped workflow steps | Performs side effect before required step | ✅ |
| Obeying malicious input | Untrusted text tells it to do something destructive | ✅ |

---

## Primary use cases

### 1) BudgetGuard — keep session cost bounded

```python
from actguard import BudgetGuard, BudgetExceededError
import openai

client = openai.OpenAI()

try:
    with BudgetGuard(user_id="alice", usd_limit=0.05):
        client.chat.completions.create(
            model="gpt-4o",
            messages=[{"role": "user", "content": "Summarize this ticket thread."}],
        )
except BudgetExceededError:
    # stop, downgrade model/tooling, or request user approval
    pass
```
**Prevents silent cost drift** when an agent keeps exploring, retrying, or over-calling models/tools.

### 2) Prove / Enforce — validate the journey, not just the input

```python
import actguard
from actguard.exceptions import GuardError

@actguard.prove(kind="order_id", extract="id")
def list_orders(user_id: str) -> list[dict]:
    return [{"id": "o1"}]

@actguard.enforce([actguard.RequireFact("order_id", "order_id")])
def cancel_order(order_id: str) -> str:
    return f"cancelled:{order_id}"

try:
    with actguard.session("req-123", {"user_id": "alice"}):
        list_orders("alice")
        cancel_order("o1")
except GuardError as e:
    hint_for_llm = e.to_prompt()
    # Feed hint_for_llm back to the agent so it can self-correct.
```
Blocks actions that look valid by input, but are invalid for the session’s workflow history.

## Complementary tool guards

After budget + workflow integrity, these decorators cover common runtime guardrails:
- rate_limit — cap call volume in a time window
- circuit_breaker — stop hammering unhealthy dependencies
- max_attempts — cap retries/attempts per run
- timeout — bound wall-clock execution time
- idempotent — deduplicate side-effectful operations
- tool(...) — compose multiple guards in one declaration

```python
import actguard
from actguard import RunContext

@actguard.tool(
    idempotent={"ttl_s": 600, "on_duplicate": "return"},
    max_attempts={"calls": 3},
    rate_limit={"max_calls": 10, "period": 60, "scope": "user_id"},
    circuit_breaker={"name": "search_api", "max_fails": 3, "reset_timeout": 60},
    timeout=2.0,
)
def search_web(user_id: str, query: str, *, idempotency_key: str) -> str:
    ...

with RunContext():
    search_web("alice", "latest earnings", idempotency_key="req-1")
```

# Install

```bash
pip install actguard
```

# Quick links
- 📘 [Getting Started](https://github.com/ActGuard/actguard/blob/main/docs/getting-started.md)
- 🧰 [Tool Guards](https://github.com/ActGuard/actguard/blob/main/docs/tool-guards.md)
- 🔎 [API Reference](https://github.com/ActGuard/actguard/blob/main/docs/api-reference.md)

# Repo structure

```bash
actguard/
├── docs/           # Documentation
├── examples/       # Usage examples
└── libs/
    ├── sdk-py/     # Python SDK
    └── sdk-js/     # JavaScript/Node.js SDK (in progress)
```

# Contributing / Development

Python SDK setup, tests, and lint commands live in libs/sdk-py/.

