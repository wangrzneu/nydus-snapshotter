# Test Record: k3s + containerd + nydus-snapshotter (2026-04-13)

## Scope

This record captures the end-to-end tests executed on **2026-04-13** for:

- nydus-snapshotter with system containerd and k3s containerd
- `nerdctl commit` generated OCI images
- idmap-related commit/repro flow
- zran trigger flow (`--oci-ref`) and runtime behavior
- known issues and mitigations

For upstream-review-friendly, distilled guidance of the ID-mapped runtime issue and
its configuration fix, see:

- `docs/k3s_containerd_idmap.md`

## Conversation coverage map

The following items map to the full discussion scope and current status:

- Rootless mode on host: **discussed, not fully closed as independent rootless containerd track** in this round.
  - Practical reason: later tests were switched to `k3s + containerd` real workload path (rootful runtime socket), then continued with image commit/zran/idmap/compatibility troubleshooting.
- idmap with real runtime: **covered** by k3s/containerd repro flow and script.
- k3s + containerd integration: **covered**.
- image commit capability:
  - `nerdctl commit` OCI image (non-idmap): **covered and pass**
  - `nydusify commit` in idmap scenario: **covered with known run-time issue pattern**
- whether committed OCI image can run through nydus snapshotter: **covered and pass** for non-idmap path.
- zran usage in test: **covered**.
- how to trigger zran mode: **covered**.
- overlayfs ChainID conflict cause and production impact: **covered**.

## Environment

- Host: `165.154.147.192`
- Date: `2026-04-13`
- nydusify: `v2.4.1`
- nydus-snapshotter service: `containerd-nydus-grpc --config /etc/nydus/config.toml`
- nydus-snapshotter fs driver: `fusedev`
- system containerd socket: `/run/containerd/containerd.sock`
- k3s containerd socket: `/run/k3s/containerd/containerd.sock`

Host/session timeline notes:

- temporary switch discussed to `165.154.200.220`, then reverted.
- final stable test host for this record: `165.154.147.192` after reboot recovery.

## Baseline checks

### Snapshotter plugin registration

Validated both runtimes expose `nydus` snapshotter plugin as `ok`.

Representative check commands:

```bash
sudo ctr plugins ls | grep snapshotter
sudo ctr --address /run/k3s/containerd/containerd.sock plugins ls | grep snapshotter
```

Result: `nydus` present and `ok` on both runtimes.

## Test A: `nerdctl commit` OCI image without idmap

### A1. System containerd path

Method:

1. Run a source container from `busybox`.
2. Write marker file in container.
3. `nerdctl commit` to new image tag.
4. Run image with `--snapshotter overlayfs`.
5. Run same image with `--snapshotter nydus`.
6. Inspect media types.

Representative commands:

```bash
sudo nerdctl run -d --name <src> docker.io/library/busybox:1.36 \
  sh -c 'echo committed-<ts> >/root/hello.txt; sleep 120'
sudo nerdctl commit <src> localhost/nydus-test/commit-oci:<ts>
sudo nerdctl run --rm --snapshotter overlayfs localhost/nydus-test/commit-oci:<ts> cat /root/hello.txt
sudo nerdctl run --rm --snapshotter nydus    localhost/nydus-test/commit-oci:<ts> cat /root/hello.txt
sudo nerdctl image inspect localhost/nydus-test/commit-oci:<ts> --mode=native
```

Result: PASS.

- `overlayfs` run succeeded
- `nydus` run succeeded
- content output matched
- image remained regular OCI/Docker manifest + tar.gz layers (not nydus-native layers)

### A2. k3s containerd path

Method: same as A1, but use k3s socket and namespace.

Representative commands:

```bash
sudo nerdctl --address /run/k3s/containerd/containerd.sock --namespace k8s.io \
  run -d --name <src> docker.io/library/busybox:1.36 \
  sh -c 'echo k3s-commit-<ts> >/root/hello.txt; sleep 120'
sudo nerdctl --address /run/k3s/containerd/containerd.sock --namespace k8s.io \
  commit <src> localhost/nydus-test/k3s-commit-oci:<ts>
sudo nerdctl --address /run/k3s/containerd/containerd.sock --namespace k8s.io \
  run --rm --snapshotter overlayfs localhost/nydus-test/k3s-commit-oci:<ts> cat /root/hello.txt
sudo nerdctl --address /run/k3s/containerd/containerd.sock --namespace k8s.io \
  run --rm --snapshotter nydus localhost/nydus-test/k3s-commit-oci:<ts> cat /root/hello.txt
```

