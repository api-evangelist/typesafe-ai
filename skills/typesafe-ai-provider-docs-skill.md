---
name: TypeSafe
description: Use when building systems that need fast, structured AI decisions: routing tickets to teams, classifying documents, scoring severity or quality, detecting patterns, or verifying content. TypeSafe is for agents helping developers integrate System One models into applications where code controls the workflow and AI makes focused judgments.
metadata:
    mintlify-proj: typesafe
    version: "1.0"
---

# TypeSafe Skill

## Product summary

TypeSafe is a System One model API (Jev) that makes fast, structured decisions for software. Send a state (text, JSON, or array) and typed questions; get back constrained answers with probabilities and confidence scores. Use the Python SDK (`typesafe-sdk`), JavaScript SDK (`@typesafe-ai/sdk`), or HTTP API at `https://api.typesafe.ai/v1/systemone`. Set `TYPESAFE_API_KEY` in your environment. The model is `jev-latest` by default. See [TypeSafe docs](https://docs.typesafe.ai) for the full reference.

## When to use

Reach for TypeSafe when:
- Building routing, classification, or detection systems where code controls the workflow
- You need a judgment a knowledgeable person could make in seconds (not extended reasoning)
- You want structured answers your code can branch on, sort, or threshold—not generated text
- You need confidence scores to decide when to act automatically vs. escalate to a human
- You're processing many items and need cheap, fast decisions (150ms latency, 100x cheaper than LLMs)
- You need to verify or check content: citations, policy violations, jailbreaks, hallucinations

Do not use TypeSafe for: open-ended generation, creative writing, extended reasoning, or tasks requiring multiple reasoning steps. Use an LLM for those.

## Quick reference

### Three question types

| Type | Use when | Returns | Example |
|------|----------|---------|---------|
| **Choice** | Answer is one of a fixed set of options | `choice`, `probabilities`, `confidence` | Which team: billing, technical, sales? |
| **Score** | Answer is a position on an ordered spectrum | `score`, `legend`, `probabilities`, `confidence` | Bug severity: 0=low, 1=medium, 2=critical |
| **Noul** | Answer is yes/no (probability matters) | `noul` (0–1) | Does this message request a refund? |

### SDK imports and basic call

**Python (sync):**
```python
from typesafe_sdk import TypeSafeClient, Choice, Noul, Score

client = TypeSafeClient()  # reads TYPESAFE_API_KEY from env
response = client.system_one(
    state="Customer message here",
    questions={
        "department": Choice(
            instructions="Which team should handle this?",
            criteria={"billing": "...", "technical": "..."}
        ),
        "is_urgent": Noul(instructions="Is this urgent?"),
    }
)
print(response.answers["department"].choice)
print(response.answers["is_urgent"].noul)
```

**Python (async):**
```python
from typesafe_sdk import AsyncTypeSafeClient, Choice, Noul, Score

async with AsyncTypeSafeClient() as client:
    response = await client.system_one(state=..., questions=...)
```

**JavaScript:**
```ts
import { TypeSafeClient, choice, noul, score } from "@typesafe-ai/sdk";

const client = new TypeSafeClient();
const response = await client.systemOne({
  state: "Customer message here",
  questions: {
    department: choice("Which team?", { billing: null, technical: null }),
    is_urgent: noul("Is this urgent?"),
  },
});
console.log(response.answers.department.choice);
```

### HTTP API

```bash
curl -X POST https://api.typesafe.ai/v1/systemone \
  -H "Authorization: Bearer $TYPESAFE_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "state": "Customer message",
    "model": "jev-latest",
    "questions": {
      "department": {
        "type": "choice",
        "instructions": "Which team?",
        "criteria": {"billing": "...", "technical": "..."}
      }
    }
  }'
```

### State structure

State is the content to evaluate. Use a string for simple cases, JSON object for structured data:

```python
# Simple string
state = "I was charged twice. Please help ASAP."

# Structured object (preferred)
state = {
    "ticket": {
        "subject": "Duplicate charge",
        "messages": [
            {"from": "customer", "text": "I was charged twice..."},
            {"from": "support", "text": "We are checking..."}
        ]
    },
    "order": {"id": "A-104", "charges": [...]},
    "refund_policy": "Duplicate charges are eligible..."
}
```

Reference specific fields in instructions with backtick paths: `Does `ticket.messages[0].text` request a refund?`

### Configuration options

| Option | Where | Purpose | Default |
|--------|-------|---------|---------|
| `TYPESAFE_API_KEY` | Environment | API authentication | Required |
| `TYPESAFE_DEFAULT_MODEL` | Environment | Model to use | `jev-latest` |
| `TYPESAFE_LOG_LEVEL` | Environment | Logging verbosity | `info` |
| `TYPESAFE_BASE_URL` | Environment | API endpoint | `https://api.typesafe.ai` |
| `model` | Request | Override default model | `jev-latest` |
| `retry` | Client init | Retry policy (Python) | Automatic with backoff |

## Decision guidance

### When to use Choice vs. Score vs. Noul

| Decision | Use Choice | Use Score | Use Noul |
|----------|-----------|-----------|----------|
| **Answer shape** | One of N unordered options | Position on ordered spectrum | Yes/no probability |
| **Example: routing** | Which team: billing, technical, sales | — | — |
| **Example: severity** | — | Bug severity: low, medium, critical | — |
| **Example: detection** | — | — | Does this contain spam? |
| **Example: skill level** | — | Python experience: none, some, daily, expert | — |
| **When uncertain** | Use if code branches on the choice | Use if you threshold on a position | Use if probability itself is the signal |

### When to ask one request vs. multiple requests

| Scenario | Approach | Why |
|----------|----------|-----|
| All questions use the same state | **One request** | Parallel evaluation, barely slower, cheaper |
| Second question depends on first answer | **Two requests** | Need the first answer to build the second state or pick options |
| You might not need some answers | **One request (speculative fan-out)** | Extra questions cost only tokens; code ignores unused answers |
| Questions are independent | **One request** | No context-rot; answers don't affect each other |

### When to use confidence thresholds

| Confidence range | Action | Example |
|------------------|--------|---------|
| **< 0.5** | Route to human or escalate | Model is genuinely uncertain; don't guess |
| **0.5–0.7** | Proceed with caution | Ask user to confirm, flag for review, gather more info |
| **> 0.7–0.85** | Act for low-stakes decisions | Show the right screen, route to a queue |
| **> 0.85–0.95** | Act for medium-stakes decisions | Approve a refund, execute a transfer with confirmation |
| **> 0.95** | Act for high-stakes decisions | Execute without confirmation (rare) |

Thresholds depend on your domain and risk tolerance. Start conservative and adjust based on observed results.

## Workflow

### Typical task: build a ticket router

1. **Understand the problem.** What decisions does your code need to make? (e.g., route to billing, technical, or sales)

2. **Check existing patterns.** Search the docs for similar use cases. Look at [Patterns](/patterns) (fan-out, confidence routing, intent routing) and [Cookbooks](/cookbooks) (classification, hierarchical classification, skill suggestion).

3. **Define your state.** Gather the content to evaluate. Use a structured object with named fields:
   ```python
   state = {
       "message": customer_message,
       "order_history": recent_orders,
       "account_status": account_info,
   }
   ```

4. **Define your questions.** Ask one focused judgment per question. Avoid multi-factor questions; decompose them:
   ```python
   questions = {
       "department": Choice(
           instructions="Which team should handle this ticket?",
           criteria={
               "billing": "Payment, subscription, refund issues",
               "technical": "Bugs, outages, integration problems",
               "sales": "Pricing, upgrades, new accounts",
           }
       ),
       "is_urgent": Noul(
           instructions="Does the message convey urgency or time-sensitivity?"
       ),
       "frustration": Score(
           instructions="How frustrated is the customer?",
           criteria=["Calm, neutral", "Frustrated but civil", "Very angry"]
       ),
   }
   ```

5. **Make the request.** Send all questions together; they run in parallel:
   ```python
   response = client.system_one(state=state, questions=questions)
   ```

6. **Route based on answers and confidence.** Combine answers with your logic:
   ```python
   dept = response.answers["department"]
   if dept.confidence < 0.5:
       route_to_human(ticket)
   elif dept.choice == "billing":
       queue_for_billing(ticket)
   else:
       queue_for_technical(ticket)
   ```

7. **Verify results.** Test with real data. Check that confidence thresholds match your risk tolerance. Adjust questions or thresholds if results don't match your team's judgment.

## Common gotchas

- **Asking for reasoning or explanation.** TypeSafe returns structured answers, not text. Don't ask "explain why" or "analyze this." Ask focused yes/no, choice, or scoring questions instead.

- **Multi-factor questions.** Don't ask "is this a good candidate?" (depends on many things). Ask separately: "Does the resume mention Python?", "Does it mention distributed systems?", "Years of experience?" Then combine in code.

- **Criteria that are too vague.** "High priority" is unclear. Define it: "Blocks customer revenue" or "Affects more than 10% of users." The model needs concrete definitions.

- **Contradictory instructions and criteria.** If your instruction says "is this urgent?" but your criteria say "true: not time-sensitive", the model will be confused. Align them.

- **Ignoring confidence.** Confidence tells you when the model is uncertain. Use it to decide when to escalate. A low-confidence answer is not wrong; it's a signal to ask a human.

- **Asking too many questions in one request.** The request has a ~32,000 token budget (shared with state). Very large states + many questions can hit the limit. Check token usage in the response.

- **Literal reading of numbers.** Don't ask the model to do math. "Is the total over $100?" requires the model to parse and add. Instead, extract the numbers in code and compare them.

- **Assuming Noul confidence.** Noul answers don't carry a separate `confidence` field. The `noul` value itself (0–1) is the probability; use it directly as your confidence signal.

- **Changing questions mid-stream.** If you add or remove questions, answers for the remaining questions don't change (they're independent). But your code paths might break if you're not careful. Test changes.

