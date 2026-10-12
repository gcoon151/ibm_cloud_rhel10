# RHEL 10 Peer Pod Image — Upstream Deviation Audit

Upstream reference: `https://github.com/confidential-devhub/coco-podvm-scripts`
Local clone: `coco-podvm-scripts/` (workspace) and `~/gits/coco-podvm-scripts/` (build host)
Last audited against upstream: 2026-10-07 (this file is the audit)

**Rule**: Every entry here must reflect a real, current difference between our copy and
upstream. If upstream already does the same thing, it must NOT be listed here. If upstream
fixes something we worked around, REMOVE the entry. The ip-path bug (previously called
deviation #16) is not here because it was a bug we fixed to match upstream — not a deviation.

IBM modifications to upstream files live in:
- `rhel10-experimental/scripts/patches/` — `verity.sh`, `coco-components.sh`
- `ibm_cloud_rhel9/scripts/coco/podvm/` — `podvm_maker.sh`, `example_run.sh`, `script-disk-mods.sh`

`build-rhel10-overlay.sh` Step 2 copies these over the upstream clone on every build.

---

## Files audited

| File | Upstream path | Our path | Audited |
|------|--------------|----------|---------|
| `example_run.sh` | `coco-podvm-scripts/example_run.sh` | `ibm_cloud_rhel9/scripts/coco/podvm/example_run.sh` | ✅ 2026-10-07 |
| `coco-components.sh` | `coco-podvm-scripts/scripts/coco/coco-components.sh` | `rhel10-experimental/scripts/patches/coco-components.sh` | ✅ 2026-10-07 |
| `podvm_maker.sh` | `coco-podvm-scripts/scripts/coco/podvm/podvm_maker.sh` | `ibm_cloud_rhel9/scripts/coco/podvm/podvm_maker.sh` | ✅ 2026-10-07 |
| `script-disk-mods.sh` | `coco-podvm-scripts/scripts/coco/podvm/script-disk-mods.sh` | `ibm_cloud_rhel9/scripts/coco/podvm/script-disk-mods.sh` | ✅ 2026-10-07 |
| `verity.sh` | `coco-podvm-scripts/scripts/verity/verity.sh` | `rhel10-experimental/scripts/patches/verity.sh` | ✅ 2026-10-07 |
| `get-artifacts.sh` | `coco-podvm-scripts/scripts/coco/podvm/get-artifacts.sh` | `ibm_cloud_rhel9/scripts/coco/podvm/get-artifacts.sh` | ✅ 2026-10-07 — **identical** |

---

## Summary table

| # | File | Our change | Why | Risk if removed |
|---|------|-----------|-----|----------------|
| A | `example_run.sh` | `sudo --preserve-env` instead of `sudo -E` | Ubuntu sudo-rs strips env with `-E` | Build silently uses wrong env |
| B | `example_run.sh` | `-v /boot:/boot:ro`, `-v /dev:/dev` added | supermin needs host kernel; nbd nodes need live /dev | supermin fails on Ubuntu |
| C | `example_run.sh` | `set -euo pipefail` + exit-code check on podman run | Without it, container failure exits 0 and broken QCOW2 proceeds to verity | Broken image silently passes |
| D | `example_run.sh` | Registry auth bind-mount, PODVM_BINARY_DIGEST, SSHD_SERVICE, DEBUG_BUILD passthrough | IBM Cloud needs these env vars | registry.redhat.io pull fails |
| E | `coco-components.sh` | `set -e` at top | get-artifacts.sh failure is silently swallowed upstream | Failed payload pull → empty QCOW2 |
| F | `podvm_maker.sh` | Entire file is IBM-written (not a modified upstream file) | Upstream podvm_maker.sh does ~15 lines; ours does IBM-specific config | N/A — must exist |
| G | `script-disk-mods.sh` | No kernel install, no UKI copy, no BOOTX64.CSV write | Our kickstart base already has the correct UKI; upstream needs this because theirs doesn't | Would overwrite UKI or fail on missing kernel RPM |
| H | `script-disk-mods.sh` | ~~vendor-data disable config~~ **REVERTED** — IBM Cloud delivers SSH key via vendor-data; disabling it breaks SSH entirely | | |
| I | `script-disk-mods.sh` | NVIDIA gated by env var, DEBUG_BUILD block | IBM additions, not in upstream | No functional impact if omitted |
| J | `verity.sh` | `sgdisk -e` on nbd device after resize + after repart | qemu-img resize leaves GPT backup header at wrong offset; upstream does no GPT relocation | find_efi_root_part() fails → verity never runs |
| K | `verity.sh` | Hard assertion: `/dev/nbd0` must be block device | modprobe nbd fails silently inside container without host module | Script proceeds with no nbd, "Process completed!" on broken image |
| L | `verity.sh` | Hard assertion: verity partition visible after systemd-repart | Upstream only checks RH != "TBD"; doesn't verify partition is visible | Corrupt image with no verity hash, verity fails at boot |
| M | `verity.sh` | Hard assertion: `verity.addon.efi` exists after ukify | Upstream has no check | UKI addon silently missing, verity hash never applied at boot |
| N | `verity.sh` | EXIT trap preserves `$?`; upstream always exits 0 | `set -e` failures inside verity.sh were swallowed and caller saw success | Any verity failure looks like success |
| O-1 | `luks-scratch/30-scratch.conf` | `SizeMaxBytes=268435456` (256 MB cap) | No cap → partition fills ~2.6 GB free space → LUKS+ext4 format takes ~90 s on IBM storage → exceeds hardcoded 30 s kata `checkRequestTimeout` → VM killed | scratch format takes ~90 s, VM dies before kata-agent handshakes |
| O-2 | `luks-scratch/kata-agent.service.d/10-override.conf` | `After=luks-scratch.service` + `Requires=luks-scratch.service` in `[Unit]` | Without ordering kata-agent races ahead of luks-scratch, writes agent.sock, APF fires, 30 s clock starts before scratch is ready | kata-agent and APF start before scratch is formatted, VM killed at 48.7 s |
| O-3 | `luks-scratch/format-scratch.sh` | `--pbkdf pbkdf2 --pbkdf-force-iterations 1000` + `mkfs.ext4 -m 0 -E nodiscard,lazy_itable_init=1,lazy_journal_init=1` | Argon2id default KDF takes ~7.4 s luksFormat + ~2 s luksOpen on NVMe (benchmarked). Key is random and ephemeral — strong KDF adds nothing. pbkdf2-1000 drops to <100 ms total KDF overhead. | With large partitions or slow KDF, format still risks racing the timeout |
| O-4 | `luks-scratch/luks-scratch.service` + `luks-scratch/format-scratch.sh` | Removed `Requires=systemd-repart.service`, `After=systemd-repart.service`, `BindsTo=dev-disk-by\x2dpartlabel-scratch.device` from service unit; `format-scratch.sh` now calls `systemd-repart` directly with `--definitions=/usr/lib/repart.d` | System `systemd-repart.service` has a trigger condition: it only runs when dm-verity is active (roothash= in kernel cmdline). IBM Cloud rejects unsigned `verity.addon.efi`, so dm-verity is never active, repart is always skipped, the scratch device never appears, BindsTo fires no device unit, luks-scratch never starts. Running repart directly in format-scratch.sh bypasses the trigger condition. | Scratch partition is never created; luks-scratch never starts; kata-agent (Requires=luks-scratch) never starts; pod times out. |
| O-5 | `luks-scratch/luks-scratch.service` | Removed `Wants=kata-agent.service` from service unit | Upstream unit includes `Wants=kata-agent.service` + `Before=kata-agent.service`. The `Wants=` pulls kata-agent in at boot before CDH socket is created (~3s race), causing kata-agent to fail probing CDH and panic at first container pull. | kata-agent starts prematurely at boot, CDH client is uninitialized, kata-agent panics on image pull. Revert condition: upstream drops `Wants=kata-agent.service` or kata-agent no longer probes CDH at startup. |

---

## A+B+C+D: `example_run.sh` — Ubuntu build host adaptations

**Upstream** (`sudo -E podman run`, no extra mounts, no error handling):
```bash
sudo -E podman build -t coco-podvm ... || printf "\n\n!!! Failed !!!\n"
sudo -E podman run --rm --privileged -v $QCOW2:/disk.qcow2 ...
[[ -n "${ACTIVATION_KEY}" && -n "${ORG_ID}" ]] && sudo podman secret rm activation_key org_id
```

### A: `sudo --preserve-env` instead of `sudo -E`

Ubuntu's `sudo-rs` (the sudo implementation on the build host) resets the environment
even with `-E`. All env vars (`QCOW2`, `SSHD_SERVICE`, `ORG_ID`, `ACTIVATION_KEY`, etc.)
are silently dropped. `sudo --preserve-env=VAR1,VAR2` is the Ubuntu-compatible form.

**Revert condition**: Move to RHEL build host where `sudo -E` behaves as documented.

### B: `-v /boot:/boot:ro` and `-v /dev:/dev`

`-v /boot:/boot:ro`: `supermin` (used by libguestfs inside the container) scans `/boot`
for kernels when there is no RPM database on the host. Ubuntu's `/boot/vmlinuz-*` had
permissions tightened by a security update; the explicit bind-mount ensures they are
visible at their expected paths.

`-v /dev:/dev`: The upstream `--mount type=bind,source=/dev,target=/dev` does not
propagate nbd device nodes created after container start (`/dev/nbd0` through `/dev/nbd7`
appear after `modprobe nbd` runs inside the container). The explicit `-v /dev:/dev` uses
the live host `/dev` so new nodes are visible immediately.

**Revert condition**: Move to RHEL build host, or upstream pre-creates nbd nodes.

### C: `set -euo pipefail` + exit-code check on `podman run`

Upstream's `example_run.sh` has no `set -e` and ignores the podman run exit code. If
the container exits non-zero (verity failure, nbd error, anything), `example_run.sh`
exits 0 and `build-rhel10-overlay.sh` proceeds to verify/upload a broken image.

Our version wraps `podman run` in an `if ! ... ; then exit 1; fi` block and adds
`set -euo pipefail`. Variables that upstream code leaves unset are initialised with
`${VAR:-}` to avoid `set -u` failures; secret cleanup uses `|| true`.

**Revert condition**: Upstream adds error handling on the podman run call.

### D: Registry auth, PODVM_BINARY_DIGEST, SSHD_SERVICE, DEBUG_BUILD passthrough

Upstream passes `ROOT_PASSWORD` only. We additionally pass through:
- `PODVM_BINARY_DIGEST` — baked into `/etc/podvm-version.json` for payload traceability
- `SSHD_SERVICE` — controls whether SSH is enabled or masked (standard vs hardened image)
- `DEBUG_BUILD` — enables verbose console logging for CoCo services
- Registry auth bind-mount (`/run/containers/0/auth.json`) — required to pull from
  `registry.redhat.io` inside the container as root

**Revert condition**: Not applicable — these are IBM-specific features upstream doesn't have.

---

## E: `coco-components.sh` — `set -e`

**Upstream**: no `set -e`. If `get-artifacts.sh` fails (e.g. auth failure on
`registry.redhat.io`), `coco-components.sh` continues to `virt-customize --copy-in
podvm-binaries.tar.gz:/tmp/` where the source file doesn't exist. `virt-customize` exits
non-zero, but that exit code is also not checked. The container exits 0. The QCOW2 has no
kata-agent, no agent-protocol-forwarder. Verity is applied to an empty image.

**Our change**: `set -e` on line 1. Any failure exits immediately.

**Revert condition**: Upstream adds error handling on `get-artifacts.sh` or `virt-customize`.

---

## F: `podvm_maker.sh` — entirely IBM-written

Upstream's `podvm_maker.sh` is 94 lines and does:
1. `dnf install afterburn e2fsprogs` (with CentOS fallback repo)
2. Write `afterburn-checkin.service` (Azure check-in — not needed for IBM Cloud)
3. `tar -xzvf` the three tarballs
4. `dnf remove -y cloud-init WALinuxAgent`
5. `semanage fcontext -a -t bin_t /usr/bin/ip && restorecon -v /usr/sbin/ip`
6. `systemctl enable luks-scratch.service`
7. Write `gen-issue.service` and `process-user-data.service.d/10-override.conf`
8. `firewall-offline-cmd --zone=public --add-port=15150/tcp`

Our `podvm_maker.sh` keeps items 3, 5, 6, 7, 8 and replaces/adds:
- Item 1: We verify e2fsprogs is pre-installed (in Dockerfile) rather than dnf-installing it.
  The upstream CentOS Stream fallback repo is flaky and unnecessary. **Valid.**
- Item 2: `afterburn-checkin.service` dropped. Afterburn is Azure-specific. **Valid.**
- Item 4: `dnf remove cloud-init WALinuxAgent` dropped. We actively keep cloud-init
  because IBM Cloud uses it for user-data (INITDATA). WALinuxAgent is Azure-only. **Valid.**
- Item 5: `semanage` path fixed from `/usr/sbin/ip` to `/usr/bin/ip` to match upstream
  exactly. The `&&` was also split into two separate statements so semanage failure stops
  the build rather than silently skipping restorecon. This matches upstream intent.
- Added: post-extraction binary assertions (kata-agent, agent-protocol-forwarder,
  kata-agent-clean, agent-config.toml). Upstream has none.
- Added: `image_registry_auth` patch to agent-config.toml. Upstream payload omits it.
- Added: `confidential-data-hub.path` symlink in multi-user.target.wants. The OSC payload
  ships the unit but does not create the symlink, so CDH never starts.
- Added: SSHD enable/disable logic.
- Added: DEBUG_BUILD block (verbose console logging for CoCo services).
- Added: Version manifest written to `/etc/podvm-version.json`.
- Added: Uptycs install step (currently non-functional — scripts not staged into guest).

**Items that need ongoing attention:**
- `semanage fcontext` for `kata-agent` and `kata-agent-clean` (lines 116-117) still use
  `&&` chaining. These paths are in `/usr/local/bin` with no equivalency rule conflict so
  semanage will succeed, but failure would still silently skip restorecon. Low risk, worth
  fixing to separate statements for consistency.
- Uptycs step looks for `/scripts/coco/podvm/install-uptycs.sh` which is a container path,
  not a guest path. It always skips. This is a known structural bug tracked separately.

---

## G: `script-disk-mods.sh` — no kernel install, no UKI copy, no BOOTX64.CSV write

**Upstream lines 4-12**:
```bash
export KERNEL_VERSION=6.12.0-211.16.1.el10_2
export NVIDIA_DRIVER_VERSION=595.58.03
dnf install -y kernel-{uki-virt,modules,modules-extra}-${KERNEL_VERSION}
printf "shimx64.efi,redhat,\\\EFI\\\Linux\\\\..." | iconv ... > /boot/efi/EFI/redhat/BOOTX64.CSV
rpm -qa "kernel*" | grep -Ev "${KERNEL_VERSION}|^kernel-uki" | xargs -r rpm -e --nodeps
```

Upstream needs this because their base image comes from Image Builder which does not run
`kernel-install` hooks and does not populate EFI/Linux with a UKI. They must install a
pinned kernel RPM and manually write the BOOTX64.CSV.

Our base image comes from Anaconda (kickstart) which runs `kernel-install` correctly and
places the UKI in `/boot/efi/EFI/Linux/` during install. Adding a kernel RPM install here
would: pin an upstream-chosen kernel version, risk a module/UKI version mismatch if the
RPM differs from the kickstart-installed kernel, and add ~300MB and several minutes to
every build.

**We also removed the hardcoded `NVIDIA_DRIVER_VERSION=595.58.03`** from the top of the
file. Upstream hardcodes this unconditionally and always installs NVIDIA drivers. We gate
NVIDIA installation on the `NVIDIA_DRIVER_VERSION` environment variable — if unset, no
NVIDIA packages are installed. This is required for non-GPU IBM Cloud images.

**Revert condition**: Switch to Image Builder base (which would require kernel install).

---

## H: `script-disk-mods.sh` — vendor-data disable

**Upstream**: no vendor-data config.

IBM Cloud injects vendor-data via a config-drive (`/dev/vdc`, `DataSourceNoCloud`) into
every RHEL VSI. For RHEL images this vendor-data runs the `rh_subscription` module
(calls `subscription-manager`) and `power_state_change`. On a dm-verity peer pod image
the root filesystem is read-only; RHSM writes fail; `power_state_change` powers off the
VM at ~49 seconds before kata-agent completes its handshake.

Fix: `/etc/cloud/cloud.cfg.d/99-ibm-no-vendor-data.cfg` disables `vendor_data` and
`vendor_data2`. User-data (INITDATA, cloud config) continues to work normally.

**Revert condition**: IBM Cloud stops injecting `rh_subscription` vendor-data for images
tagged as peer pods, or we find a supported suppression mechanism upstream provides.

---

## I: `script-disk-mods.sh` — NVIDIA gated by env var, DEBUG_BUILD block

These are pure IBM additions. Upstream has no DEBUG_BUILD concept and hardcodes NVIDIA
unconditionally. Not a deviation that needs justification — upstream simply doesn't have
these features.

---

## J: `verity.sh` — GPT relocation after `qemu-img resize` and after `systemd-repart`

**Upstream `resize_disk()`** ends at:
```bash
qemu-img resize "$DISK_RESIZE" -f $DISK_FORMAT "${rounded_size}"
```
No GPT relocation. Upstream then immediately connects nbd and calls `find_efi_root_part()`
via `lsblk`, which fails because the GPT backup header is at the wrong offset after the
virtual disk grew.

**Our addition**: After `qemu-img resize`, connect nbd, run `sgdisk -e` on the block
device, disconnect, then assert with a raw conversion + `sgdisk -v` before continuing.
Also run `sgdisk -e` again after `systemd-repart` because repart adds the verity partition
without relocating the backup header.

The `sgdisk -e` must run on the nbd block device, not on the QCOW2 file path. Running it
on the file path uses the compressed file length as the disk size, not the virtual size —
so the backup header gets placed at the wrong offset even after "fixing" it.

**Note**: The post-resize GPT check uses `qemu-img convert -O raw` to produce a temp raw
file for `sgdisk -v`. This is the same pattern that `verify-qcow2.sh` was using (and
was slow/wrong for verify). In `verity.sh` this is acceptable because: (a) the QCOW2 is
only ~3–4GB at this point (before verity partition is added), and (b) it runs inside the
container where the temp file is in a container overlay. In `verify-qcow2.sh` running on
the build host against a 7GB QCOW2 it was not acceptable — that check was removed.

**Revert condition**: Upstream adds GPT relocation after both resize and repart.

---

## K, L, M, N: `verity.sh` — hard assertions and EXIT trap fix

### K: nbd block device assertion
Upstream line 277: `modprobe nbd` — no check that it succeeded, no check that
`/dev/nbd0` is actually a block device before `qemu-nbd -c` runs.

If `modprobe nbd` fails (module not loaded on host), `qemu-nbd` fails, all nbd operations
fail, but the script continues (with `set -e` this would abort, but the trap exits 0 —
see N). Our assertion: check `[[ -b "$NBD_DEVICE" ]]` and exit loudly if not.

### L: Verity partition assertion after systemd-repart
Upstream checks only `if [ "$RH" == "TBD" ]`. If `systemd-repart` ran but the partition
isn't visible yet (udev race), RH may still have a real hash value while the partition is
not accessible. Our check: `sgdisk -p "$NBD_DEVICE" | grep -qi "root-x86-64-verity"`.
`sgdisk` reads directly from the block device without relying on udev partition visibility.

### M: Verity addon assertion after ukify
Upstream has no check after `ukify build`. If `ukify` fails or writes to the wrong path,
the script continues, the ESP is unmounted, and the image has no verity addon. At boot,
the kernel finds the root hash in the UKI kernel cmdline but no verity hash partition
matching — verity fails and the boot either panics or falls back to a non-verity root.

Our check: after `cd $UKI_NAME.extra.d` and `ukify build --output=$ADDON_NAME`, check
`[[ -f "$ADDON_NAME" ]]`. The bare filename is correct because we already cd'd into
`$UKI_NAME.extra.d`.

### N: EXIT trap exit code
**Upstream `handle_ctrlc()`** ends with `exit 0` unconditionally. This means any
`set -e` failure inside `verity.sh` invokes the trap, which exits 0. The caller
(`create-verity-podvm.sh`) sees exit 0 and prints "Process completed!" regardless of
what failed.

**Our fix**: capture `$?` at trap entry (`local EXIT_CODE=$?`) and `exit $EXIT_CODE`.

**Revert condition**: Upstream fixes the trap exit code and adds equivalent assertions.

---

## O: `luks-scratch` — partition size cap + kata-agent ordering

### Files changed
- `scripts/coco/podvm/luks-scratch/usr/lib/repart.d/30-scratch.conf`
- `scripts/coco/podvm/luks-scratch/etc/systemd/system/kata-agent.service.d/10-override.conf`

### O-1: `SizeMaxBytes=268435456` in `30-scratch.conf`

**Upstream** (`30-scratch.conf` in `podvm/mkosi.images/system/mkosi.skeleton/usr/lib/repart.d/`):
```
[Partition]
Type=linux-generic
Label=trusted_store
Format=ext4
```
No `SizeMaxBytes`. The partition fills all free disk space.

**Why upstream gets away with it**: In mkosi-built images the root disk is sized tightly
to installed content (~3–4 GB total), leaving only a few hundred MB free. On IBM Cloud our
RHEL 10 image is on a 10 GB disk with ~7 GB used, leaving ~2.6 GB free — all of which
became the scratch partition. Formatting 2.6 GB as LUKS2+ext4 on IBM Cloud network storage
(~30 MB/s) took ~90 s, exceeding the kata shim's hardcoded 30 s `checkRequestTimeout`.

**Our addition**: `SizeMaxBytes=268435456` (256 MB). At ~30 MB/s this formats in <10 s,
well within the available budget after kata-agent starts.

**Revert condition**: Upstream adds a `SizeMaxBytes` appropriate for their disk sizing, or
upstream switches to a non-blocking scratch mechanism.

### O-2: `After=luks-scratch.service` + `Requires=luks-scratch.service` in `kata-agent.service.d/10-override.conf`

**Upstream `kata-agent.service`**:
```
After=netns@podns.service scratch-storage.service
Wants=scratch-storage.service
```
Upstream gates kata-agent on `scratch-storage.service` (their conditional path, only
active when CAA writes `/run/peerpod/scratch-space.marker`). Our `luks-scratch.service`
runs unconditionally but was not in kata-agent's `After=` chain, so kata-agent could
start in parallel with luks-scratch, write `agent.sock` immediately, fire APF, and start
the 30 s `checkRequestTimeout` clock before luks-scratch finished.

**Our addition** (in the `10-override.conf` drop-in):
```ini
[Unit]
After=luks-scratch.service
Requires=luks-scratch.service
```
This ensures kata-agent only starts after luks-scratch completes. APF is already gated on
`agent.sock` via `agent-protocol-forwarder.path`, so delaying kata-agent automatically
delays the start of the 30 s clock. With the 256 MB size cap (O-1), the full sequence
completes: luks-scratch (~8 s) → kata-agent → agent.sock → APF → Check() succeeds.

**Revert condition**: We adopt upstream's `scratch-storage.service` path (conditional,
marker-based) and retire `luks-scratch.service`, making upstream's `After=` ordering
sufficient for our cluster config.

