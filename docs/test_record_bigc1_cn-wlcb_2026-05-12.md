# Test Record: bigc1 cn-wlcb cluster (2026-05-12)

End-to-end deployment of `nydus-snapshotter` on the bigc1 cn-wlcb k3s
cluster with **UCloud US3** as the lazy-pull blob backend, plus the
`bigc1-serverless` image-sync-worker producing nydus variants of every
synced OCI image.

## Scope

- nydus-snapshotter DaemonSet on a single pilot worker, without
  disturbing the existing default snapshotter (`uimgacc`, UCloud image
  acceleration).
- `nydusify convert` integrated into the image-sync-worker so each
  synced image gets a sibling `<tag>-nydus` reference with blob content
  pushed to US3.
- Verified reader-side end-to-end: `nerdctl pull` + `nerdctl run` of the
  nydus image actually mounts the rootfs lazily from US3 and the
  container runs `/bin/sh` successfully.

## Environment

| Component | Value |
|---|---|
| Cluster | k3s v1.33.3+k3s1, control-plane `10.60.183.56` |
| Pilot worker | `10-60-207-36` (UCloud uhost in `cn-wlcb-01`) |
| Other workers (idle this round) | `10-60-134-225`, `10-60-153-178`, `10-60-75-67` |
| Container runtime | **External standalone** `containerd 2.1.4` (k3s started with `--container-runtime-endpoint=/run/containerd/containerd.sock`) |
| containerd config | `/etc/containerd/config.toml`, version 2 |
| Default snapshotter | `uimgacc` (UCloud) — preserved |
| Kernel | 6.12.35-061235-generic |
| nydus-static toolchain | v2.3.0 (`nydusify`, `nydus-image`, `nydusd`) |
| nydus-snapshotter binary | `v0.15.14-15-ge537237` (user fork) |
| US3 backend | `internal.s3-cn-wlcb.ufileos.com` (HTTP, intra-VPC), bucket `big-c1-nydus-images` |

The k3s containerd at `/var/lib/rancher/k3s/agent/etc/containerd/config.toml`
is **not** the one in use — k3s offloads runtime to the standalone
service. All config edits below target `/etc/containerd/config.toml`.

## Writer side (image-sync-worker)

The image-sync-worker is a `bigc1-serverless` component. PR #31 in that
repo wires `nydusify convert` after each OCI copy. Key choices captured
from the rollout:

- `WITH_NYDUS=true` Docker build arg toggles a `nydus-true` stage that
  fetches `nydus-static-${NYDUS_VER}-linux-amd64.tgz`; other builds
  (gateway, worker-agent) keep `WITH_NYDUS=false` so they stay slim.
- `NYDUS_ENABLED=true` env makes the worker shell out to `nydusify
  convert` after OCI copy succeeds; nydus tag = `<target>-nydus`.
- Auth: writes a per-invocation `$DOCKER_CONFIG/config.json` containing
  the source and target registry auths and exports `DOCKER_CONFIG=...`.
  nydusify v2.3.0 has no `--target-auth` flag; it reads creds the
  docker way. Defer-RemoveAll cleans the dir after each task.
- Inherits `os.Environ()` when exec'ing nydusify; otherwise PATH is
  empty and nydusify cannot spawn `nydus-image`.
- Container runs as `runAsUser: 0` because OCI layer unpack calls
  `Lchown` to UID 0; nonroot uid 65532 hits EPERM on root-owned files.
- `NYDUS_BACKEND_REGION="cn-wlcb"` (nydusify v2.3.0 rejects empty
  region even though UCloud US3 ignores the value when signing).
- `NYDUS_BACKEND_SCHEME="http"` for the UCloud US3 internal endpoint;
  its TLS cert is valid for `*.mxgent.cn`, not
  `internal.s3-cn-wlcb.ufileos.com`. Traffic stays on the UCloud VPC.

Verified syncs:

| Source | Result | Duration |
|---|---|---|
| `docker.io/library/alpine:3.20` | `uhub.service.ucloud.cn/runc-serverless/nydus-alpine:3.20{,-nydus}` | ~15 s |
| `docker.io/library/busybox:1.36` | `uhub.service.ucloud.cn/runc-serverless/nydus-busybox:1.36{,-nydus}` | ~2 min (multi-platform manifest list) |

Both also have blob content under `s3://big-c1-nydus-images/`.

## Reader side (nydus-snapshotter)

This is the side this document captures in detail because the cluster
needed **two non-obvious config changes outside of the snapshotter
manifests** before a container would actually start.

### DaemonSet baseline

The kustomize **base** overlay was used (not the `k3s` overlay) because
the cluster runs standalone containerd, not k3s-embedded containerd:

