# Etcd Restore Component

The `etcd-restore` component helps automate restoring from an [etcd database
snapshot][etcd-snapshot].

## Prerequisites

- An etcd database snapshot on the computer that is accessing the cluster.
- The sha512sum of the etcd database snapshot file.

## Configuration

`etcd-restore` refers to a ConfigMap for user configuration. Check the
component's `configMapGenerator` for all options and default values.

### Required Options

The following keys are required.
- `SNAPSHOT_SHA512SUM`
  - The sha512sum of the etcd database snapshot. Used to ensure the snapshot is
    valid before performing a restore.

## Demo

### Configure And Apply `etcd-restore`

First let's get all the configuration in place to perform an etcd restore, and
apply the resources.
```sh
demo=$(mktemp -d)
pushd $demo

snapshot_file="/tmp/my-etcd-snapshot.db"
snapshot_sha512sum="$(sha512sum "${snapshot_file}" | awk '{print $1}')"

# Start with a base TinK cluster configuration.
kustomize create --resources github.com/bzub/tink/resources/cluster

# Add the etcd-restore component.
kustomize edit add component github.com/bzub/tink/components/etcd-restore

# Add some required options for etcd-restore. In this example the snapshot was
# gathered from the data directory of an etcd member, so it doesn't contain a
# built in hash unlike one gathered via `etcdctl snapshot save`.
kustomize edit add configmap etcd-restore \
  --behavior=merge \
  --from-literal="SNAPSHOT_SHA512SUM=${snapshot_sha512sum}" \
  --from-literal="SKIP_HASH_CHECK=true"

kustomize build | kubectl apply -f -
```

### Send The Database Snapshot To The Pod For Restore

The `etcd-restore` component patches a TinK controlplane StatefulSet by adding
`initContainers`. One of these containers waits for the user to send the etcd
database snapshot to it, and the other performs the restore operation.

```sh
kubectl cp "${snapshot_file}" controlplane-0:/etcd-snapshot/snapshot.db \
  -c etcd-snapshot-upload-wait
```

The pod should now start a new etcd cluster using the database keys/values from
the provided snapshot.

## HA Etcd Clusters

The procedure for restoring an HA etcd cluster starts the same as the single
node demo above. After one node is up and Ready you can add members like normal
without the `etcd-restore` component.

It is recommended to prevent changes to the first etcd member that was used for
restore while the additional members join the cluster. To do this with TinK you
can configure the controlplane StatefulSet with
`spec.updateStrategy.rollingUpdate.partition` set to `1`.

```sh
# Scale up the controlplane.
kustomize edit set replicas controlplane=3

# Spin up new controlplane nodes before applying changes to the restore node.
kustomize edit add patch \
  --kind=StatefulSet \
  --version=v1 \
  --label-selector=app.kubernetes.io/name=talos,app.kubernetes.io/component=node,node-role.kubernetes.io/control-plane= \
  --patch="$(cat <<'EOF'
- op: replace
  path: /spec/updateStrategy
  value:
    rollingUpdate:
      partition: 1
EOF
)"

# Apply the configs.
kustomize build | kubectl apply -f -
```

## Post-Restore Cleanup

Remove the `etcd-restore` component and `updateStrategy` patch from your
`kustomization.yaml` and build/apply to get back to normal operations.

[etcd-snapshot]: https://etcd.io/docs/v3.4/op-guide/recovery/#snapshotting-the-keyspace