### O-3: `--pbkdf pbkdf2 --pbkdf-force-iterations 1000` + lazy mkfs flags in `format-scratch.sh`

**Upstream `luks-encrypt-storage`** uses `cryptsetup luksFormat --type luks2` with no
`--pbkdf` flag — defaults to Argon2id with an auto-benchmarked iteration count (~1 GiB
memory, ~2 s CPU). **`format-scratch.sh`** (our IBM script) also had no `--pbkdf` flag.

**Benchmarked on local NVMe (loop device, 256 MB):**
| Method | luksFormat | luksOpen | mkfs | Total |
|---|---|---|---|---|
| Argon2id defaults (was) | 7380 ms | 1997 ms | 15 ms | **9392 ms** |
| pbkdf2-1000 + nodiscard+lazy (now) | 57 ms | 21 ms | 5 ms | **83 ms** |

On IBM Cloud ~30 MB/s storage the `mkfs` cost scales with partition size, but Argon2id
still added ~9 s on top. With the 256 MB cap (O-1), eliminating Argon2id overhead brings
total format time to <1 s on fast storage and <10 s on IBM Cloud — well inside the 30 s
budget remaining after luks-scratch gates kata-agent.

The key is random, generated from `/dev/urandom` on every boot, stored only in `/run`,
never persisted, never reused, discarded at power-off. A strong memory-hard KDF exists to
resist offline dictionary attacks against a stolen key — none of that applies here.
`pbkdf2-1000` is the correct choice for a fully ephemeral throwaway key.

