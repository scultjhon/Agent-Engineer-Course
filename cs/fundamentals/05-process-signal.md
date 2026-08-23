# Lesson 05 — Processes and signals

Processes have parents, process groups, file descriptors, and lifecycle events. Graceful shutdown usually sends `SIGTERM`, waits, then escalates to `SIGKILL` only when policy allows.

**Lab:** start a child process, forward termination, reap it, and verify no orphan remains.
