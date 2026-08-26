# Lesson 12 — Kubernetes RBAC, ServiceAccount, Secret, and Agent Least Privilege

## Learning goals

After this lesson you should be able to explain why an Agent workload needs its own identity, distinguish `ServiceAccount`, `Role`, `RoleBinding`, `ConfigMap`, and `Secret`, and verify permissions with `kubectl auth can-i`.

The core rule is simple:

> The component that schedules Agent Jobs may need limited Kubernetes API permissions. The Agent Runner executing model-directed tools normally should not.

## 1. Two identities, two responsibilities

A useful enterprise design separates the control plane from the execution plane:

```text
Agent API / Scheduler
        |
        | create/get/watch Jobs
        v
Kubernetes API
        |
        v
Agent Job Pod
        |
        +-- Git
        +-- Shell
        +-- Build / Test
        +-- Model / Tools
```

Use two ServiceAccounts:

```text
agent-scheduler
  -> may create/get/watch selected Jobs

agent-runner
  -> should not create Pods
  -> should not list cluster Secrets
  -> should not modify Deployments
```

This limits the blast radius if a prompt, tool call, dependency, or generated command behaves unexpectedly.

## 2. ServiceAccount: workload identity

A Kubernetes `ServiceAccount` gives a workload an identity inside the cluster.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: agent-runner
  namespace: agent-lab
```

Do not confuse it with a human login account. It is an identity intended for workloads such as Pods.

## 3. Role: what is allowed

A namespaced `Role` describes allowed operations on selected Kubernetes resources.

For a scheduler that only manages Jobs:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: agent-job-scheduler
  namespace: agent-lab
rules:
  - apiGroups: ["batch"]
    resources: ["jobs"]
    verbs: ["create", "get", "list", "watch", "delete"]
```

Read the rule as:

```text
resource = jobs
operations = create/get/list/watch/delete
scope = agent-lab namespace
```

Notice what is deliberately missing: `secrets`, `nodes`, arbitrary `pods`, cluster-wide administration, and unrestricted wildcard permissions.

## 4. RoleBinding: connect identity to permission

A Role does nothing by itself. A `RoleBinding` associates the Role with a subject.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: agent-scheduler-jobs
  namespace: agent-lab
subjects:
  - kind: ServiceAccount
    name: agent-scheduler
    namespace: agent-lab
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: agent-job-scheduler
```

Mental model:

```text
ServiceAccount = who?
Role           = may do what?
RoleBinding    = give this Role to this identity
```

## 5. Verify instead of guessing: `kubectl auth can-i`

Permission debugging should be testable.

```bash
kubectl auth can-i create jobs \
  --as=system:serviceaccount:agent-lab:agent-scheduler \
  -n agent-lab
```

Expected for the scheduler:

```text
yes
```

Now test a dangerous capability:

```bash
kubectl auth can-i list secrets \
  --as=system:serviceaccount:agent-lab:agent-runner \
  -n agent-lab
```

Desired result:

```text
no
```

This is much better than assuming the YAML is correct.

### PowerShell

The same `kubectl` commands work in PowerShell. For multiline commands use the PowerShell backtick rather than Bash `\`:

```powershell
kubectl auth can-i create jobs `
  --as=system:serviceaccount:agent-lab:agent-scheduler `
  -n agent-lab
```

## 6. Role vs ClusterRole

Start with the smallest scope.

```text
Role
  -> normally namespace scoped

ClusterRole
  -> can describe cluster-scoped or reusable permissions
```

An Agent platform should not jump directly to broad cluster-wide privileges merely because they are easier to configure.

A common anti-pattern is:

```text
Agent Runner
  -> cluster-admin
```

That turns model-directed execution into a potential cluster control interface.

## 7. ConfigMap vs Secret

Not every configuration value is a secret.

Use a `ConfigMap` for non-sensitive configuration such as:

```text
LOG_LEVEL=INFO
AGENT_MAX_STEPS=100
DEFAULT_MODEL=example-model
```

Use a `Secret` for sensitive values such as credentials or tokens.

But an important engineering rule is:

> Kubernetes Secret is a storage/API abstraction, not permission by itself.

If the Runner can read every Secret in the namespace, putting credentials in `Secret` objects has not created least privilege.

The stronger design is:

```text
Task
 -> receives only credentials required for that task
 -> preferably short lived
 -> credential scope is narrow
 -> credential is not printed to logs or event streams