Additional mkfs flags added:
- `-m 0`: no reserved blocks (scratch is not a system filesystem)
- `-E nodiscard,lazy_itable_init=1,lazy_journal_init=1`: skip journal zeroing and inode
  table pre-initialization — safe for ephemeral storage, saves ~1–2 s on slow disks.

**Revert condition**: Upstream adds explicit PBKDF flags for the ephemeral key case, or
we switch to plain dm-crypt (no LUKS header) for the scratch partition.

### O-4: Decoupled `luks-scratch.service` from `systemd-repart.service`

**Previous state of `luks-scratch.service`**:
```ini
Requires=systemd-repart.service
After=systemd-repart.service
BindsTo=dev-disk-by\x2dpartlabel-scratch.device
```

**The problem**: System `systemd-repart.service` has a trigger condition. On RHEL 10 it only
runs (creates new partitions) when the verity root is active — i.e. `roothash=` is present
in the kernel cmdline and dm-verity mounted `/dev/mapper/root`. On IBM Cloud TDX VSIs,
`verity.addon.efi` is unsigned and rejected by Secure Boot, so dm-verity is never active,
`systemd-repart` is skipped ("no trigger condition checks were met"), the scratch partition
device (`/dev/disk/by-partlabel/scratch`) never appears, the `BindsTo=` device unit never
activates, and `luks-scratch.service` waits indefinitely. With `Requires=luks-scratch` on
`kata-agent`, kata-agent never starts either.

