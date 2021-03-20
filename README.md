# Talos in Kubernetes

## Prerequisites

- A Kubernetes cluster.
- `kustomize` build tool.
- `talosctl` and `kubectl`
- `jq`

## Quick Start

Start a single node cluster with default settings.
```sh
kustomize build "github.com/bzub/tink/resources/cluster" | kubectl apply -f -
```

Proxy the talos and kubernetes APIs to your local machine.
```sh
kubectl port-forward svc/controlplane 50000:50000 &
kubectl port-forward svc/controlplane 6443:6443 &
```

Get the talosconfig and kubeconfig for the cluster.
```sh
kubectl get secret mycluster -o json |\
  jq -r '.data["talosconfig"]' | base64 -D > /tmp/talosconfig

talosctl --talosconfig=/tmp/talosconfig \
  config nodes controlplane-0.controlplane.default.svc.cluster.local

talosctl --talosconfig /tmp/talosconfig \
  kubeconfig --force --merge=false /tmp/kubeconfig

kubectl --kubeconfig /tmp/kubeconfig \
  config set-cluster mycluster --server=https://127.0.0.1:6443
```

Bootstrap the cluster.
```sh
talosctl --talosconfig=/tmp/talosconfig bootstrap
```

Watch the health checks until the boot completes. Should only take a few
minutes. Some errors when services are starting up are normal.
```sh
talosctl --talosconfig=/tmp/talosconfig health
```

Now you can manage the cluster like any other Kubernetes workload.
```sh
kubectl scale statefulset --replicas 3 controlplane
```

### Cleanup

To tear everything down kill the port-forward background jobs and delete the
resources.
```sh
kill %1 %2
kustomize build "github.com/bzub/tink/resources/cluster" | kubectl delete -f -
kubectl -n default delete secret mycluster
```
