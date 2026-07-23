RBAC (Role-Based Access Control) is the Kubernetes authorization mechanism that controls who can do what on which resources. Every API request is checked against RBAC before being processed.

There are four objects to understand: Subject (who), Role/ClusterRole (what permissions), and RoleBinding/ClusterRoleBinding (the glue between them).