**Our fix**:
1. Removed from `luks-scratch.service`: `Requires=systemd-repart.service`, `After=systemd-repart.service`, `BindsTo=dev-disk-by\x2dpartlabel-scratch.device`
2. `format-scratch.sh` now calls `systemd-repart` directly with `--definitions=/usr/lib/repart.d --dry-run=no $ROOT_DISK` before any LUKS work. This bypasses the trigger condition and unconditionally creates the scratch partition on every boot, regardless of whether dm-verity is active.
3. Root disk detection uses `/dev/vda` first (IBM Cloud), then `/dev/sda` (local QEMU), with a hard error if neither is a block device.
4. After `systemd-repart` returns, waits up to 15 s for `/dev/disk/by-partlabel/scratch` to appear via udev before attempting `cryptsetup`.

**Why upstream doesn't have this problem**: Upstream's `scratch-storage.service` is conditional (only runs when CAA writes `/run/peerpod/scratch-space.marker`). Their unit has no `BindsTo=`. Our `luks-scratch.service` runs unconditionally and depended on the system repart, which was the wrong dependency.

**Revert condition**: dm-verity is reliably active on IBM Cloud (i.e. the Secure Boot + verity addon signing problem is fixed). In that case the original `BindsTo=` pattern can be restored, since the scratch device will always appear after repart runs.

