# Argo Workflows concepts — a working tutorial

Written while getting every workflow in [`workflows/`](../workflows/),
[`templates/`](../templates/), and [`cron/`](../cron/) to actually reach
`Succeeded` on a real [kind](https://kind.sigs.k8s.io/) cluster running the
Argo Workflows `quick-start-minimal.yaml` install (controller + server +
in-cluster MinIO). It's organized by concept; each section links to the
example and calls out what wasn't obvious from the upstream docs.

## Setup gotcha #1: `kubectl apply` chokes on the CRDs

The very first install attempt failed:

```
Error from server (Invalid): error when creating ".../quick-start-minimal.yaml":
CustomResourceDefinition.apiextensions.k8s.io "workflows.argoproj.io" is invalid:
metadata.annotations: Too long: must have at most 262144 bytes
```

`kubectl apply` (client-side) stores the full manifest it applied in a
`kubectl.kubernetes.io/last-applied-configuration` annotation, capped at
256KiB. Argo's `Workflow` and `WorkflowTemplate` CRDs embed a large OpenAPI
schema and blow past that cap. Fix: use server-side apply, which doesn't
maintain that annotation at all:

```bash
kubectl apply -n argo --server-side -f quick-start-minimal.yaml
```

## Container templates ([`hello-container`](../workflows/hello-container/workflow.yaml))

The minimal unit: one `container:` block under `templates:`, referenced by
`spec.entrypoint`. Nothing else is required. Everything below builds on this.

## Script templates ([`hello-script`](../workflows/hello-script/workflow.yaml))

A `script:` template is a `container:` template plus one convenience: the
`source:` string is written to a file inside the container and executed with
`command` as the interpreter (`python`, `sh`, `node`, ...). Useful once your
inline logic is more than a one-liner — no need to `COPY` a script into a
custom image just to run a few lines of Python.

## Steps templates ([`hello-steps`](../workflows/hello-steps/workflow.yaml))

`steps:` is a list of lists: each inner list (`- - name: a` / `- name: b`)
runs **in parallel**; each outer list entry runs **sequentially** after the
previous one finishes. `hello-steps` has two outer entries, each with one
step, so it's a strict A-then-B sequence.

To pass a value forward, the producing template writes it to a file and
declares an output parameter that reads that file:

```yaml
outputs:
  parameters:
    - name: greeting
      valueFrom:
        path: /tmp/greeting.txt
```

The consumer references it with `{{steps.<step-name>.outputs.parameters.<param-name>}}`.
This worked first try — the main gotcha is just remembering the step name in
the reference is the **step's** name (`generate-greeting`), not the
**template's** name (`generate`).

## DAG templates ([`hello-dag`](../workflows/hello-dag/workflow.yaml))

Same idea as Steps, but you declare `dependencies: [task-a, task-b]` on each
task explicitly instead of relying on list position. This is what makes
fan-out/fan-in possible: two tasks can both list the same single
`dependencies: [start]`, so they run in parallel once `start` finishes, and a
final task lists `dependencies: [fan-out-a, fan-out-b]` to wait for both
before running. Argo builds the DAG from these edges — order in the YAML
doesn't matter, only the `dependencies` lists do.

## Loops: `withItems` and `withParam` ([`hello-loops`](../workflows/hello-loops/workflow.yaml))

- `withItems:` takes a **static, inline** YAML list known at authoring time.
- `withParam:` takes a **string produced at runtime** — normally a prior
  step's `{{steps.X.outputs.result}}` (the step's stdout) which must be valid
  JSON (a JSON array). Argo parses it and fans out over the elements exactly
  like `withItems`.

Both produce parallel task instances, each with `{{item}}` bound to one
element. `withParam` is the one to reach for when the list itself is
computed by an earlier step (query a DB, list files, call an API) rather than
known up front.

## Conditionals: `when:` ([`hello-conditional`](../workflows/hello-conditional/workflow.yaml))

`when:` is a plain string expression evaluated with the prior step/task
outputs already substituted in — `"{{steps.decide.outputs.result}} == heads"`.
If it evaluates false, Argo marks that step `Omitted`/skipped (shown as `○`
in `argo get`), not failed, and the workflow can still succeed overall. This
is different from a task that fails a condition inside its own script and
exits non-zero — that would actually fail the step.

