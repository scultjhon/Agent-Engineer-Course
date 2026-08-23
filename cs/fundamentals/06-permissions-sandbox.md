# Lesson 06 — Permissions and sandboxes

Least privilege is a design constraint: minimize filesystem access, network access, credentials, and command surface. Make approvals explicit and auditable. A sandbox reduces blast radius but is not a substitute for defense in depth.

**Lab:** run a task with a temporary directory, read-only inputs, no secret environment variables, and a time limit.