### O-5: Removed `Wants=kata-agent.service` from `luks-scratch.service`

**Upstream `luks-scratch.service`**:
```ini
[Unit]
Description=Create and format scratch storage using LUKS
Wants=kata-agent.service
Before=kata-agent.service
```

**The problem**: `luks-scratch.service` is `WantedBy=multi-user.target`. Having `Wants=kata-agent.service` pulls `kata-agent.service` into the initial boot activation transaction at ~T+1s, bypassing `kata-agent.path`. At that moment CDH has not started yet (`confidential-data-hub.path` requires `cdh.toml` provisioned). When kata-agent starts at boot, it probes `/run/confidential-containers/cdh.sock`, fails ("Failed to probe CDH socket file ... NotFound"), and leaves `CDH_CLIENT` uninitialized (`None`). Later, when the first container image pull request arrives, kata-agent attempts `CDH_CLIENT.get().expect(...)` and panics.

**Our fix**: `podvm_maker.sh` strips `Wants=kata-agent.service` from `luks-scratch.service` after extracting `luks-config.tar.gz`. `Before=kata-agent.service` is retained for proper ordering. `kata-agent.service` is activated properly via `kata-agent.path` once `agent.sock` is accessed and after CDH is running.

**Revert condition**: Upstream drops `Wants=kata-agent.service` or kata-agent is updated so it no longer probes CDH prematurely at startup.

