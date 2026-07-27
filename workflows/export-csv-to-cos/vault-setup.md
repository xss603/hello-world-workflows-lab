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

```bash
kubectl -n vault exec vault-0 -- sh -c '
  export VAULT_TOKEN=root
  vault kv put secret/db/postgres \
    host=postgres port=5432 \
    username=labuser password=lab-password-change-me database=labdb

  vault kv put secret/cos/ibm \
    endpoint=http://minio.argo.svc.cluster.local:9000 \
    access_key=admin secret_key=password bucket=csv-exports
'
```

In production, swap the `cos/ibm` values for a real IBM COS HMAC key pair
and endpoint (e.g. `https://s3.us-south.cloud-object-storage.appdomain.cloud`)
— nothing else about the workflow changes, since `mc` and the injector
templates don't know or care that MinIO is standing in for COS here.

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
