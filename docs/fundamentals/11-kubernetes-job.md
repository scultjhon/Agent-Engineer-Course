# Lesson 11 — Kubernetes Jobs

A Job represents work that should complete. It is a natural primitive for isolated agent tasks: define an image, command, resource limits, retry policy, service account, and output location. Pods are ephemeral; durable state belongs outside them.

**Lab:** run a one-shot `sample-app` Job, inspect logs, test a failing retry, and remove the resource safely.
