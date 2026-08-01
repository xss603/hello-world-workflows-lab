# SigNoz setup for `hello-otel`

One-time cluster setup `hello-otel` depends on: SigNoz (an open-source
observability platform - traces/metrics/logs on ClickHouse, with an
OpenTelemetry Collector as its ingestion point) installed via Helm into the
same kind cluster as the rest of this lab. Verified end-to-end against a
real install, including two real bugs hit getting it running - this isn't
aspirational documentation.

## 1. Install

```bash
helm repo add signoz https://charts.signoz.io
helm repo update signoz
kubectl create namespace signoz
helm install signoz signoz/signoz -n signoz
kubectl -n signoz wait --for=condition=Ready pod --all --timeout=300s
```

This installs ClickHouse (via the Altinity ClickHouse Operator, as a
`ClickHouseInstallation` CRD - a real `chi-signoz-clickhouse-cluster-0-0-0`
StatefulSet pod gets created, not just the operator), zookeeper (ClickHouse's
coordination dependency), the `signoz-otel-collector` (OTLP ingestion), and
`signoz-0` (the query-service + frontend, bundled as one pod/container in
this chart version). Full default values - no resource trimming - fit
comfortably alongside everything else already running in this lab's kind
cluster (~4.2GB/7.75GB node memory after this install, checked with `docker
stats <kind-node-container>` rather than assumed).

## 2. Gotcha: a fresh install's collector can't actually receive anything until you complete first-run setup in the UI

`hello-otel`'s first submission failed with `ConnectionRefusedError`
against `signoz-otel-collector.signoz.svc.cluster.local:4318` - port
correctly declared on the Service, pod `Running`, DNS resolving correctly,
and still refused. The actual cause took real digging, not a one-line fix:

1. `kubectl -n signoz exec deploy/signoz-otel-collector -- cat /proc/net/tcp`
   showed the OTLP receiver ports (4317/4318) simply weren't bound at all -
   only loopback-only debug ports (pprof, zpages) were listening, despite
   the collector's own ConfigMap correctly declaring `otlp: protocols: {grpc:
   0.0.0.0:4317, http: 0.0.0.0:4318}` wired into the traces pipeline.
2. The collector's logs showed why: a repeating, since-startup
   `opamp-server-client` error, `"Server returned an error response"`,
   every 30 seconds. This collector is OpAmp-managed - the static ConfigMap
   is only a starting point; the collector negotiates its *actual* running
   config with the SigNoz backend (`signoz-0`) over OpAmp, and the receivers
   don't fully activate until that negotiation succeeds.
3. `signoz-0`'s own logs had the real root cause: `"failed to find or
   create agent"` / `"cannot create agent without orgId"`. A brand new
   SigNoz install has no organization yet - that only gets created by
   completing the first-run sign-up flow, normally done by hand in the
   browser before anyone thinks to send it data.

**Fix**: open the UI (`kubectl -n signoz port-forward svc/signoz 8080:8080`,
then `http://localhost:8080`) and complete "Create your account" - this
creates the org the backend needs. That alone wasn't enough, though: the
collector pod had already been stuck in its failed OpAmp retry loop since
before the org existed, and didn't spontaneously recover once one appeared.
It needed an explicit restart to retry the handshake fresh:

```bash
kubectl -n signoz delete pod -l app.kubernetes.io/component=otel-collector
kubectl -n signoz wait --for=condition=Ready pod -l app.kubernetes.io/component=otel-collector --timeout=90s
```

After the restart, the collector's own logs show the fix took:
`"Starting GRPC server" endpoint="[::]:4317"` and `"Starting HTTP server"
endpoint="[::]:4318"` - confirmed by actually re-testing the connection
(`nc -vz signoz-otel-collector.signoz.svc.cluster.local 4318`), not just by
reading a log line and assuming it meant what it said.

One more thing this uncovered: those ports bind on `[::]` (the IPv6
wildcard), not `0.0.0.0`. A quick first check via `/proc/net/tcp` (IPv4
only) genuinely showed nothing listening even after the fix - the real
listener was sitting in `/proc/net/tcp6`. Worth remembering if you ever
debug "is anything listening on this port" inside a container this way
again: check both tables, or just test the actual connection instead of
inferring from `/proc`.

## 3. Where `hello-otel` sends spans

`http://signoz-otel-collector.signoz.svc.cluster.local:4318/v1/traces` -
OTLP/HTTP, plain JSON (not protobuf), no auth. Fine for this lab; a
production SigNoz install would sit behind auth/TLS and the collector
endpoint wouldn't be open to every namespace in the cluster the way it is
here.

## 4. Viewing the trace

```bash
kubectl -n signoz port-forward svc/signoz 8080:8080
```

Then `http://localhost:8080` → Traces, or jump straight to a specific trace
at `http://localhost:8080/trace/<trace-id>` (the workflow logs print each
span's ID as it's sent, and `{{workflow.uid}}` with dashes stripped is the
trace ID - see workflow.yaml's comments for why).

## Cleanup

```bash
helm uninstall signoz -n signoz
kubectl delete namespace signoz
```