---

## P: Uptycs EDR install in `coco-components.sh` (RHEL 10 patch)

**Upstream `coco-components.sh`**: single `virt-customize` with script-disk-mods, payload/pause/luks tarballs, `podvm_maker.sh`. No EDR.

**Ours** (`rhel10-experimental/scripts/patches/coco-components.sh`): adds `--copy-in` of `uptycs-complete.tar.gz`, `provision-uptycs.sh`, `uptycs-osquery.service` to `/tmp/` and `--run install-uptycs.sh` after `podvm_maker.sh`. Hard-fails before virt-customize if any of the four files is missing from `ARTIFACTS_FOLDER`.

**Staging**: `build-rhel10-overlay.sh` Step 2 copies `edr/uptycs-complete.tar.gz` (resolved symlink, produced by `scripts/openshift extract-uptycs`) and the three scripts into `coco-podvm-scripts/scripts/coco/podvm/` *after* the stale-tarball cleanup. Step 5e verifies binaries, cert, unit, enablement symlink, `/etc/osquery` symlink and `/UPTYCS_INSTALL_SUCCESS` in the finished QCOW2; Step 5c checks `/usr/bin/osqueryd` is `bin_t`.

**Why**: IBM requires Uptycs EDR on all peer pod VMs. Upstream has no hook for extra payloads.

**Revert condition**: never (IBM-specific). If upstream adds an extra-payload hook, move to that.