- **Not using speculative fan-out.** Asking 13 questions in one call is 11.5x cheaper and 9.6x faster than 13 separate calls. Include questions your code might not need; let the code decide.

## Verification checklist

Before submitting work:

- [ ] **State is clear and structured.** Use a JSON object with named fields, not a wall of text.
- [ ] **Each question asks one focused judgment.** No multi-factor questions; decompose them.
- [ ] **Instructions are specific.** "Does this message request a refund?" not "Analyze this message."
- [ ] **Criteria are concrete.** "Calm, neutral" not "happy"; "Blocks revenue" not "important."
- [ ] **All questions in one request.** Don't make separate calls unless the second depends on the first answer.
- [ ] **Confidence thresholds are set.** Define what confidence level triggers each action (escalate, confirm, auto-approve).
- [ ] **Tested with real data.** Run a few examples and verify answers match your team's judgment.
- [ ] **Token usage is reasonable.** Check the response's `usage` field; if it's near 32,000, simplify the state.
- [ ] **Error handling is in place.** Handle `401 Unauthorized` (bad key), `422 Unprocessable Entity` (bad request), `429 Too Many Requests` (rate limit).

## Resources

- **[TypeSafe docs llms.txt](https://docs.typesafe.ai/llms.txt)** — Comprehensive page-by-page navigation for agents.
- **[Primitives (Questions)](https://docs.typesafe.ai/primitives)** — Detailed guide to Choice, Score, and Noul with examples.
- **[Patterns](https://docs.typesafe.ai/patterns)** — Speculative fan-out, confidence routing, composite scoring, intent routing.
- **[API Reference](https://docs.typesafe.ai/api)** — Full HTTP API schema and response formats.

---

> For additional documentation and navigation, see: https://docs.typesafe.ai/llms.txt