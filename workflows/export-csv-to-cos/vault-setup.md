# Vault setup for `export-csv-to-cos`

One-time cluster setup this workflow depends on. Everything here was run
against a real Vault **dev-mode** server + Agent Injector installed via Helm
into the same kind cluster as the rest of this lab, and the workflow was
verified end-to-end against it — this isn't aspirational documentation.

> **Dev mode only.** `server.dev.enabled=true` and a hardcoded root token are
> for this lab's disposable kind cluster, never for anything real. Production
> Vault needs real unsealing, storage, and auth beyond the scope of this repo.

## 1. Install Vault + the Agent Injector

```bash
helm repo add hashicorp https://helm.releases.hashicorp.com
helm repo update hashicorp
kubectl create namespace vault
helm install vault hashicorp/vault \
  --namespace vault \
  --set "server.dev.enabled=true" \
  --set "server.dev.devRootToken=root" \
  --set "injector.enabled=true" \
  --wait --timeout=180s
```

This deploys two things that matter here: the `vault-0` dev server, and
`vault-agent-injector` — a mutating admission webhook that watches for pods
carrying `vault.hashicorp.com/agent-inject: "true"` and adds init/sidecar
containers to them that fetch secrets and write them to files.

## 2. Enable and configure Kubernetes auth

```bash
kubectl -n vault exec vault-0 -- sh -c '
  export VAULT_TOKEN=root
  vault auth enable kubernetes
  vault write auth/kubernetes/config \
    kubernetes_host="https://$KUBERNETES_SERVICE_HOST:$KUBERNETES_SERVICE_PORT_HTTPS"
'
```

Because Vault itself runs inside the cluster it's authenticating against,
the in-cluster `$KUBERNETES_SERVICE_HOST`/`_PORT_HTTPS` env vars (present in
every pod) are enough — no separate reviewer token or CA bundle to wire up
by hand for this lab.

## 3. Least-privilege policies, one per secret path

```bash
kubectl -n vault exec vault-0 -- sh -c '
  export VAULT_TOKEN=root
  vault policy write export-csv-cos-pg - <<EOF
path "secret/data/db/postgres" {
  capabilities = ["read"]
}
EOF
  vault policy write export-csv-cos-upload - <<EOF
path "secret/data/cos/ibm" {
  capabilities = ["read"]
}
EOF
'
```

## 4. Kubernetes-auth roles bound to each step's ServiceAccount

```bash
kubectl -n vault exec vault-0 -- sh -c '
  export VAULT_TOKEN=root
  vault write auth/kubernetes/role/export-csv-cos-pg \
    bound_service_account_names=export-csv-cos-pg-sa \
    bound_service_account_namespaces=argo \
    policies=export-csv-cos-pg \
    ttl=15m
  vault write auth/kubernetes/role/export-csv-cos-upload \
    bound_service_account_names=export-csv-cos-upload-sa \
    bound_service_account_namespaces=argo \
    policies=export-csv-cos-upload \
    ttl=15m
'
```

The `export-csv` step's pod can only ever read `secret/data/db/postgres`;
the `upload-cos` step's pod can only ever read `secret/data/cos/ibm` — each
because of *which ServiceAccount ran the pod*, not because of anything in
the Workflow YAML itself. `rbac.yaml` in this directory creates both
ServiceAccounts (plus the `executor` RoleBinding every custom ServiceAccount
in this lab needs — see docs/argo-workflows-concepts.md).

## 5. Seed the secrets

`secret/db/postgres` intentionally holds credentials for `export_csv_reader`,
**not** `labuser`. `labuser` is the Postgres bootstrap user and is a
superuser by default (`\du labuser` will show `Superuser, Create role,
Create DB, Replication, Bypass RLS`) — confirmed the hard way, not assumed.
`export-csv` only ever runs `SELECT`, and `queries` is a workflow parameter
containing arbitrary SQL, so there's no reason for that pod to hold
superuser credentials merely because it's convenient. `export_csv_reader` is
created by `cron/db-backup-postgres/postgres.yaml`'s
`zz-create-reader-role.sh` init script with exactly `SELECT` on the schema
(plus `ALTER DEFAULT PRIVILEGES` so it also covers tables added later) —
verified directly: connecting as `export_csv_reader` and running `SELECT`
works, `INSERT`/`DROP TABLE` both fail with `permission denied` /
`must be owner of table`.

