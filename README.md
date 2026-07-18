# hello-world-workflows-lab

A hands-on lab for learning [Argo Workflows](https://argoproj.github.io/argo-workflows/)
concepts, one minimal example at a time. Every manifest here uses a real,
pull-able container image (`busybox`, `python:3.12-alpine`, `curlimages/curl`)
and actually runs end-to-end on a local [kind](https://kind.sigs.k8s.io/)
cluster — nothing is pseudocode.

## Structure

```
hello-world-workflows-lab/
├── workflows/            one directory per Workflow concept
├── templates/             a WorkflowTemplate and a ClusterWorkflowTemplate
├── cron/                  a CronWorkflow example
├── ci/validate.yaml        GitHub Actions job: argo lint on every PR
└── docs/                  tutorial write-up of what wasn't obvious
```

## Prerequisites (local run against kind)

- [Docker](https://www.docker.com/)
- [kind](https://kind.sigs.k8s.io/)
- [kubectl](https://kubernetes.io/docs/tasks/tools/)
- [Argo Workflows CLI](https://github.com/argoproj/argo-workflows/releases) (`argo`)

```bash
kind create cluster --name argo-lab
kubectl create namespace argo
# --server-side avoids "annotations: Too long" errors client-side apply hits
# on the Workflow/WorkflowTemplate CRDs — see docs/argo-workflows-concepts.md
kubectl apply -n argo --server-side -f https://github.com/argoproj/argo-workflows/releases/latest/download/quick-start-minimal.yaml
kubectl -n argo wait --for=condition=available --timeout=300s deploy --all
kubectl -n argo port-forward svc/argo-server 2746:2746 &
export ARGO_SECURE=false
```

The `quick-start-minimal.yaml` manifest installs the workflow-controller, the
argo-server, and an in-cluster MinIO instance pre-wired as the default
artifact repository — everything `hello-artifacts` needs, with no extra
configuration. `workflows/hello-resource` additionally needs its own RBAC
applied first: `kubectl -n argo apply -f workflows/hello-resource/rbac.yaml`
(see the submit command in the table below and
[docs/argo-workflows-concepts.md](docs/argo-workflows-concepts.md) for why).

## Workflows

All commands assume you're in this directory and `ARGO_SECURE=false` is set
per above. Add `-n argo` if your default kubeconfig context isn't already
pointed at the `argo` namespace.

| Path | Concept | Submit command |
|---|---|---|
| [`workflows/hello-container`](workflows/hello-container/workflow.yaml) | Simplest possible Workflow — one container template | `argo submit --watch workflows/hello-container/workflow.yaml` |
| [`workflows/hello-script`](workflows/hello-script/workflow.yaml) | Script template (inline interpreted source) | `argo submit --watch workflows/hello-script/workflow.yaml` |
| [`workflows/hello-steps`](workflows/hello-steps/workflow.yaml) | Steps template — sequential stages, output param passed forward | `argo submit --watch workflows/hello-steps/workflow.yaml` |
| [`workflows/hello-dag`](workflows/hello-dag/workflow.yaml) | DAG template — explicit `dependencies:`, fan-out/fan-in | `argo submit --watch workflows/hello-dag/workflow.yaml` |
| [`workflows/hello-loops`](workflows/hello-loops/workflow.yaml) | `withItems` (static list) and `withParam` (runtime JSON list) | `argo submit --watch workflows/hello-loops/workflow.yaml` |
| [`workflows/hello-conditional`](workflows/hello-conditional/workflow.yaml) | `when:` expression skipping a step based on prior output | `argo submit --watch workflows/hello-conditional/workflow.yaml` |
| [`workflows/hello-retry`](workflows/hello-retry/workflow.yaml) | `retryStrategy` + `activeDeadlineSeconds`, fails then recovers | `argo submit --watch workflows/hello-retry/workflow.yaml` |
| [`workflows/hello-artifacts`](workflows/hello-artifacts/workflow.yaml) | Output/input artifacts via the MinIO artifact repository | `argo submit --watch workflows/hello-artifacts/workflow.yaml` |
| [`workflows/hello-exit-handler`](workflows/hello-exit-handler/workflow.yaml) | `onExit` handler, runs regardless of success/failure | `argo submit --watch workflows/hello-exit-handler/workflow.yaml` |
| [`workflows/hello-resource`](workflows/hello-resource/workflow.yaml) | `resource` template creating/patching a ConfigMap directly | `kubectl apply -f workflows/hello-resource/rbac.yaml && argo submit --watch workflows/hello-resource/workflow.yaml` |
| [`templates/workflow-template`](templates/workflow-template/template.yaml) | Reusable `WorkflowTemplate` invoked via `templateRef` | `kubectl apply -f templates/workflow-template/template.yaml && argo submit --watch templates/workflow-template/consumer-workflow.yaml` |
| [`templates/cluster-workflow-template`](templates/cluster-workflow-template/template.yaml) | Cluster-scoped `ClusterWorkflowTemplate` (see its [README](templates/cluster-workflow-template/README.md) for when to pick this over a `WorkflowTemplate`) | `kubectl apply -f templates/cluster-workflow-template/template.yaml && argo submit --watch templates/cluster-workflow-template/consumer-workflow.yaml` |
| [`cron/hello-cron`](cron/hello-cron/cronworkflow.yaml) | `CronWorkflow` on a schedule, reusing the `greeter` `WorkflowTemplate` | `kubectl apply -f templates/workflow-template/template.yaml && kubectl apply -f cron/hello-cron/cronworkflow.yaml` (then `argo cron list` / wait for the schedule, or `argo submit --from cronworkflow/hello-cron --watch` to trigger one run immediately) |

## CI

[`ci/validate.yaml`](ci/validate.yaml) is a GitHub Actions workflow definition
that runs `argo lint --offline` over every manifest under `workflows/`,
`templates/`, and `cron/` on every PR. GitHub only picks up workflow
definitions from `.github/workflows/`, so activate it with:

```bash
mkdir -p .github/workflows
cp ci/validate.yaml .github/workflows/validate.yaml
```

It's kept at `ci/validate.yaml` in-tree (rather than directly under
`.github/workflows/`) so it reads as part of this lab's own structure; copy
or symlink it in once you push this repo to GitHub.

## Further reading

[`docs/argo-workflows-concepts.md`](docs/argo-workflows-concepts.md) is a
tutorial covering each concept above plus the gotchas actually hit while
getting these workflows to pass on a real cluster.
