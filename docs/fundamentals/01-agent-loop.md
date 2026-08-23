# Lesson 01 — The agent loop

An agent is a loop that observes context, chooses an action, executes a tool or returns an answer, then incorporates the result. The engineering problem is controlling state, permissions, failures, and cost.

```text
request → plan/decision → tool call → result → next decision → final response
```

**Lab:** implement a loop with a step limit, structured events, timeout, and a fake calculator tool. Ask: what happens when a tool returns malformed data?