## Retries ([`hello-retry`](../workflows/hello-retry/workflow.yaml))

`retryStrategy.limit` bounds the number of extra attempts; `backoff` controls
the delay between them. The genuinely useful bit for **writing** a retry demo
is the built-in `{{retries}}` variable, available inside a template that has
a `retryStrategy` — it's the current attempt number, **0-indexed**. The demo
script fails while `{{retries}} < 2` and succeeds on attempt `2`, so
`argo get` shows exactly what you'd want to see when debugging retries in
production: two red `✖` attempts followed by one green `✔`.

`activeDeadlineSeconds` on the template caps total wall-clock time across all
attempts combined — set it generously enough to cover `limit` attempts at
worst-case `backoff`, or the deadline can fire before the retries you asked
for are exhausted.

## Artifacts ([`hello-artifacts`](../workflows/hello-artifacts/workflow.yaml))

This is the one that needs a real artifact repository, not just parameter
plumbing. The quick-start install already wires one up — its
`artifact-repositories` ConfigMap points at the in-cluster MinIO deployment
(bucket `my-bucket`) — so no extra configuration was needed here; a plain
`outputs.artifacts` / `inputs.artifacts` pair on two steps was enough:

```yaml
outputs:
  artifacts:
    - name: message
      path: /tmp/out/message.txt
# ...
inputs:
  artifacts:
    - name: message
      path: /tmp/in/message.txt
```

Proof it's real, not simulated: after the run,
`kubectl -n argo exec deploy/minio -- ls /data/my-bucket/` shows a directory
per workflow name, and the consuming step's logs print the exact string
written by the producing step, fetched back from MinIO in between.

**Gotcha**: every workflow — not just ones using `outputs.artifacts` —
writes into that same MinIO bucket, because the quick-start config sets
`archiveLogs: true`. Don't be surprised to see a `my-bucket/<workflow-name>/`
folder for workflows that never mention "artifact" in their YAML at all;
that's just the archived pod logs.

## Exit handlers ([`hello-exit-handler`](../workflows/hello-exit-handler/workflow.yaml))

`spec.onExit: <template-name>` runs that template exactly once after the
main DAG/Steps finishes, whether it succeeded or failed. Inside the handler,
`{{workflow.status}}` resolves to `Succeeded`/`Failed`/`Error`, which is
enough to fan out a notification step by outcome. The example's main step
succeeds (so the *workflow* — and this lab's "everything reaches Succeeded"
bar — stays green); flip its `exit 0` to `exit 1` locally to watch the exact
same handler fire on the `Failed` path instead. It does — this was verified
during development before being reverted to the always-green version checked
into this repo.

## Resource templates ([`hello-resource`](../workflows/hello-resource/workflow.yaml)) — the RBAC gotcha

A `resource:` template (`action: create|patch|delete|get|apply`) talks to
the Kubernetes API directly — Argo's executor does the equivalent of
`kubectl create/patch/...` in-process, no extra container needed. This is
also the one template type that doesn't run as whatever ServiceAccount the
controller uses; it runs as **the Workflow's own `serviceAccountName`**. The
default install's `default` ServiceAccount has no permissions at all, so
this example ships its own `rbac.yaml` with a scoped ServiceAccount.

First attempt failed anyway, with a confusing symptom: the ConfigMap
actually got created (`kubectl -n argo get configmap` showed it existed with
the right data), yet the workflow step reported `Failed`:

```
main: Error (exit code 64): workflowtaskresults.argoproj.io is forbidden:
User "system:serviceaccount:argo:hello-resource-sa" cannot create resource
"workflowtaskresults" in API group "argoproj.io" in the namespace "argo"
```

The root cause: **every** pod's executor — regardless of template type —
calls back to the API server to report its result via the `WorkflowTaskResult`
CRD. That's a separate permission from whatever the template's own action
needs. The quick-start install already defines a Role named `executor` with
exactly this permission (`workflowtaskresults`: `create`, `patch`); the fix
was one more `RoleBinding` pointing our custom ServiceAccount at that
existing Role, rather than inventing the rule from scratch:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: hello-resource-executor
subjects:
  - kind: ServiceAccount
    name: hello-resource-sa