```

## 8. SecurityContext: identity is only one layer

RBAC controls access to the Kubernetes API. It does not automatically restrict Linux permissions inside the container.

A Runner can also use a restrictive `securityContext`:

```yaml
securityContext:
  runAsNonRoot: true
  allowPrivilegeEscalation: false
  capabilities:
    drop:
      - ALL
```

Think in layers:

```text
Kubernetes RBAC
  -> What cluster APIs may this workload call?

Container / Linux security
  -> What may the process do inside its execution environment?

Agent tool policy
  -> Which shell/files/network/tool actions may the model request?
```

No single layer replaces the others.

## 9. Why this matters for Coding Agents

Suppose an Agent generates:

```bash
kubectl get secrets -A
```

There are two very different outcomes.

Bad design:

```text
Agent has cluster-admin
 -> command succeeds
 -> cluster credentials may enter model/tool output
```

Better design:

```text
Agent Runner lacks Kubernetes API permission
 -> request is denied
 -> control plane remains separated
```

This illustrates defense in depth: even if an Agent-level permission check is misconfigured, infrastructure permissions can still block the action.

## 10. Mini lab

Create a namespace and the two identities:

```bash
kubectl create namespace agent-lab
kubectl create serviceaccount agent-scheduler -n agent-lab
kubectl create serviceaccount agent-runner -n agent-lab
```

Create `scheduler-role.yaml` containing the Role and RoleBinding shown above, then apply it:

```bash
kubectl apply -f scheduler-role.yaml
```

Verify four questions:

```bash
kubectl auth can-i create jobs --as=system:serviceaccount:agent-lab:agent-scheduler -n agent-lab
kubectl auth can-i get jobs --as=system:serviceaccount:agent-lab:agent-scheduler -n agent-lab
kubectl auth can-i create jobs --as=system:serviceaccount:agent-lab:agent-runner -n agent-lab
kubectl auth can-i list secrets --as=system:serviceaccount:agent-lab:agent-runner -n agent-lab
```

A sensible result is:

```text
scheduler create jobs -> yes
scheduler get jobs    -> yes
runner create jobs    -> no
runner list secrets   -> no
```

## 11. Failure case: Agent can read Secrets unexpectedly

Start with evidence:

```bash
kubectl auth can-i list secrets \
  --as=system:serviceaccount:agent-lab:agent-runner \
  -n agent-lab
```

If the result is `yes`, investigate:

```text
Which ServiceAccount is the Pod actually using?
        |
Which RoleBindings reference it?
        |
Is a ClusterRoleBinding granting broader access?
        |
Are wildcard resources or verbs used?
        |
Can the permission be narrowed to one namespace/resource?
```

Do not solve it by adding another broad Role. Find the permission path that produced the unexpected Allow.

## 12. Enterprise design rule

A useful default split is:

```text
agent-scheduler
  Kubernetes API permissions:
    create/get/list/watch/delete Jobs

agent-runner
  Kubernetes API permissions:
    none unless a concrete task requires them

Agent tools
  shell/files/git/network:
    separately constrained
```

This creates a clean control-plane/execution-plane boundary.

## Checkpoint

You should now be able to answer:

1. What is the difference between ServiceAccount, Role, and RoleBinding?
2. Why should an Agent Runner normally not receive `cluster-admin`?
3. Why does storing a token in a Kubernetes Secret not automatically make access safe?
4. What does `kubectl auth can-i` help diagnose?
5. Why are RBAC and Linux/container sandboxing separate security layers?

## Next lesson

**Lesson 13 — Kubernetes NetworkPolicy, egress control, and Agent network boundaries**: control which external services an Agent Job may reach, and connect network isolation to package installation, Git hosting, model APIs, MCP servers, and data-exfiltration risk.
