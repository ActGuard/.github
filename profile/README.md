# ActGuard

**ActGuard validates agent behavior over time.**

Your code validates what is being done.  
ActGuard validates whether it *should* be done, given **how we got here**.

Traditional checks handle one call at a time. Agent failures often happen **across calls**:
hallucinated IDs carried between steps, retry loops, and budget drift over a session.

---

## What ActGuard covers

| Concern | Typical solution | ActGuard |
|---|---|---|
| Is the input valid? | `if/else` | — |
| Is the caller authorized? | RBAC / Auth | — |
| Is the amount within limits? | Business logic | — |
| Did the agent hallucinate an ID? | — | ✅ |
| Did required steps happen before this action? | — | ✅ |
| Is the agent retrying in a loop? | — | ✅ |
| Is the session within cost budget? | — | ✅ |

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
