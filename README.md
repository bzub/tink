# Talos in Kubernetes

## Prerequisites

- A Kubernetes cluster.
- `kustomize` build tool.
- `talosctl` and `kubectl`
- `jq`

## Quick Start

Deploy a cluster with one control-plane node, one worker node, and default settings.
```sh
kustomize build "github.com/bzub/tink/resources/cluster" | kubectl apply -f -
```

### Interacting With The New Cluster

Proxy the talos and kubernetes APIs to your local machine.
```sh
kubectl port-forward svc/controlplane 50000:50000 &
kubectl port-forward svc/controlplane 6443:6443 &
```

Get the talosconfig and kubeconfig for the cluster.
```sh
kubectl get secret mycluster -o json |\
  jq -r '.data["talosconfig"]|@base64d' > /tmp/talosconfig

talosctl --talosconfig=/tmp/talosconfig \
  config nodes localhost

talosctl --talosconfig=/tmp/talosconfig \
  config endpoint localhost

talosctl --talosconfig /tmp/talosconfig -e localhost -n localhost \
  kubeconfig --force --merge=false /tmp/kubeconfig

kubectl --kubeconfig /tmp/kubeconfig \
  config set-cluster mycluster --server=https://localhost:6443
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
