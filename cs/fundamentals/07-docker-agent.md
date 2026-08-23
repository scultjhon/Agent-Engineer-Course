# Lesson 07 — Docker and agents

Containers package a predictable user space, but isolation depends on configuration. Prefer non-root users, pinned base images, minimal packages, resource limits, and explicit mounts.

**Lab:** containerize `sample-app`, expose one port, mount a temporary workspace, and verify the process cannot write to the image filesystem.
