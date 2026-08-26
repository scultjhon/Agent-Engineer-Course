# Agent Engineer Course

An open, practical learning path for enterprise Agent engineering, Coding Agents, model tooling, deployment, observability, evaluation, and security.

> **Mobile / tablet:** open the course site first. It provides navigation, full-text search, previous/next lesson links, and a better reading layout than the GitHub file view.

## Start learning

- **[Open the online course](https://scultjhon.github.io/Agent-Engineer-Course/)**
- **[Course roadmap](https://scultjhon.github.io/Agent-Engineer-Course/roadmap/)**
- **[Continue with the latest published lesson: Lesson 12 — Kubernetes RBAC](https://scultjhon.github.io/Agent-Engineer-Course/fundamentals/12-kubernetes-rbac/)**
- **[Source-code study](https://scultjhon.github.io/Agent-Engineer-Course/source-code/)** — Codex, OpenCode, Kimi Code, and Claude Code public mechanisms
- **[Model engineering](https://scultjhon.github.io/Agent-Engineer-Course/model-engineering/)** — Hugging Face and model-engineering topics
- **[Labs](https://scultjhon.github.io/Agent-Engineer-Course/labs/)**

## Current published progress

Lessons 1–12 are currently in the repository. Lesson 12 introduces Kubernetes ServiceAccount, Role/RoleBinding, Secret boundaries, SecurityContext, and least privilege for Agent Scheduler and Runner workloads. Next is NetworkPolicy and Agent egress control.

## Local preview

```bash
python -m pip install mkdocs-material
mkdocs serve
```

Then open the local address printed by MkDocs in a desktop browser.

## Public-content policy

Examples use generic names such as `sample-app`, `demo-repo`, and placeholder paths. Do not add personal projects, credentials, private paths, account-specific data, or secrets to course examples.

## Publishing

A GitHub Actions workflow in `.github/workflows/deploy.yml` builds the MkDocs Material site from `main` and deploys it to GitHub Pages. If Pages is not yet visible, repository Settings → Pages may need **GitHub Actions** selected as the source.

## License and disclaimer

Educational material only. Verify current product behavior and official documentation before using these ideas in production. See [Disclaimer](docs/disclaimer.md).
