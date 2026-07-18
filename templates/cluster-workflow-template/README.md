# WorkflowTemplate vs. ClusterWorkflowTemplate

Both are libraries of reusable templates invoked via `templateRef` — the only
difference is scope.

| | `WorkflowTemplate` | `ClusterWorkflowTemplate` |
|---|---|---|
| Scope | One namespace | Whole cluster |
| Who can invoke it | Workflows in the same namespace | Workflows in any namespace |
| RBAC surface | Smaller — bound to a namespace's Role | Larger — needs a ClusterRole |
| Typical owner | A team that owns the namespace | Platform/infra team |

**Use a `WorkflowTemplate`** when the template is specific to one team or
application — e.g. "the build pipeline for service X" that only lives in
`team-x`'s namespace. This is the default choice; start here.

**Use a `ClusterWorkflowTemplate`** when you're publishing a shared building
block that many teams/namespaces should be able to invoke without copying
YAML around — e.g. a company-standard "notify Slack on failure" template, a
vulnerability-scan step, or a shared CI utility. Because it's visible
cluster-wide, treat changes to it like changes to a shared library: they can
affect every namespace that references them, so guard write access to
`ClusterWorkflowTemplate` objects with RBAC more tightly than you would a
namespaced one.