roleRef:
  kind: Role
  name: executor
  apiGroup: rbac.authorization.k8s.io
```

**Lesson**: if a `resource:` (or any) template's side effect visibly happens
but the step still reports `Failed`, check for `workflowtaskresults`
permission errors in the pod logs before assuming the resource action itself
is broken.

The second gotcha in this example: `resource` templates don't automatically
expose a `name` output parameter just because the manifest used
`generateName`. It must be declared explicitly with a `jsonPath`:

```yaml
outputs:
  parameters:
    - name: name
      valueFrom:
        jsonPath: '{.metadata.name}'
```

`argo lint --offline` catches a reference to an undeclared output before you
ever get to the cluster — `templates.create-configmap.outputs.parameters.name`
failed to resolve until this was added.

## WorkflowTemplate vs. ClusterWorkflowTemplate

See [`templates/cluster-workflow-template/README.md`](../templates/cluster-workflow-template/README.md)
for the scope trade-off. Mechanically, invoking either looks the same
(`templateRef:` inside a step/task) except a `ClusterWorkflowTemplate`
reference needs `clusterScope: true`:

```yaml
templateRef:
  name: cluster-greeter
  template: greet
  clusterScope: true   # omit this and Argo looks for a *namespaced*
                        # WorkflowTemplate named cluster-greeter instead,
                        # and the reference fails to resolve
```

## CronWorkflow ([`hello-cron`](../cron/hello-cron/cronworkflow.yaml)) — a schema surprise

`argo lint --offline` rejected the first draft:

```
strict decoding error: unknown field "spec.schedule"
```

The `argo` CLI installed for this lab is v4.0.7. In that version
`CronWorkflowSpec` uses `schedules:` (a **list** of cron strings, supporting
multiple schedules per CronWorkflow) — the older singular `schedule:` field
from earlier Argo versions doesn't exist in this schema. If you're following
older blog posts or examples, check which field your installed CLI/CRD
version actually expects:

```yaml
spec:
  schedules:
    - "*/5 * * * *"
  timezone: "UTC"
