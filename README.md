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

Watch the logs until the boot completes. Should only take a few minutes. Some
errors when services are starting up are normal.
```sh
kubectl -n default logs mycluster-controlplane-0 --follow
```

Proxy the talos and kubernetes APIs to your local machine.
```sh
kubectl -n default port-forward svc/mycluster-controlplane 50000:50000 &
kubectl -n default port-forward svc/mycluster-controlplane 6443:6443 &
```

Get the talosconfig and kubeconfig for the cluster.
```sh
kubectl -n default get secret mycluster -o json |\
  jq -r '.data["talosconfig"]' | base64 -D > /tmp/talosconfig

talosctl --talosconfig /tmp/talosconfig -n mycluster-controlplane-0 \
  kubeconfig --force --merge=false /tmp/kubeconfig

kubectl --kubeconfig /tmp/kubeconfig \
  config set-cluster mycluster --server=https://127.0.0.1:6443
```

Some examples of querying the new kubernetes/talos cluster.
```sh
kubectl --kubeconfig /tmp/kubeconfig \
  get pods -A

talosctl --talosconfig /tmp/talosconfig -n mycluster-controlplane-0 \
  health
```

Now you can manage the cluster like any other Kubernetes workload.

### Cleanup

To tear everything down kill the port-forward background jobs and delete the
resources.
```sh
kill %1 %2
kustomize build "github.com/bzub/tink/resources/cluster" | kubectl delete -f -
kubectl -n default delete secret mycluster
```