Result: PASS.

Conclusion for Test A:

- `nerdctl commit` produced OCI images are usable by nydus snapshotter in current environment.
- This is compatibility behavior; it does not imply the image itself was converted to nydus-native format.

## Test B: idmap-related commit scenario (k3s)

Existing repro script in repo:

- `scripts/repro_k3s_idmap_commit.sh`

Script purpose:

- run idmapped container on nydus snapshotter
- commit via `nydusify commit`
- check image
- run committed image
- verify mount path

Observed issue pattern in runtime logs:

- stale/missing parent snapshot references
- `lazy parent recovery not supported for fs_driver="fusedev"`

Operational note:

- this failure pattern is metadata/reference consistency related (snapshot state), not a generic format incompatibility.

Current conclusion for idmap path in this round:

- `nydusify check`/mount-level verification can pass, but runtime execution in k3s path may still fail when parent snapshot metadata is stale/inconsistent.
- this was treated as runtime snapshot metadata consistency issue, not immediate evidence that idmap-committed artifact format is always invalid.

## Test C: How zran was triggered and validated

### C1. Trigger command

`nydusify convert --help` on host confirms:

- `--oci-ref  Convert to OCI-referenced nydus zran image`

Additional doc basis discussed:

- `README.md` states OCI lazy pulling can be supported via zran without explicit conversion in general capability terms.
- In this test, zran path was explicitly forced by `nydusify convert --oci-ref` to make runtime behavior directly observable.

Trigger conversion:

```bash
sudo nydusify convert \
  --source docker.io/library/busybox:1.36 \
  --target localhost:5000/nydus/busybox:zran-verify-<ts> \
  --platform linux/amd64 \
  --merge-platform \
  --oci-ref \
  --plain-http
```

### C2. Runtime conditions required

To allow runtime to select nydus alternative manifest from OCI index:

- `/etc/nydus/config.toml`
  - `[experimental] enable_index_detect = true`

For local HTTP registry (`localhost:5000`) in this environment:

- `/etc/nydus/config.toml`
  - `[remote] skip_ssl_verify = true`
- `/etc/nydus/nydusd-config.fusedev.json`
  - `device.backend.config.skip_verify = true`

Then restart service:

```bash
sudo systemctl restart nydus-snapshotter
```

### C3. Validation evidence

Run in a clean namespace to avoid snapshot key collision:

```bash
sudo nerdctl --address /run/k3s/containerd/containerd.sock --namespace nydus-zran3 \
  run --rm --snapshotter nydus \
  localhost:5000/nydus/busybox:zran-verify-<ts> \
  sh -c 'echo k3s-zran3-ok'
```

Observed:

- container run succeeded (`k3s-zran3-ok`)
- nydus-snapshotter logs contained:
  - `Found nydus alternative image in index for image: localhost:5000/nydus/busybox:zran-verify-...`
  - `Nydus remote snapshot ... is ready`

Conclusion for Test C:

- zran path was successfully triggered and used at runtime.

## Issues encountered and resolution

### 1) Permission denied on containerd socket

Symptom:

- `dial unix /run/containerd/containerd.sock: permission denied`

Resolution:

- run with `sudo`.

### 2) Intermittent SSH handshake instability

Symptom:

- `banner exchange: Connection ... invalid format`

Resolution:

- retry; transient network/sshd path issue.

### 3) `nerdctl pull --plain-http` unsupported on host version

Symptom:

- `unknown flag: --plain-http`

Resolution:

- use `k3s ctr images pull --plain-http` where needed.

### 4) Direct OCI-ref image pull unpack failure

Symptom:

- `no processor for media-type: application/vnd.oci.image.layer.nydus.blob.v1`

Resolution:

- use merge-platform flow and run via nydus snapshotter path; avoid forcing regular unpack semantics for nydus blob media type.

### 5) ChainID key collision with overlayfs snapshots

Symptom:

- `target snapshot "...": already exists`

Root cause:

- same namespace had existing overlayfs snapshots keyed by identical ChainID (`diffID`-derived).

Resolution:

- run tests in clean namespace (for example `nydus-zran3`).

### 6) Missing parent / stale snapshot references