```

To verify a `CronWorkflow` works without waiting for the clock:
`argo submit --from cronworkflow/<name> --wait` builds and runs the same
`workflowSpec` immediately. This lab's cron was verified both ways — once
via `--from` and once by leaving it unsuspended long enough to watch a real
scheduled tick (`hello-cron-<timestamp>`) fire and succeed on its own — then
suspended (`argo cron suspend hello-cron`) so it doesn't keep submitting
workflows every 5 minutes in the background afterward.

## A real-world example: nightly Postgres backups ([`db-backup-postgres`](../cron/db-backup-postgres/cronworkflow.yaml))

Everything above is a minimal, single-concept demo. This one composes several
of them into something closer to what you'd actually run in production: a
nightly `CronWorkflow` that `pg_dump`s a real Postgres instance
([`postgres.yaml`](../cron/db-backup-postgres/postgres.yaml) deploys one,
seeded with a `widgets` table via `/docker-entrypoint-initdb.d`) and uploads
the dump as an artifact.

A few choices here are deliberately different from the earlier examples, and
worth calling out because they're the kind of thing that actually matters for
a backup job specifically:

- **`pg_dump --format=custom` instead of piping through `gzip`.** Postgres's
  custom format is already compressed and — unlike a raw SQL dump — restorable
  selectively (`pg_restore --list` / `pg_restore -t widgets`) without piping
  the whole file through `psql` again. One less moving part than shelling out
  to `gzip` separately, and closer to what you'd actually script for a real
  database.
- **A dedicated `verify` step, not just a non-zero exit code.** `pg_dump`
  exiting 0 tells you the connection worked and it wrote *something*; it
  doesn't tell you the file is a valid, restorable archive. `verify` reruns
  `pg_restore --list` against the artifact — parsing the archive's own table
  of contents — so a truncated or corrupted dump fails the workflow instead
  of silently becoming "the backup" for the day. Confirmed for real: the
  logs from the first run show `pg_restore --list` printing the actual TOC —
  `TABLE public widgets`, `TABLE DATA public widgets`, the sequence and
  primary key — not a stub.
- **`concurrencyPolicy: Forbid`, not `Replace`** (unlike `hello-cron`). If a
  backup is still dumping when the next scheduled tick arrives, you want the
  new run skipped, not the in-flight dump killed partway through.
- **`retryStrategy` on `dump` is a real retry, not the artificial-failure
  trick from the Retries section above (`hello-retry`).** A couple of quick
  retries with backoff covers a transient connection blip
  to the database without masking a persistent problem (`limit: "2"` still
  fails the workflow if Postgres is actually down).

Nothing new broke getting this one green — the `workflowtaskresults` RBAC
requirement and the `--server-side` apply gotcha from earlier sections apply
here too (this CronWorkflow uses the default ServiceAccount, which already
has the `executor` Role bound cluster-wide by the quick-start install, so no
extra RBAC file was needed this time — that requirement only bites
`resource:` templates with a *custom* ServiceAccount, like `hello-resource`).
Verified by triggering an immediate run (`argo submit --from
cronworkflow/db-backup-postgres --wait`), confirming `Succeeded`, and
checking the artifact actually landed in MinIO
(`kubectl -n argo exec deploy/minio -- ls /data/my-bucket/<run-name>/`)
before suspending the schedule the same way as `hello-cron`.

## Shared volumes between steps, and Vault-injected secrets ([`export-csv-to-cos`](../workflows/export-csv-to-cos/workflow.yaml))

Every earlier example either used a single step, or passed data between
steps as a parameter/artifact. This one needs an actual shared filesystem
between two steps — Postgres tables get exported to CSV files in
`export-csv`, then `upload-cos` reads those same files back to push them to
an S3-compatible bucket with `mc`. It also swaps hardcoded/Secret-based
credentials for Vault, injected via the [Vault Agent
Injector](https://developer.hashicorp.com/vault/docs/platform/k8s/injector) —
setup in [vault-setup.md](../workflows/export-csv-to-cos/vault-setup.md),
verified against a real (dev-mode) Vault + injector installed via Helm into
this lab's kind cluster, not just written up as theory.

### Sharing a volume across steps means a PVC, not `emptyDir`

Each step in a Steps/DAG template is its own Pod, scheduled independently -
unlike a multi-container Pod (e.g. a `sidecar`), an `emptyDir` on one step's
Pod is gone by the time the next step's Pod starts. The mechanism that
actually persists a filesystem across steps is `spec.volumeClaimTemplates`
at the *workflow* level:

```yaml
spec:
  volumeClaimTemplates:
    - metadata:
        name: work
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 256Mi
```

Argo provisions this PVC once per workflow run and **automatically makes a
volume named `work` available to every template's Pod** - the only thing a
template needs to do is mount it:

```yaml
volumeMounts:
  - name: work
    mountPath: /work
```

**Gotcha actually hit**: the first draft of this workflow also declared an
explicit `volumes: [{name: work, persistentVolumeClaim: {claimName: work}}]`
block in each template, on the assumption you always have to declare where a
`volumeMounts` name comes from. That's correct for `configMap`/`secret`
volumes, but wrong here — it points at a PVC literally named `work`, which
never exists (the real one Argo creates is named `<workflow-name>-work`).
The pod sat `Pending` with `persistentvolumeclaim "work" not found` until
the redundant `volumes:` block was deleted from both templates; Argo's
auto-injected volume (from `volumeClaimTemplates`) was already correct and
just needed to be left alone.

The PVC is automatically deleted once the workflow finishes — confirmed by
checking `kubectl -n argo get pvc` immediately after a `Succeeded` run and
finding it already gone, with no cleanup step of our own.

### Vault Agent Injector: two things that aren't obvious from the annotations alone

The injector works entirely off pod annotations — no Argo-specific
integration exists or is needed, since Argo's executor doesn't care what a
mutating webhook added to the Pod spec before it was scheduled:

```yaml
metadata:
  annotations:
    vault.hashicorp.com/agent-inject: "true"
    vault.hashicorp.com/agent-pre-populate-only: "true"
    vault.hashicorp.com/role: "export-csv-cos-pg"
    vault.hashicorp.com/agent-inject-secret-db-creds.sh: "secret/data/db/postgres"
    vault.hashicorp.com/agent-inject-template-db-creds.sh: |
      {{- with secret "secret/data/db/postgres" -}}
      export PGHOST="{{ .Data.data.host }}"
      ...
      {{- end -}}