```bash
kubectl apply -f misc/snapshotter/nydus-snapshotter-rbac.yaml
kubectl apply -f misc/snapshotter/base/nydus-snapshotter.yaml
```

A DaemonSet `nodeSelector: kubernetes.io/hostname: 10-60-207-36` limited
the pilot to one node.

The ConfigMap was overridden to set:

```yaml
FS_DRIVER: "fusedev"
ENABLE_CONFIG_FROM_VOLUME: "true"
ENABLE_RUNTIME_SPECIFIC_SNAPSHOTTER: "true"   # CRITICAL — see below
ENABLE_SYSTEMD_SERVICE: "false"
```

`ENABLE_RUNTIME_SPECIFIC_SNAPSHOTTER` defaults to `false` in
`snapshotter.sh deploy`. With that default the script runs

```bash
sed -i -e '/\[plugins\..*\.containerd\]/,/snapshotter =/ s/snapshotter = "[^"]*"/snapshotter = "nydus"/' "${CONTAINER_RUNTIME_CONFIG}".bak
```

against `/etc/containerd/config.toml`, which **replaces the cluster's
default snapshotter (`uimgacc`) with `nydus`**. Setting the env to
`"true"` skips that sed and leaves the default alone; nydus is added
purely as an additional `[proxy_plugins.nydus]` entry.

### Config fix 1: containerd `transfer.v1.local.unpack_config` for nydus

`containerd 2.x` resolves pulls through the **transfer plugin**, and
the transfer plugin only unpacks into snapshotters listed in its
`unpack_config`. The bigc1 baseline config has overlayfs only:

```toml
[[plugins."io.containerd.transfer.v1.local".unpack_config]]
differ = ""
platform = "linux/amd64"
snapshotter = "overlayfs"
```

A `ctr/nerdctl pull --snapshotter nydus` against that config commits a
snapshot with **no nydus labels**, so the snapshotter later returns an
empty Active mount during container start — the container then crashes
with `exec: "sh": executable file not found in $PATH`.

Fix: append a second `unpack_config` for nydus:

```toml
[[plugins."io.containerd.transfer.v1.local".unpack_config]]
differ = ""
platform = "linux/amd64"
snapshotter = "nydus"
```

then `systemctl restart containerd`. Backup at
`/etc/containerd/config.toml.pre-nydus-unpack.<timestamp>`. Existing
containers keep running through the ~3-5s shim-only restart window;
new pod create/delete blocks momentarily.

### Config fix 2: `nydusd-config.fusedev.json` scheme/region

The nydus snapshotter's runtime config file (delivered via the
ConfigMap when `ENABLE_CONFIG_FROM_VOLUME=true`) was initially:

```json
"backend": {
  "type": "s3",
  "config": {
    "scheme": "https",
    "endpoint": "internal.s3-cn-wlcb.ufileos.com",
    "bucket_name": "big-c1-nydus-images",
    "region": "",
    "access_key_id": "TOKEN_...",
    "access_key_secret": "..."
  }
}
```

`nydusd` started but failed before opening its api.sock:

```
service mount error: RAFS failed to handle request,
Failed to load config: failed to parse configuration information
```

Two reasons:

1. **TLS cert mismatch on internal endpoint.** UCloud's internal US3
   host (`internal.s3-cn-wlcb.ufileos.com`) serves a certificate valid
   for `*.mxgent.cn`. nydusd's S3 client refuses the handshake. Inside
   the VPC, HTTP is acceptable; the traffic never leaves the UCloud
   private network.
2. **Region must be non-empty.** Even though UCloud US3 ignores the
   value during signing, nydusd v2.3.0 rejects the config when region
   is missing/empty.

Fix the ConfigMap to:

```json
"scheme": "http",
"region": "cn-wlcb",
```

Then `kubectl rollout restart ds/nydus-snapshotter -n nydus-system`.

### Snapshotter ConfigMap (final, for reference)

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nydus-snapshotter-configs
  namespace: nydus-system