```bash
kubectl -n vault exec vault-0 -- sh -c '
  export VAULT_TOKEN=root
  vault kv put secret/db/postgres \
    host=postgres port=5432 \
    username=export_csv_reader password=export-reader-password-change-me database=labdb

  vault kv put secret/cos/ibm \
    endpoint=http://minio.argo.svc.cluster.local:9000 \
    access_key=admin secret_key=password bucket=csv-exports
'
```

In production, swap the `cos/ibm` values for a real IBM COS HMAC key pair
and endpoint (e.g. `https://s3.us-south.cloud-object-storage.appdomain.cloud`)
— nothing else about the workflow changes, since `mc` and the injector
templates don't know or care that MinIO is standing in for COS here.

## 6. Dynamic Postgres credentials for `workflow-beginner.yaml` (Database Secrets Engine)

Everything above is Vault's **KV v2** engine: a fixed value (`export_csv_reader`'s
username/password) written once, read many times, unchanged until someone
overwrites it. `workflow.yaml` still uses exactly that. `workflow-beginner.yaml`'s
`export-one-query` step instead uses Vault's **Database Secrets Engine**,
which doesn't store a credential at all — it connects to Postgres itself
and *creates a brand-new, expiring role* every time something asks it for
one.

```bash
kubectl -n vault exec vault-0 -- sh -c '
  export VAULT_TOKEN=root

  vault secrets enable database

  # Vault needs its own admin-capable connection to Postgres to run the
  # CREATE ROLE / GRANT / DROP ROLE statements below on demand. Reusing
  # labuser (already a superuser here, see above) rather than provisioning
  # a separate Vault-admin Postgres role: fine for this lab as long as you
  # never run `vault write -f database/rotate-root/postgres-lab` against
  # it - that would let Vault take over managing labusers own password,
  # silently breaking db-backup-postgres, which still needs to know it.
  # A real deployment would give Vault a dedicated admin identity instead
  # of sharing one with an application that has its own static credential.
  vault write database/config/postgres-lab \
    plugin_name=postgresql-database-plugin \
    connection_url="postgresql://{{username}}:{{password}}@postgres.argo.svc.cluster.local:5432/labdb?sslmode=disable" \
    allowed_roles="export-csv-cos-pg-role" \
    username="labuser" \
    password="lab-password-change-me"

  # creation_statements runs once, at the moment something reads
  # database/creds/export-csv-cos-pg-role - {{name}}, {{password}} and
  # {{expiration}} are filled in by Vault itself. revocation_statements
  # runs once the lease ends, dropping the role Vault just created.
  vault write database/roles/export-csv-cos-pg-role \
    db_name=postgres-lab \
    creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '"'"'{{password}}'"'"' VALID UNTIL '"'"'{{expiration}}'"'"'; GRANT USAGE ON SCHEMA public TO \"{{name}}\"; GRANT SELECT ON ALL TABLES IN SCHEMA public TO \"{{name}}\";" \
    revocation_statements="REVOKE ALL PRIVILEGES ON ALL TABLES IN SCHEMA public FROM \"{{name}}\"; REVOKE ALL PRIVILEGES ON SCHEMA public FROM \"{{name}}\"; DROP ROLE IF EXISTS \"{{name}}\";" \
    default_ttl="5m" \
    max_ttl="15m"

  vault policy write export-csv-cos-pg-dynamic - <<EOF
path "database/creds/export-csv-cos-pg-role" {
  capabilities = ["read"]
}
EOF

  vault write auth/kubernetes/role/export-csv-cos-pg-dynamic \
    bound_service_account_names=export-csv-cos-pg-dynamic-sa \
    bound_service_account_namespaces=argo \
    policies=export-csv-cos-pg-dynamic \
    ttl=15m
'
```