```

1. **`agent-pre-populate-only: "true"` is what makes this compatible with
   Argo at all.** Without it, the injector adds its usual long-running
   `vault-agent` *sidecar* container alongside `main` - and Argo's executor
   waits for every container in the pod to exit before marking the step
   done. A sidecar that's designed to run forever (to keep renewing/
   re-rendering the secret) means the step Pod never completes. With this
   annotation, the injector instead runs Vault Agent as an **init
   container only** - it fetches the secret, renders the template to
   `/vault/secrets/db-creds.sh`, and exits, exactly like any other init
   container. This is the standard fix for one-shot workloads (Argo steps,
   Kubernetes `Job`s, CI runners) — anything that isn't a long-lived
   Deployment.
2. **The `{{ }}` in the Vault template annotation didn't collide with
   Argo's own `{{ }}` variable substitution, but it was worth verifying
   rather than assuming.** Argo uses the same delimiter for its own
   variables (`{{workflow.name}}`, `{{inputs.parameters.x}}`); the concern
   going in was that Argo might try to resolve `{{ .Data.data.host }}` as
   an (unresolvable) Argo variable and fail lint or submission. It didn't -
   `argo lint --offline` passed and the workflow ran cleanly with the
   annotation inlined exactly as shown above, because Argo only substitutes
   recognized variable prefixes and leaves anything else untouched. See
   vault-setup.md for the fallback (`vault.hashicorp.com/agent-configmap`)
   if a different Argo version ever behaves differently here.

Sourcing the rendered file at the top of each step's script
(`. /vault/secrets/db-creds.sh`) is what actually gets the credentials into
the shell's environment — the injector's job ends at "write a file inside
the pod," reading it is on the workload like any other init-container output.

### Least privilege, enforced by *which ServiceAccount ran the pod*

`export-csv` and `upload-cos` each run under their own ServiceAccount
(`export-csv-cos-pg-sa`, `export-csv-cos-upload-sa`), each bound to a
different Vault Kubernetes-auth role, each of which maps to a policy that
can read exactly one secret path. The `upload-cos` step's pod is
*structurally incapable* of reading the Postgres credentials — it has no
Vault role that grants it — the same principle as the Resource templates
section above (`hello-resource`) scoping Kubernetes RBAC per-workflow, just
enforced by Vault instead of the Kubernetes API server. And, same lesson as
`hello-resource`: both of these
custom ServiceAccounts still needed the `executor` Role bound via
`rbac.yaml`, or the pods fail on `workflowtaskresults.argoproj.io is
forbidden` regardless of whether Vault injection itself works correctly.

## One more thing that isn't a "gotcha" so much as a trap for the unwary: retention

The quick-start `workflow-controller-configmap` ships with:

```yaml
retentionPolicy:
  completed: 10
  failed: 3
  errored: 3
```

The controller garbage-collects `Workflow` custom resources past those
counts. Partway through verifying every workflow in this lab, `argo list`
stopped showing `hello-container`, `hello-script`, and `hello-steps` even
though each had individually printed `Succeeded` when submitted — they'd
simply aged out once more than 10 completed workflows existed. This is
expected controller behavior, not a bug: each workflow's success was already
confirmed at submit time (`argo submit --wait` printed
`<name> Succeeded at <timestamp>` for every single one), it's just that
`argo list`/`argo get` afterward can't show you a workflow the controller has
already pruned. For a lab you want to keep browsing in the UI, raise
`retentionPolicy.completed` or check the archived-workflows API/Postgres
backend if you've configured one — neither was needed here since the
CLI output at submit time was already the source of truth.