Symptom:

- `missing parent ...`
- `lazy parent recovery not supported for fs_driver="fusedev"`

Resolution:

- cleanup stale snapshot references and/or use clean namespace for test isolation.

### 7) Index detection initially failed against HTTP local registry

Symptom:

- `http: server gave HTTP response to HTTPS client`
- `index detection failed`

Resolution:

- enable insecure/skip-verify settings as described in Test C2.
- subsequent logs showed `retrying with http for localhost:5000` and successful nydus alternative selection.

## Repair playbook used in this round

When runtime failed in mixed scenarios, the applied repair sequence was:

1. Confirm snapshotter plugin health (`ctr plugins ls`).
2. Isolate test namespace (for example `nydus-zran3`) to avoid snapshot key collisions.
3. Enable index detection and insecure local-registry settings in nydus config.
4. Restart `nydus-snapshotter`.
5. Re-run pull/run and verify nydus path from logs.

This sequence restored zran-index runtime path and avoided prior collision/interference in shared namespace tests.

## Production guidance

- Avoid mixing snapshotters in the same namespace for the same workload images.
- Use dedicated namespaces for validation/repro tasks.
- Keep snapshot metadata consistent during restart/upgrade (drain and cleanup strategy).
- Alert on:
  - `target snapshot ... already exists`
  - `missing parent`
  - `lazy parent recovery not supported`
  - repeated `index detection failed`

Production impact conclusion for ChainID conflict question:

- yes, this can happen in production if the same namespace reuses identical rootfs/diffID-derived keys across different snapshotter workflows (or after interrupted cleanup/restart paths).
- probability increases in long-lived shared namespaces with mixed snapshotter experiments.
- risk is manageable with namespace isolation, consistent snapshotter policy per workload group, and regular stale-snapshot cleanup.

## Artifacts in this repo

- Repro script: `scripts/repro_k3s_idmap_commit.sh`
- This record: `docs/test_record_k3s_containerd_nydus_2026-04-13.md`

---

## Addendum: 2026-04-15 (k3s + nydus native + S3 + idmap deep dive)

### Additional environment deltas

- k3s containerd generated config confirmed:
  - `snapshotter = "nydus"`
  - `disable_snapshot_annotations = false`
  - runtime handler `nydus-runc` added and used by Pod `runtimeClassName`.
- nydusd backend switched to S3 (endpoint `internal.s3-sg.ufileos.com`, scheme `http`, bucket `s3fstest`, prefix `nydus-test/`, region `sg`).

### New test matrix and outcomes

1. Non-idmap Pod with nydus-native image (`runtimeClassName: nydus-runc`, default hostUsers):
   - PASS (Pod Running).
   - In-container write to `/etc` succeeded.

2. idmap Pod with same nydus-native image (`hostUsers: false`):
   - FAIL.
   - Stable error pattern:
     - `create target of file bind-mount: mknod regular file ... /rootfs/etc/resolv.conf: permission denied`
     - or `/rootfs/etc/hosts: permission denied`

3. Minimal non-k8s repro (same host runtime):
   - FAIL with nydus:
     ```bash
     sudo CONTAINERD_SNAPSHOTTER=nydus k3s ctr run \
       --uidmap 0:<hostid>:65536 --gidmap 0:<hostid>:65536 --remap-labels \
       --mount type=bind,src=/tmp/hosts-test,dst=/etc/hosts,options=rbind:rw \
       docker.io/rancher/mirrored-library-busybox:1.36.1 <id> sh -lc 'sleep 300'
     ```
   - Error is identical `mknod ... /rootfs/etc/hosts: permission denied`.

4. Same command but **without** `--remap-labels`:
   - PASS with nydus.

5. Same command with `--snapshotter overlayfs` and `--remap-labels`:
   - PASS.

### Critical evidence

- nydus snapshot list/metadata for failing path contains `...-remap` intermediate snapshots.
- mount command for failing nydus path includes `uidmap/gidmap` and lowerdir chain with remap layer:
  - `lowerdir=.../snapshots/<remap>/fs:.../snapshots/<base>/fs`
- ownership check showed remap layer trees owned by mapped host UID/GID.
- manual experiment:
  - after creating failed container, `chown -R 0:0` on remap lower tree (and active upper root) allowed `ctr tasks start` to succeed.

### Conclusion (updated)

