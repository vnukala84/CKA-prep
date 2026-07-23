RBAC (Role-Based Access Control) is the Kubernetes authorization mechanism that controls who can do what on which resources. Every API request is checked against RBAC before being processed.

There are four objects to understand: Subject (who), Role/ClusterRole (what permissions), and RoleBinding/ClusterRoleBinding (the glue between them).

<img width="595" height="143" alt="image" src="https://github.com/user-attachments/assets/d666d6f3-16e1-4fd5-a2cc-0884f32cdaa3" />
