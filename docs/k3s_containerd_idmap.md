# K3s + containerd: ID-mapped Workloads with nydus-snapshotter

This note summarizes the required containerd proxy-plugin setting for user-namespace
(`idmap`) workloads when using `nydus-snapshotter`.

## Symptom

When running containers/pods with UID/GID remapping, startup may fail with errors like:

- `mknod ... /rootfs/etc/hosts: permission denied`
- `mknod ... /rootfs/etc/resolv.conf: permission denied`

In the failing path, snapshot keys may include a compatibility suffix such as `-remap`.

## Root Cause

For proxy snapshotters, containerd relies on capability negotiation.
If `[proxy_plugins.nydus]` does not advertise `remap-ids`, containerd may route
ID-mapped requests to a compatibility remap fallback path (intermediate `*-remap`
snapshots) instead of using the snapshotter's direct idmap-aware flow.

In affected `k3s + containerd` environments, that fallback path can fail during
bind-mount target creation inside the container rootfs.

## Required Configuration

Add `remap-ids` capability under the nydus proxy plugin:

```toml
[proxy_plugins]
  [proxy_plugins.nydus]
    type = "snapshot"
    address = "/run/containerd-nydus/containerd-nydus-grpc.sock"
    capabilities = ["remap-ids"]
```

### K3s-specific note

K3s usually renders containerd config from template:

- `/var/lib/rancher/k3s/agent/etc/containerd/config-v3.toml.tmpl`

Make the same change in the template, then restart k3s so generated containerd config
contains `capabilities = ["remap-ids"]`.

## Quick Validation

1. Confirm generated containerd config includes `capabilities = ["remap-ids"]`.
2. Run an ID-mapped container with remap labels and a bind mount.
3. Verify the container starts successfully.
4. (Optional) Check nydus logs and confirm no fallback `*-remap` chain is used for
   the container key in the successful run.

Example (k3s `ctr`):

```bash
sudo CONTAINERD_SNAPSHOTTER=nydus k3s ctr run \
  --uidmap 0:100000:65536 \
  --gidmap 0:100000:65536 \
  --remap-labels \
  --mount type=bind,src=/tmp/hosts-test,dst=/etc/hosts,options=rbind:rw \
  docker.io/library/busybox:1.36 idmap-check sh -lc 'sleep 30'
```

## Scope Notes

- Non-idmap workloads typically run regardless of this capability.
- This is independent from insecure registry settings.
- This is also independent from media-type unpack support for nydus-native blobs
  (for example `application/vnd.oci.image.layer.nydus.blob.v1` processor availability).

## Related Documents

- [run_nydus_in_kubernetes.md](./run_nydus_in_kubernetes.md)
