# Service-Booking Agent Harness

A proposed execution layer for a conversational assistant that searches, creates, reschedules, and cancels service appointments while checking that the database reflects the customer's confirmed request.

**Repository status:** architecture and evaluation design. This repository currently contains documentation; it does not include a runnable booking service or measured evaluation results.

## The problem

A booking assistant must do more than extract a date. A customer may change their mind after confirming, a slot may become unavailable before a write, or the database may complete an operation before its response is lost. Retrying the conversation can then create a duplicate booking or apply an outdated instruction.

This design places an execution layer between the conversation workflow and PostgreSQL. The intended setting is a service-booking sandbox, using NeuralSeek mAIstro as an example orchestration platform. Public platform documentation provides the scenario reference; this repository makes no claim of company deployment or customer adoption.

## Proposed execution flow

1. **Interpret the request.** LLM structured output would identify the intent, service, date window, sorting, and result limit. Missing information or ambiguous references would trigger targeted clarification while retaining confirmed details.
2. **Construct a permitted operation.** A predefined mapping would translate validated fields into SQLAlchemy expressions with bound values. Queries would be scoped to the authenticated customer. This supports a fixed booking schema rather than unrestricted text-to-SQL.
3. **Confirm the current parameters.** Writes would require confirmation tied to the exact service, slot, and booking. Changing those parameters would invalidate earlier confirmation.
4. **Execute and recover.** Transactions, row locks, uniqueness constraints, and idempotency keys would protect writes. Persistent checkpoints would retain the operation ID and confirmed parameters across interruptions.
5. **Verify the final state.** The assistant would read back the booking before reporting success. An uncertain write would be checked by operation ID before any retry.

The five intended business tools are `list_slots`, `get_booking`, `create_booking`, `reschedule_booking`, and `cancel_booking`. Python, FastAPI, HTTPX, and Pydantic are proposed supporting components.

| Situation | Intended response |
|---|---|
| Missing fields or several matching bookings | Ask a specific clarification question |
| No available slot | Explain the result and offer alternative search conditions |
| Slot conflict during a write | Preserve the original booking and obtain confirmation for an alternative |
| Lost response after a possible write | Verify the operation before retrying |
| Unsupported request or exhausted recovery | Offer an actionable alternative or contextual human handoff |

## Evaluation design

The planned setup uses 20 development scenarios and 50 separate held-out scenarios. Each held-out scenario would run four times per configuration, giving 200 runs: 100 routine runs and 100 fault-injection runs. Faults would cover timeouts before submission, rate limits, lost responses after writes, concurrent slot conflicts, and process interruption.

The baseline would use direct tool calls with basic parameter validation and fixed retries. The full harness would add clarification, state checks, idempotency, checkpoints, and write verification. Initial database state, model, user scripts, budgets, and fault conditions would be held constant.

Success would require the correct database state, valid confirmation for every write, no duplicate or unintended update, and an answer consistent with the backend. Additional measures would include fault-case completion, duplicate-write incidence, and API calls across both successful and failed runs. Development failures could inform human-reviewed prompt and intent revisions before a fresh evaluation.

## References

- [NeuralSeek mAIstro documentation](https://documentation.neuralseek.com/ui/maistro/)
- [SQLAlchemy bound parameters](https://docs.sqlalchemy.org/en/20/core/sqlelement.html#sqlalchemy.sql.expression.bindparam)