data:
  FS_DRIVER: "fusedev"
  ENABLE_CONFIG_FROM_VOLUME: "true"
  ENABLE_RUNTIME_SPECIFIC_SNAPSHOTTER: "true"
  ENABLE_SYSTEMD_SERVICE: "false"

  config.toml: |
    version = 1
    root = "/var/lib/containerd/io.containerd.snapshotter.v1.nydus"
    address = "/run/containerd-nydus/containerd-nydus-grpc.sock"
    daemon_mode = "dedicated"
    [daemon]
    nydusd_config = "/etc/nydus/nydusd-config.fusedev.json"
    nydusd_path = "/usr/local/bin/nydusd"
    nydusimage_path = "/usr/local/bin/nydus-image"
    fs_driver = "fusedev"
    recover_policy = "failover"
    threads_number = 4
    # ... (other defaults from upstream config.toml)

  nydusd-config.fusedev.json: |
    {
      "device": {
        "backend": {
          "type": "s3",
          "config": {
            "scheme": "http",
            "endpoint": "internal.s3-cn-wlcb.ufileos.com",
            "bucket_name": "big-c1-nydus-images",
            "region": "cn-wlcb",
            "object_prefix": "",
            "access_key_id": "TOKEN_<...>",
            "access_key_secret": "<...>"
          }
        },
        "cache": {
          "type": "blobcache",
          "config": { "work_dir": "/var/lib/containerd-nydus/cache" }
        }
      },
      "mode": "direct",
      "digest_validate": false,
      "iostats_files": false,
      "enable_xattr": true,
      "amplify_io": 1048576,
      "fs_prefetch": {
        "enable": true,
        "threads_count": 8,
        "merging_size": 1048576,
        "prefetch_all": true
      }
    }
```

## End-to-end verification

```bash
# On the pilot worker, with image-sync-worker creds for runc-serverless org
sudo /usr/local/bin/nerdctl --address=/run/containerd/containerd.sock \
    --namespace nydus-runtest --snapshotter nydus \
    login uhub.service.ucloud.cn -u <user> -p <pass>

# Clean any pre-fix snapshot remnants (committed without nydus labels)
sudo ctr --address=/run/containerd/containerd.sock --namespace nydus-runtest \
    images rm uhub.service.ucloud.cn/runc-serverless/nydus-alpine:3.20-nydus
sudo ctr --address=/run/containerd/containerd.sock --namespace nydus-runtest \
    snapshot --snapshotter nydus rm sha256:<bootstrap-digest>

# Pull via the transfer service (now has nydus unpack_config) so the
# manifest annotations are propagated to the snapshotter
sudo /usr/local/bin/nerdctl --address=/run/containerd/containerd.sock \
    --namespace nydus-runtest --snapshotter nydus pull \
    --platform linux/amd64 \
    uhub.service.ucloud.cn/runc-serverless/nydus-alpine:3.20-nydus

# Run -- snapshotter will spawn nydusd, which FUSE-mounts the rootfs by
# fetching blob chunks from US3 on demand
sudo /usr/local/bin/nerdctl --address=/run/containerd/containerd.sock \
    --namespace nydus-runtest --snapshotter nydus run \
    --rm --net=none --platform linux/amd64 \
    uhub.service.ucloud.cn/runc-serverless/nydus-alpine:3.20-nydus \
    sh -c 'echo HELLO_FROM_NYDUS; cat /etc/os-release | head -3; uname -a'
```

Output:

```
HELLO_FROM_NYDUS
NAME="Alpine Linux"
ID=alpine
VERSION_ID=3.20.10
Linux 10-60-207-36 6.12.35-061235-generic ...
```

`--net=none` skips nerdctl's default CNI bridge plugin requirement; the
test exercises the rootfs/FUSE path, not networking.

## Surface area to roll out / harden

| Item | Where |
|---|---|
| Replicate the two reader-side fixes to other workers when scaling out the snapshotter | Each `runc.ai/region-code=*` worker |
| RuntimeClass `nydus` so Pods get the snapshotter automatically via `runtimeClassName`, no explicit `--snapshotter nydus` needed | Cluster-level CRD |
| Migrate the deployment manifests' US3 creds out of the `ConfigMap` into a `Secret` and mount alongside the ConfigMap so `snapshotter.sh`'s `find /etc/nydus-snapshotter` picks both up | `nydus-system/nydus-snapshotter-credentials` |
| Add a containerd `imagePullSecret` on the worker to let kubelet pull `runc-serverless` images for real Pod tests | host `/etc/containerd/certs.d/...` or a `RegistryCredential` |

## Known-issue index

- "exec: \"sh\": executable file not found in $PATH" right after pull —
  almost always means the snapshot was committed WITHOUT nydus labels.
  Root cause is the `unpack_config` gap above. Verify with `ctr images
  ls` showing empty LABELS column on the nydus image.
- nydusd `Failed to load config: failed to parse configuration
  information` — check `region` is non-empty and `scheme` matches a
  reachable + TLS-trusted endpoint.
- nydusd starts but never opens api.sock — backend config likely
  unreachable; snapshotter log shows `Process <pid> has been a zombie`.
- `ENABLE_RUNTIME_SPECIFIC_SNAPSHOTTER=false` silently flips the
  cluster default snapshotter to nydus. Always set to `"true"` unless
  you really want that.
