# Kubernetes RBAC — Practical Reference

Role-Based Access Control (RBAC) is how Kubernetes decides **who** is allowed to do **what** against **which resources**. This repo is a working reference: concepts, manifests you can apply, verification commands, and the gotchas that bite in real clusters (and in the CKA exam).

> Source of truth: [kubernetes.io — Using RBAC Authorization](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)

---

## Table of Contents

- [Where RBAC sits in the request path](#where-rbac-sits-in-the-request-path)
- [Enabling RBAC](#enabling-rbac)
- [The four API objects](#the-four-api-objects)
- [Roles and ClusterRoles](#roles-and-clusterroles)
- [Bindings](#bindings)
- [Referring to resources](#referring-to-resources)
- [Subjects](#subjects)
- [Default roles shipped with the cluster](#default-roles-shipped-with-the-cluster)
- [Privilege escalation prevention](#privilege-escalation-prevention)
- [Imperative commands](#imperative-commands)
- [Verifying permissions](#verifying-permissions)
- [ServiceAccount patterns](#serviceaccount-patterns)
- [Troubleshooting](#troubleshooting)
- [Good practices](#good-practices)

---

## Where RBAC sits in the request path

Every API call passes through three gates in order:

```
Request → [ Authentication ] → [ Authorization (RBAC) ] → [ Admission Control ] → etcd
            who are you?         are you allowed?           is it acceptable?
```

RBAC only answers the middle question. It never establishes identity — that comes from certificates, tokens, or an OIDC provider upstream.

Two properties worth internalising:

- **Permissions are purely additive.** There is no "deny" rule. If nothing grants an action, it is denied by default.
- **Policy is data.** Rules live in the `rbac.authorization.k8s.io` API group, so you change authorization by applying YAML — no API server restart needed.

---

## Enabling RBAC

Modern clusters use a structured authorization config file passed to `kube-apiserver` via `--authorization-config`:

```yaml
apiVersion: apiserver.config.k8s.io/v1
kind: AuthorizationConfiguration
authorizers:
  - type: Node
  - type: RBAC
```

The older flag form still works:

```bash
kube-apiserver --authorization-mode=Node,RBAC
```

When several authorizers are configured they run in parallel — an allow from **any** of them permits the request.

Check what your cluster is running:

```bash
kubectl -n kube-system get pod kube-apiserver-$(hostname) -o yaml | grep authorization
```

---

## The four API objects

| Object | Scoped to | Grants permissions | Typical use |
|---|---|---|---|
| `Role` | one namespace | rules only | app-team permissions inside a namespace |
| `ClusterRole` | cluster-wide | rules only | reusable rule sets, node/PV access, non-resource URLs |
| `RoleBinding` | one namespace | attaches a Role **or** ClusterRole to subjects | grant scoped to that namespace |
| `ClusterRoleBinding` | cluster-wide | attaches a ClusterRole to subjects | grant across every namespace |

The mental model: **Roles define capability, bindings hand it out.** Nothing takes effect until a binding exists.

The one combination that surprises people:

```
RoleBinding + ClusterRole  =  permissions limited to the RoleBinding's namespace
```

This is the reuse pattern — write the rule set once as a ClusterRole, bind it namespace by namespace.

And the one that is illegal:

```
ClusterRoleBinding + Role  =  not allowed
```

---

## Roles and ClusterRoles

### Namespaced Role

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: payments
  name: deployment-viewer
rules:
  - apiGroups: ["apps"]
    resources: ["deployments", "replicasets"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]          # "" is the core API group
    resources: ["pods", "events"]
    verbs: ["get", "list", "watch"]
```

A Role must declare its namespace, and it can only ever grant access inside it.

### ClusterRole

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-inspector
rules:
  - apiGroups: [""]
    resources: ["nodes", "nodes/status"]
    verbs: ["get", "list", "watch"]
  - nonResourceURLs: ["/healthz", "/metrics"]
    verbs: ["get"]
```

Use a ClusterRole when you need any of:

- cluster-scoped resources (nodes, PersistentVolumes, namespaces, CRDs)
- non-resource endpoints such as `/healthz` or `/metrics`
- namespaced resources across *every* namespace (`kubectl get pods -A`)
- a rule set you intend to reuse in many namespaces

### Aggregated ClusterRoles

A ClusterRole can be assembled automatically from others via label selectors. The controller fills in `rules` for you — never write them by hand on an aggregating role.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: monitoring
aggregationRule:
  clusterRoleSelectors:
    - matchLabels:
        rbac.example.com/aggregate-to-monitoring: "true"
rules: []   # managed by the controller
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: monitoring-endpoints
  labels:
    rbac.example.com/aggregate-to-monitoring: "true"
rules:
  - apiGroups: [""]
    resources: ["services", "endpoints"]
    verbs: ["get", "list", "watch"]
```

This is exactly how the built-in `view`, `edit`, and `admin` roles pick up permissions for newly installed CRDs.

---

## Bindings

### RoleBinding to a Role

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: view-deployments
  namespace: payments
subjects:
  - kind: User
    name: asha            # case sensitive
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: deployment-viewer
  apiGroup: rbac.authorization.k8s.io
```

### RoleBinding to a ClusterRole (scoped down)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: staging-readers
  namespace: staging      # this namespace decides the blast radius
subjects:
  - kind: Group
    name: qa-team
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: view
  apiGroup: rbac.authorization.k8s.io
```

The subject gets `view` permissions **only in `staging`**, despite the role being cluster-scoped.

### ClusterRoleBinding

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: platform-node-inspectors
subjects:
  - kind: Group
    name: platform-sre
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: node-inspector
  apiGroup: rbac.authorization.k8s.io
```

### `roleRef` is immutable

Once created, a binding's `roleRef` cannot be edited — attempts fail validation. Delete and recreate instead.

Two reasons for the rule: it lets you safely grant someone permission to edit the *subject list* without letting them silently swap in a more powerful role, and it forces a deliberate re-check of every existing subject when the granted role really does change.

`kubectl auth reconcile -f rbac.yaml` handles the delete-and-recreate dance for you when applying a manifest set.

---

## Referring to resources

Resource names in rules match the plural form used in the API URL path — `pods`, `deployments`, `configmaps`.

### Subresources

Use a slash. These are separate from the parent resource, so granting `pods` does not grant `pods/log`.

```yaml
rules:
  - apiGroups: [""]
    resources: ["pods", "pods/log", "pods/exec"]
    verbs: ["get", "list", "create"]
```

Common subresources worth knowing: `pods/log`, `pods/exec`, `pods/portforward`, `deployments/scale`, `nodes/status`.

### Named resources

Restrict a rule to specific object instances:

```yaml
rules:
  - apiGroups: [""]
    resources: ["configmaps"]
    resourceNames: ["app-config"]
    verbs: ["get", "update"]
```

Constraints to remember:

- `resourceNames` cannot restrict `deletecollection` or top-level `create` (the object name isn't known at authorization time). It *does* work on subresources like `pods/exec`.
- If you restrict `list` or `watch` by name, clients must send a matching field selector: `kubectl get configmaps --field-selector=metadata.name=app-config`.
- An empty `resourceNames` list means "no name restriction", i.e. all instances.

### Wildcards

`*` works in `apiGroups`, `resources`, `verbs`, and as a suffix glob in `nonResourceURLs`.

```yaml
rules:
  - apiGroups: ["example.com"]
    resources: ["*"]
    verbs: ["*"]        # dangerous — grants future verbs and subresources too
```

Wildcards silently expand as the cluster grows. A new CRD, subresource, or custom verb is covered the moment it appears, with no review. Enumerate verbs explicitly on anything sensitive.

### Verb cheat sheet

| Verb | HTTP | Notes |
|---|---|---|
| `get` | GET (single object) | |
| `list` | GET (collection) | grants reading every object's full content |
| `watch` | GET with `?watch` | streaming |
| `create` | POST | cannot be restricted by `resourceNames` |
| `update` | PUT | full replace |
| `patch` | PATCH | partial |
| `delete` | DELETE | |
| `deletecollection` | DELETE (collection) | cannot be restricted by `resourceNames` |

`list` is not a "safe" read verb — it returns object bodies. Granting `list` on secrets is equivalent to granting `get` on all of them.

---

## Subjects

Three kinds:

```yaml
subjects:
  - kind: User
    name: asha
    apiGroup: rbac.authorization.k8s.io

  - kind: Group
    name: system:authenticated
    apiGroup: rbac.authorization.k8s.io

  - kind: ServiceAccount
    name: ci-runner
    namespace: build            # required; no apiGroup field
```

Notes:

- Users and Groups are **not** Kubernetes objects. There is no `kubectl get users`. They are strings that come out of authentication — a client cert's `CN` and `O` fields, or OIDC claims.
- ServiceAccounts *are* real objects, and their subject entry needs a `namespace` and no `apiGroup`.
- All names are case sensitive.
- Useful built-in groups: `system:authenticated`, `system:unauthenticated`, `system:serviceaccounts`, `system:serviceaccounts:<namespace>`, `system:masters` (mapped to `cluster-admin` and effectively unrestricted — treat membership as root).

Every ServiceAccount also has a canonical username:

```
system:serviceaccount:<namespace>:<name>
```

---

## Default roles shipped with the cluster

The API server creates and continuously repairs a set of default roles at startup — this is **auto-reconciliation**. If you edit a default role, your change is reverted on the next restart unless you annotate it:

```yaml
metadata:
  annotations:
    rbacauthorization.kubernetes.io/autoupdate: "false"
```

User-facing roles, in ascending power:

| ClusterRole | Grants |
|---|---|
| `view` | read-only on most namespaced resources, **excluding** Secrets and roles/bindings |
| `edit` | everything `view` has plus create/update/delete of most objects, and read access to Secrets |
| `admin` | `edit` plus management of Roles and RoleBindings in the namespace; intended for a RoleBinding |
| `cluster-admin` | unrestricted; via ClusterRoleBinding it is superuser on everything |

Two things to be careful about: `edit` can read Secrets, and `admin` can grant permissions to others within its namespace.

Discovery roles bound to broad groups by default: `system:basic-user`, `system:discovery`, `system:public-info-viewer`.

List them all:

```bash
kubectl get clusterroles | grep -v '^system:'
kubectl describe clusterrole view
```

---

## Privilege escalation prevention

You cannot grant permissions you do not hold. The API server enforces two checks:

**Creating or editing a Role/ClusterRole** — every rule you write must already be held by you in the relevant scope, *or* you must hold the `escalate` verb on `roles`/`clusterroles`.

**Creating a binding** — you must already hold all the permissions in the referenced role, *or* hold the `bind` verb on that role.

This is why a namespace `admin` cannot bootstrap themselves into `cluster-admin`, and why your first RBAC objects on a fresh cluster must be created by a credential in `system:masters` (the default `kubeadm` admin kubeconfig).

Escape hatch when delegating safely — let a team bind a specific role without holding `escalate`:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: binder-of-app-deployer
rules:
  - apiGroups: ["rbac.authorization.k8s.io"]
    resources: ["clusterroles"]
    resourceNames: ["app-deployer"]
    verbs: ["bind"]
```

---

## Imperative commands

Fast paths — invaluable under exam time pressure. Add `--dry-run=client -o yaml` to any of them to generate a manifest.

```bash
# Role
kubectl create role pod-reader \
  --verb=get,list,watch \
  --resource=pods \
  -n payments

# Role including a subresource
kubectl create role pod-log-reader \
  --verb=get,list \
  --resource=pods,pods/log \
  -n payments

# Role limited to named objects
kubectl create role cm-updater \
  --verb=get,update \
  --resource=configmaps \
  --resource-name=app-config \
  -n payments

# ClusterRole
kubectl create clusterrole node-reader \
  --verb=get,list,watch \
  --resource=nodes

# ClusterRole for a non-resource URL
kubectl create clusterrole health-checker \
  --verb=get \
  --non-resource-url=/healthz

# RoleBinding to a Role
kubectl create rolebinding read-pods \
  --role=pod-reader \
  --user=asha \
  -n payments

# RoleBinding to a ClusterRole, scoped to one namespace
kubectl create rolebinding staging-view \
  --clusterrole=view \
  --group=qa-team \
  -n staging

# RoleBinding for a ServiceAccount
kubectl create rolebinding ci-deploy \
  --role=deployer \
  --serviceaccount=build:ci-runner \
  -n payments

# ClusterRoleBinding
kubectl create clusterrolebinding sre-node-read \
  --clusterrole=node-reader \
  --group=platform-sre

# Apply a manifest set, recreating bindings whose roleRef changed
kubectl auth reconcile -f rbac/
```

`--resource-name` is not available on `create clusterrole` for every case, and `create rolebinding` takes `--user`, `--group`, or `--serviceaccount=<ns>:<name>` (repeatable).

---

## Verifying permissions

`kubectl auth can-i` is the fastest way to close the loop on any RBAC change.

```bash
# As yourself
kubectl auth can-i create deployments -n payments

# As someone else (needs impersonation rights)
kubectl auth can-i list secrets -n payments --as=asha
kubectl auth can-i '*' '*' --as=asha

# As a ServiceAccount
kubectl auth can-i get pods -n payments \
  --as=system:serviceaccount:build:ci-runner

# Everything a subject can do
kubectl auth can-i --list -n payments --as=asha

# Who am I authenticated as?
kubectl auth whoami
```

Inspect the objects themselves:

```bash
kubectl describe clusterrole edit
kubectl get rolebindings,clusterrolebindings -A -o wide

# Every binding that references a given ClusterRole
kubectl get clusterrolebinding -o json \
  | jq -r '.items[] | select(.roleRef.name=="cluster-admin") | .metadata.name'
```

---

## ServiceAccount patterns

Every namespace has a `default` ServiceAccount, and every Pod gets one mounted unless told otherwise. The default account has essentially no permissions — which is correct, and should stay that way.

**Opt out when the Pod never calls the API:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: worker
spec:
  automountServiceAccountToken: false
  containers:
    - name: app
      image: myapp:1.0
```

**Dedicated account per workload:**

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: metrics-reader
  namespace: monitoring
automountServiceAccountToken: false
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: pod-metrics-reader
rules:
  - apiGroups: [""]
    resources: ["pods", "nodes"]
    verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: metrics-reader-binding
subjects:
  - kind: ServiceAccount
    name: metrics-reader
    namespace: monitoring
roleRef:
  kind: ClusterRole
  name: pod-metrics-reader
  apiGroup: rbac.authorization.k8s.io
```

Reference it from the Pod spec with `serviceAccountName: metrics-reader`.

Get a short-lived token for testing:

```bash
kubectl create token metrics-reader -n monitoring --duration=10m
```

---

## Troubleshooting

**`Error from server (Forbidden): ... is forbidden: User "x" cannot list resource "pods" in API group "" in the namespace "y"`**

Read the message literally — it names the user, the verb, the resource, the API group, and the namespace. Nearly always one of these is off by a character.

Checklist:

1. **Is there a binding at all?** A Role with no binding does nothing.
   `kubectl get rolebinding -n <ns> -o wide`
2. **Right namespace?** For a RoleBinding, the binding's *own* namespace determines scope — not the Role's, not the subject's.


4. **API group correct?** Deployments are in `apps`, not `""`. Pods, Services, ConfigMaps, Secrets are in `""`.
5. **Subject spelled and typed correctly?** Names are case sensitive; `ServiceAccount` subjects need `namespace`, `User`/`Group` need `apiGroup`.
6. **Missing subresource?** `kubectl logs` needs `pods/log`, `kubectl exec` needs `pods/exec`, `kubectl scale` needs `deployments/scale`.
7. **Missing verb?** `kubectl get pods` needs `list`, not just `get`. `kubectl apply` typically needs `get`, `patch`, and `create`.
8. **Cluster-scoped resource bound with a RoleBinding?** Nodes and PVs need a ClusterRoleBinding.
9. **Escalation blocked?** If *your own* create of a Role fails, you're likely trying to grant permissions you don't hold.

Then confirm the fix:

```bash
kubectl auth can-i <verb> <resource> -n <ns> --as=<subject>
```

Audit logs are the last resort — set `--audit-policy-file` on the API server and look for `RBAC: allowed by ...` / `RBAC DENY` reason annotations.

---

## Good practices

- **Least privilege by default.** Start from `view`, add specific verbs, never start from `cluster-admin` and trim.
- **Prefer namespaced Roles.** Reach for ClusterRole only when the resource or the scope genuinely requires it.
- **Bind to groups, not users.** Individual bindings rot as people join and leave.
- **Treat Secrets access as privileged.** `get`, `list`, and `watch` on secrets all leak contents. Remember `edit` includes it.
- **Avoid wildcards** on `resources` and `verbs` for anything sensitive — they silently absorb future additions.
- **Audit ClusterRoleBindings to `cluster-admin` regularly.** That list should be short and every entry explainable.
- **One ServiceAccount per workload**, and disable token automount where the Pod never calls the API.
- **Watch escalation-adjacent permissions**: `escalate`, `bind`, `impersonate`, `create` on `pods` (mount any secret), and write access to `nodes/proxy`.
- **Keep RBAC in Git** and roll it out with `kubectl auth reconcile` so immutable `roleRef` changes are handled cleanly.

---

## Further reading

- [Using RBAC Authorization](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
- [RBAC Good Practices](https://kubernetes.io/docs/concepts/security/rbac-good-practices/)
- [Controlling Access to the Kubernetes API](https://kubernetes.io/docs/concepts/security/controlling-access/)
- [Authorization Overview](https://kubernetes.io/docs/reference/access-authn-authz/authorization/)
- [ServiceAccounts](https://kubernetes.io/docs/concepts/security/service-accounts/)
  ###

  
   <img width="1024" height="1536" alt="RBAC" src="https://github.com/user-attachments/assets/a99e7b1c-6e0b-4f10-9978-b1ff504d1546" />

<img width="1536" height="1024" alt="RBAC2 2" src="https://github.com/user-attachments/assets/a2d86c69-f7a7-49db-bac6-a0a2f453e04b" />