- The 2026-04-15 failure is **not** a generic RAFS/S3 native-format issue.
- It is specific to **nydus snapshotter + ID mapping labels/remap flow** in this environment.
- Non-idmap path remains healthy.
- overlayfs snapshotter does not reproduce this behavior under the same idmap bind-mount workload.

### Related known limitation still observed

- nydus blob media type (`application/vnd.oci.image.layer.nydus.blob.v1`) cannot be unpacked by default k3s/containerd stream processors directly, and fails with:
  - `no processor for media-type`
- This is separate from the idmap remap permission-denied issue above.

### Addendum update: 2026-04-15 root cause closure for idmap failure

After additional narrowing in the same environment, the idmap failure above was traced to
containerd proxy-plugin capability negotiation, not to RAFS/S3 format itself.

Root cause:

- in `k3s + containerd v2.0.2`, when proxy snapshotter config does not include
  `capabilities = ["remap-ids"]`, containerd idmap path can fall back to a
  compatibility remap flow (`<snapshot-key>-remap`).
- this fallback path reproduced the observed bind-mount file target creation failure:
  - `mknod ... /rootfs/etc/hosts: permission denied`
  - `mknod ... /rootfs/etc/resolv.conf: permission denied`

Fix applied and validated:

1. add capability in k3s template:
   - `/var/lib/rancher/k3s/agent/etc/containerd/config-v3.toml.tmpl`
   - under `[proxy_plugins.nydus]` add:
     - `capabilities = ["remap-ids"]`
2. restart k3s so generated containerd config includes the capability.
3. rerun minimal repro with `--uidmap/--gidmap --remap-labels` and bind mount.

Result after fix:

- repro command switched from FAIL to PASS in the same host runtime.
- nydus logs for the successful container key no longer showed fallback `-remap`
  snapshot chain for that run.

Final conclusion for this issue:

- for idmap/userns-remap scenarios with k3s/containerd + nydus snapshotter, declaring
  `remap-ids` in `[proxy_plugins.nydus]` is required to avoid false-negative runtime
  failures caused by compatibility remap fallback.

## Addendum: 2026-04-15 (minikube v1.36 + containerd v2 + hostUsers:false)

### Objective

- validate whether Kubernetes `hostUsers: false` really enters UID/GID remap path
- validate nydus-native image path in minikube
- validate commit/use flow from a running `hostUsers: false` container

### Environment

- minikube: `v1.36.0`
- Kubernetes: `v1.31.0`
- containerd: `v2.1.4`
- nydusify: `v2.4.1`
- nydus proxy plugin keeps `capabilities = ["remap-ids"]`

### Key finding #1: why `hostUsers: false` previously looked ineffective

When minikube was started without explicit feature gates:

- Pod could be created with client annotation containing `hostUsers: false`
- but stored Pod spec did not honor `hostUsers`
- sandbox inspect showed:
  - `uidMappings = null`
  - `gidMappings = null`

Root reason:

- `UserNamespacesSupport` gate not enabled on control-plane/kubelet path.

### Minikube fix used

Recreate minikube with:

```bash
sudo minikube start \
  --driver=none \
  --container-runtime=containerd \
  --kubernetes-version=v1.31.0 \
  --extra-config=apiserver.feature-gates=UserNamespacesSupport=true \
  --extra-config=kubelet.feature-gates=UserNamespacesSupport=true
```

After fix:

- Pod spec preserved `hostUsers: false`
- sandbox inspect returned non-empty UID/GID mappings (`size=65536`).

### Key finding #2: backend mismatch can mask runtime validation

With `nydusd` backend configured as S3, running public registry nydus-native sample
image could fail with rootfs read errors in non-idmap path (`input/output error` on
`/etc/passwd`), which is independent of hostUsers/userns logic.

For deterministic validation in this round:

- switched nydusd config to registry backend template
- converted `busybox:1.36` to local nydus-native image in `localhost:5000`
- tested pods against local converted image.

### Validation matrix and result

Image used:

- `localhost:5000/nydus/busybox:minikube-v136` (nydus-native, locally converted)

Results:

1. Pod `runtimeClassName: nydus-runc` (default host users):
   - PASS (`Running`)
2. Pod `hostUsers: false` + `runtimeClassName: nydus-runc`:
   - PASS (`Running`)
   - sandbox `uidMappings/gidMappings` are non-empty