`export-one-query` runs under `export-csv-cos-pg-dynamic-sa` (created by
`rbac.yaml`, deliberately a *different* ServiceAccount than
`export-csv-cos-pg-sa`) — it can read `database/creds/export-csv-cos-pg-role`
and nothing else; it has no path to `secret/data/db/postgres` at all, so
switching this one workflow to dynamic secrets can't accidentally weaken or
change `workflow.yaml`'s access.

The database engine's response shape is different from KV v2's, which shows
up directly in the injector template. KV v2 nests the actual secret under
`.Data.data` (the outer `.Data` is Vault's response envelope, the inner
`data` is the KV v2 format itself); the database engine has no such second
layer - just `.Data.username` and `.Data.password`:

```yaml
vault.hashicorp.com/agent-inject-template-db-creds.sh: |
  {{- with secret "database/creds/export-csv-cos-pg-role" -}}
  export PGHOST="postgres"
  export PGPORT="5432"
  export PGDATABASE="labdb"
  export PGUSER="{{ .Data.username }}"
  export PGPASSWORD="{{ .Data.password }}"
  {{- end -}}
```

`PGHOST`/`PGPORT`/`PGDATABASE` are hardcoded here, not templated from Vault -
they're static connection details, not part of what the database engine
generates per lease. Only the credential itself (username, password) is
dynamic.

Verified, not just configured, at every stage of the lifecycle:
- `vault read database/creds/export-csv-cos-pg-role` (run manually, once,
  before touching the workflow) returned a `username`/`password` for a role
  that didn't exist a second earlier - confirmed by connecting with those
  exact credentials: `SELECT` succeeds, `INSERT` fails with `permission
  denied for table widgets`, same boundary as the static `export_csv_reader`
  role.
- `vault lease revoke <lease_id>` on that manually-read credential actually
  dropped the role - `\du` on its username afterward returned nothing.
- Submitting the real workflow (4 parallel `export-one-query` pods via
  `withItems`) produced **4 distinct** roles in Postgres at once
  (`v-kubernet-export-c-...`, each with its own `Password valid until`
  timestamp) and **4 distinct** entries under
  `vault list sys/leases/lookup/database/creds/export-csv-cos-pg-role` -
  one per pod, not one shared credential reused four times.
- Unattended auto-revocation once `default_ttl` elapses - the actual point
  of using this engine over a static secret - was watched happen, not
  assumed: same 4 roles from the run above, checked again 5+ minutes later
  with nobody in between calling `vault lease revoke` or anything else.
  `\du` on Postgres showed only `labuser` and `export_csv_reader` (the
  workflow.yaml-facing static role) - all 4 dynamic
  `v-kubernet-export-c-...` roles gone. `vault list
  sys/leases/lookup/database/creds/export-csv-cos-pg-role` returned
  `No value found` - zero active leases, zero credentials left to leak.

## The `{{ }}` collision that didn't happen

Argo Workflows and Vault Agent Injector both use `{{ }}` — Argo for its own
variable substitution (`{{workflow.name}}`, `{{inputs.parameters.x}}`, ...),
Vault's `agent-inject-template-*` annotations for Consul-template syntax
(`{{ with secret "..." }}...{{ end }}`). It was genuinely unclear going in
whether Argo would try to resolve the Vault template's `{{ .Data.data.host }}`
as an (unresolvable) Argo variable and fail lint/submission. It doesn't:
`argo lint --offline` passed and the workflow submitted and ran cleanly with
the annotation inlined as-is — Argo only substitutes recognized variable
prefixes (`workflow.`, `steps.`, `inputs.`, ...) and leaves everything else,
including `.Data.data.host`, untouched. Worth re-verifying if you're on a
much older or newer Argo version than the one this lab was built against
(`argo version --short` → v4.0.7); if a future version starts erroring on
this, the fallback is `vault.hashicorp.com/agent-configmap` pointing at a
separate ConfigMap holding the Vault Agent template - Argo never touches the
contents of an unrelated ConfigMap object.

## Cleanup

```bash
helm uninstall vault -n vault
kubectl delete namespace vault
```
