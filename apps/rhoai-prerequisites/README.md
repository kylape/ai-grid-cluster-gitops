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
* the remote-only Limitador Redis configuration that shares MaaS rate-limit
  counters through the Submariner service import.
* the catalog ServiceAccount's `imagePullSecrets` wiring is managed by
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

## Shared Limitador Redis

`apps/limitador-redis` owns the Redis instance and its local Limitador storage
configuration on `local-cluster`. On every other managed cluster, the
`rhoai-limitador-redis` Policy creates the `limitador-redis` namespace needed
for the Submariner `ServiceImport`, the `kuadrant-system/limitador-redis`
Secret, and the `Limitador/limitador` Redis storage reference. Remote
Limitadors therefore use:

```text
redis://limitador-redis-master.limitador-redis.svc.clusterset.local:6379
```

The local Redis NetworkPolicy permits Submariner GlobalNet traffic
(`242.0.0.0/8`). The policy intentionally excludes `local-cluster` so the
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