3. `nerdctl commit` from running hostUsers:false pod container:
   - PASS
   - committed image runnable with `--snapshotter nydus`
4. committed image used again in new `hostUsers: false` pod:
   - PASS (`Running`)

Additional runtime signal:

- `ctr -n k8s.io snapshots --snapshotter nydus ls | grep remap` returned no remap
  fallback keys in this validated path.

### Outcome

- In minikube, `hostUsers: false` requires explicit `UserNamespacesSupport=true`.
- With that gate enabled and `remap-ids` capability declared, nydus path can run
  hostUsers:false workload and commit/use flow successfully in this environment.

## Addendum: 2026-04-21 (backend matrix for idmapped overlay)

### Objective

Validate which nydus fs_driver backends can serve as the lower filesystem of an
idmapped overlay mount (containerd `doPrepareIDMappedOverlay` path), and encode
the unsupported case as a fast-fail guard in the snapshotter.

### Host

- `ubuntu@165.154.147.192`
- kernel: `6.12.35` (HWE)
- containerd v2.0.2, k3s path unchanged from previous addendum.

### rafs (fusedev) — not supported

- nydusd does **not** advertise `FUSE_ALLOW_IDMAP`, so the kernel refuses to
  idmap a rafs-over-FUSE lower. This is a kernel-level gate, not a snapshotter
  defect.
- Added fast-fail guard in snapshotter: when a snapshot has
  `containerd.io/snapshot/uidmapping` / `gidmapping` labels and the rafs
  instance runs with `fs_driver="fusedev"`, `Prepare`/`Mounts` returns:
  ```
  snapshot <id>: rafs fusedev backend does not support idmapped mounts;
    use fs_driver="fscache" (EROFS, Linux 6.5+) or a non-nydus image
  ```
- Implementation: `snapshot/snapshot.go:checkIDMapBackendCompat` + unit tests
  in `snapshot/snapshot_test.go:TestCheckIDMapBackendCompat`.

### fscache (EROFS) — not validated on this host

- kernel 6.12.35 HWE image ships without `CONFIG_CACHEFILES_ONDEMAND=y`, which
  nydusd fscache driver requires. `cachefilesd` / `/dev/cachefiles` not usable.
- Not a code defect; this backend is expected to work on kernels that enable
  the on-demand cachefiles config. Left for a future validation host.

### blockdev (tarfs, EROFS-on-loop) — PASS

Enabled on the host by flipping `/etc/nydus/config.toml`:

```toml
enable_tarfs = true
mount_tarfs_on_host = true
```

Pod used `imagePullPolicy: Always` to force a fresh pull under the new config
(image `alpine:3.20` in `idmap-tarfs-alpine`).

Evidence gathered from the running pod:

- snapshotter log:
  - `Prepare active snapshot k8s.io/241/... in Nydus tarfs mode`
  - `nydus tarfs for snapshot 681 is ready`
- host `losetup`:
  - `/dev/loop1 → .../snapshots/681/fs/image/image.boot`
- host mount:
  - `/dev/loop1 on .../snapshots/681/mnt type erofs (ro,relatime,user_xattr,acl,cache_strategy=readaround)`
- pod's rootfs overlay:
  - `lowerdir=/tmp/ovl-idmapped4198761186/0,upperdir=.../snapshots/682/fs,...,xino=off`
  - the `/tmp/ovl-idmapped*` prefix confirms containerd idmap-wrapped the EROFS
    lower via `open_tree(OPEN_TREE_CLONE)` + `mount_setattr(MOUNT_ATTR_IDMAP)`.
- pod `uid_map` inside userns: `0 1571946496 65536`
- `stat /bin/sh` inside container: `Uid: (0/root) Gid: (0/root)` — idmap
  correctly translated container-uid 0 to host-uid 1571946496.

### Conclusion

- rafs/fusedev: unsupported; now rejected early with actionable error.
- tarfs/blockdev: supported end-to-end on Linux 6.5+ with idmapped overlay
  over EROFS-on-loop lower.
- fscache: expected to work (EROFS lower supports FS_ALLOW_IDMAP); this host's
  kernel config blocks nydusd fscache startup. Re-validate on a kernel with
  `CONFIG_CACHEFILES_ONDEMAND=y`.
- Matrix matches `docs/kubernetes_user_namespaces.md` §1 "Supported backends".
