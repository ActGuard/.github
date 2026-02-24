# ActGuard

> Runtime control for AI agents.

Build AI agents that can act in production — with budgets, policies, and verifiable human approvals built in.

ActGuard sits between an agent’s reasoning and its execution.  
Before any tool call runs, ActGuard enforces limits, permissions, and approval rules.

No prompt tricks. No “please behave.” Deterministic enforcement.

---

## Why ActGuard?

Once agents can:

- Send emails  
- Call APIs  
- Move money  
- Modify databases  
- Execute trades  

Safety becomes an execution problem — not a text filtering problem.

ActGuard gives your agents real boundaries.

---

## Core Capabilities

### 💰 Programmable Budgets
Attach USD limits per session, user, or tenant.
Execution stops automatically when limits are reached.

### ⚙️ Runtime Policies
Wrap any function with rate limits and access rules.
Policies execute independently of the model.

### 🔐 Credential Isolation
Agents never handle raw secrets.
ActGuard injects credentials at execution time.

### ✍️ Verifiable Human Approvals
Cryptographically bound approvals.
Prove whether an action was autonomous or explicitly approved.

### 🔁 Exactly-Once Execution
Prevent duplicate transfers, deletes, or retries.
Sensitive actions execute once — and only once.

### 📘 Execution Ledger
Structured JSON record of every decision, allowance, block, and approval.

---

## Quickstart

```python
from actguard import BudgetGuard, BudgetExceededError
import openai

client = openai.OpenAI()

try:
    with BudgetGuard(user_id="alice", token_limit=10_000) as guard:
        response = client.chat.completions.create(
            model="gpt-4o",
            messages=[{"role": "user", "content": "Summarise the history of Rome."}],
        )
        print(response.choices[0].message.content)

except BudgetExceededError as e:
    print(f"Budget hit: {e}")
finally:
    print(f"Spent ${guard.usd_used:.6f} using {guard.tokens_used} tokens")


@actguard.tool(
    rate_limit={"max_calls": 10, "period": 60, "scope": "user_id"},
    circuit_breaker={"name": "search_api", "max_fails": 3, "reset_timeout": 60},
)
def search_web(user_id: str, query: str) -> str:
    ...
```

That’s it. Any function can be governed.
