# RHOAI prerequisites

This app distributes the RHOAI installation prerequisites to clusters in the
reserved `global` ACM cluster set:

* the protected `redhat-ai-grid` and `redhat-ods-operator` Namespaces;
* the `ImageDigestMirrorSet`, unless a cluster is marked for external mirror
  management;
* the `openshift-marketplace/rhoai-catalog-pull-secret` required to pull the
  RHOAI catalog image;
* merged `openshift-config/pull-secret` entries for `quay.io/rhoai`, required
  for mirrored operator-bundle pulls, and for every other registry the hub
  Secret holds, such as `registry.stage.redhat.io` for the
  `registry.redhat.io/rhaii*` mirrors that model images pull through.
- the Limitador Redis configuration, on remote ingress and combined sites only, that shares MaaS rate-limit
  counters through the Submariner service import.
- the hub-only addon registration of `rhai-metrics` and of `ai-grid-dashboard-platform-metrics`
  (the GPU metric the AI Grid dashboards need beyond ACM's default list) and, on enabled grid sites,
  the NetworkPolicy admitting the local ACM agents to each RHOAI collector's
  HTTPS exporter on port 8889. It retries until RHOAI creates its monitoring
  namespace and replaces the hub-only policy in `ai-grid-o11y`; the initial
  prune/reconcile handoff can briefly interrupt these scrapes. Do not also
  deploy an ApplicationSet managing that NetworkPolicy. Only the addon's copied
  scrape pool keeps targets, preventing duplicate collection on the hub.
  Removing the registration policy does not withdraw its addon tuple; that
  requires a separate migration preserving the other configs.
- temporary cert-manager permissions for the OpenTelemetry operator on enabled Grid sites
  with role `ingress`, `inference` or `combined`, including an enabled hub.
- the catalog ServiceAccount's `imagePullSecrets` wiring is managed by
  `apps/rhoai/catalog`; OLM does not update an already-created catalog
  ServiceAccount when `CatalogSource.spec.secrets` changes.

## External image mirror management

By default, ACM enforces the image mirror policy on every cluster selected from
`global`. To let another control plane manage a cluster's mirror, set the
`ai-grid/registry-mirror-management=external` label on that cluster's ACM
`ManagedCluster` before the mirror policy is applied. The placement then omits
only the `ImageDigestMirrorSet` policy for that cluster; the other
RHOAI prerequisites continue to apply.

For a HyperShift hosted cluster, mark the corresponding ACM `ManagedCluster`
as externally managed, then add the four RHOAI mirror mappings to the
`HostedCluster.spec.imageContentSources` list in the management cluster. The
[hub onboarding guide](../../docs/acm/hub-onboarding/README.md#dedicated-hub)
shows the exact entries and edit command. The HostedCluster field cannot set
`mirrorSourcePolicy: NeverContactSource`; if falling back to the source registry
is unacceptable, confirm a supported configuration for the installed
HyperShift version. Updating `imageContentSources` rolls out existing
NodePools. The ACM label does not configure mirrors or registry credentials.
Remove the label to return mirror management to this GitOps app.

For example, on the ACM hub:

```sh
kubectl --context "$HUB_CONTEXT" label managedcluster <cluster-name> \
  ai-grid/registry-mirror-management=external
```

The Quay credentials are never stored in Git. The pull-secret policy uses an
ACM hub template to check for `rhoai-prerequisites/rhoai-quay-pull-secret`
before rendering the remote Secret. Until that hub Secret exists and contains
a non-empty `.dockerconfigjson` key, the pull-secret policy has no remote
object templates and makes no changes on managed clusters. The image mirror is
still distributed because it does not contain credentials.

## Clean-cluster flow

`bootstrap.sh` seeds only the hub-side credential Secret. The root
ApplicationSet then discovers this app, ACM applies the prerequisites to every
cluster in the `global` cluster set, and the RHOAI ApplicationSet deploys after
them. No per-cluster `rosa`, `ocm`, or pull-secret command is required when a
managed cluster joins.

The global pull-secret policy is idempotent. It reads the current
`openshift-config/pull-secret`, preserves every existing credential, and
adds the hub-side entries it lacks: `quay.io/rhoai` from the Quay credential,
and any other registry the hub Secret holds. A bare `quay.io` entry in the hub
Secret is applied only as `quay.io/rhoai`, so the cluster's own `quay.io` login
stays. Updating that Secret triggers the normal Machine Config
Operator rollout to each node.

## OpenTelemetry cert-manager compatibility

The compatibility grant uses `openshift-opentelemetry-operator`, explicitly set in
`apps/rhoai/values.yaml`. A custom operator namespace requires updating the policy's
ServiceAccount subject as well. The grant mirrors [RHOAI's supplemental RBAC](https://github.com/opendatahub-io/odh-observability/blob/f7050b59693a1a12da127cbb757d75fa4d125ef5/charts/odh-observability/templates/opentelemetry-cert-manager-rbac.yaml).

OTel caches cert-manager discovery and authorization at startup; Argo sync waves
do not guarantee these prerequisites are ready first. If OTel started too early,
confirm policy compliance, cert-manager API availability and operator permissions.
Replace webhook pods one at a time, maintaining a Ready serving replica and
waiting for each replacement to be Ready; verify webhook
endpoints, then replace controller pods one at a time and wait for readiness.
OLM can revert `rollout restart`.
Allow RHOAI to retry, then verify collector/allocator readiness, Certificates,
Monitoring Ready at its current generation, and collector scrape targets.

Retire the workaround only after equivalent product-owned access is verified.
The policy retains its RBAC on deletion or deselection; explicitly remove the
compatibility ClusterRoleBinding and ClusterRole through GitOps during retirement.

## Shared Limitador Redis

`apps/limitador-redis` owns the Redis instance and its local Limitador storage
configuration on the hub. On every other managed cluster, the
`rhoai-limitador-redis` Policy creates the `limitador-redis` namespace needed
for the Submariner `ServiceImport`, the `kuadrant-system/limitador-redis`
Secret, and the `Limitador/limitador` Redis storage reference. Remote
Limitadors therefore use:

```text
redis://limitador-redis-master.limitador-redis.svc.clusterset.local:6379
```

The local Redis NetworkPolicy permits Submariner GlobalNet traffic
(`242.0.0.0/8`). The policy intentionally excludes the hub so the
existing local Argo Application remains the sole owner of the Redis and local
Limitador storage resources.

## Create the hub Secret

Set `RHOAI_QUAY_PULL_SECRET` to the base64-encoded `username:password` value
that can pull both the RHOAI catalog and operator-bundle repositories, then
run:

```sh
export HUB_CONTEXT='<hub-kubeconfig-context>'
read -s RHOAI_QUAY_PULL_SECRET
export RHOAI_QUAY_PULL_SECRET

apps/rhoai-prerequisites/create-hub-secret.sh
```

ACM watches the hub Secret and reconciles the policy after it is created or
updated. Removing the hub Secret does not delete an already-created pull Secret
on a managed cluster; this avoids accidentally removing credentials because of
a temporary hub-side Secret outage.

## Verify

Use metadata-only commands so the credential is not displayed:

```sh
kubectl --context "$HUB_CONTEXT" -n rhoai-prerequisites get secret rhoai-quay-pull-secret \
  -o custom-columns='NAME:.metadata.name,TYPE:.type'
kubectl --context "$HUB_CONTEXT" -n rhoai-prerequisites get placementdecisions
kubectl --context "$HUB_CONTEXT" get policies -n rhoai-prerequisites
kubectl --context "$SPOKE_CONTEXT" -n openshift-config get secret pull-secret \
  -o custom-columns='NAME:.metadata.name,TYPE:.type'
kubectl --context "$SPOKE_CONTEXT" -n rhoai-prerequisites get configurationpolicy \
  rhoai-global-pull-secret
kubectl --context "$SPOKE_CONTEXT" -n limitador-redis get serviceimport \
  limitador-redis-master
kubectl --context "$SPOKE_CONTEXT" -n kuadrant-system get limitador limitador \
  -o jsonpath='{.spec.storage.redis.configSecretRef.name}{"\n"}'
```
