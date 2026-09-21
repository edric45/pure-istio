# pure-istio

Istio ambient configuration for VMware VKS (vSphere Kubernetes Service),
delivered through the platform's Istio add-on.

Three files:

| File | Applied to | What it does |
|---|---|---|
| [`addon.yaml`](addon.yaml) | **Supervisor** | `AddonConfig` + `AddonInstall`. Every value is annotated with the requirement it satisfies. |
| [`overlays/01-waypoint-defaults.yml`](overlays/01-waypoint-defaults.yml) | **workload cluster** | Waypoint HPA, PDB and resources. The add-on has no values for HPA or PDB. |
| [`overlays/02-istio-cni-excludes.yml`](overlays/02-istio-cni-excludes.yml) | **workload cluster** | Adds `istio-system` to the istio-cni exclusion list. The add-on has no value for this. |

Verified against Istio `1.28.2+vmware.1-vks.1` on Kubernetes `v1.35.6`,
ClusterClass `builtin-generic-v3.6.0`.

## Applying

### 1. The add-on — to the Supervisor

`addon.yaml` ships with placeholders. Replace them first:

```sh
grep -n 'SUPERVISOR-NAMESPACE\|CLUSTER-NAME' addon.yaml
```

- `SUPERVISOR-NAMESPACE` — the vSphere Namespace holding your cluster
- `CLUSTER-NAME` — your workload cluster name (appears three times)

Then:

```sh
kubectl --context=<supervisor> apply -f addon.yaml
```

Reconcile takes 2 to 5 minutes. The Supervisor's `ClusterAddon` status lags the
workload cluster by a few minutes, so check the workload cluster directly:

```sh
kubectl --context=<workload-cluster> -n istio-system get ds ztunnel
```

### 2. The overlays — to the workload cluster

The add-on generates a Carvel `PackageInstall` on the workload cluster, which
renders the Istio manifests with `ytt`. kapp-controller lets you add files to
that render through an annotation pointing at a Secret. Both steps run against
the **workload cluster**, not the Supervisor:

```sh
# the Secret must be in the SAME namespace as the PackageInstall
kubectl -n vmware-system-tkg create secret generic istio-overlays \
  --from-file=01-waypoint-defaults.yml=overlays/01-waypoint-defaults.yml \
  --from-file=02-istio-cni-excludes.yml=overlays/02-istio-cni-excludes.yml \
  --dry-run=client -o yaml | kubectl apply -f -

# find your PackageInstall name first:
#   kubectl -n vmware-system-tkg get packageinstall
kubectl -n vmware-system-tkg annotate packageinstall <name> \
  ext.packaging.carvel.dev/ytt-paths-from-secret-name.0=istio-overlays --overwrite
```

The trailing `.0` is an index; use `.1`, `.2` for further Secrets.

### 3. Confirm

```sh
kubectl -n vmware-system-tkg get packageinstall <name> \
  -o jsonpath='{.status.friendlyDescription}{"\n"}{.status.usefulErrorMessage}{"\n"}'

kubectl -n istio-system get cm istio-waypoint-defaults
kubectl -n istio-system get cm istio-cni-config -o jsonpath='{.data.EXCLUDE_NAMESPACES}'
```

`usefulErrorMessage` is where render failures surface — and the only place. A
bad overlay reference fails the PackageInstall while the `ClusterAddon` may
still look healthy.

### Removing

```sh
kubectl -n vmware-system-tkg annotate packageinstall <name> \
  ext.packaging.carvel.dev/ytt-paths-from-secret-name.0-
kubectl -n vmware-system-tkg delete secret istio-overlays
```

Reconciliation recovers on the next cycle.

## How the overlays work

kapp-controller adds your files to ytt's input set. What happens next depends
on the file:

| Goal | File contents | ytt syntax |
|---|---|---|
| **Add** a new object | a plain Kubernetes manifest | none |
| **Patch** an existing object | `#@overlay/match` directives | yes |

`01-waypoint-defaults.yml` is the first kind. It cannot be a patch: istiod
generates the waypoint `Deployment` at runtime, so that object is never part of
the package render and no matcher can select it. Adding a defaults ConfigMap
for istiod to read is the route that works.

`02-istio-cni-excludes.yml` is the second kind. `istio-cni-config` **is**
rendered by the package, so it can be patched in place.

Only objects the package renders can be patched.





