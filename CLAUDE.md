# CLAUDE.md - Custom LXC Container Host System (t2d1 / future lxc0+)

## Project purpose
Custom hardened LXC container host design for GCP VMs. Test box: `t2d1`
(t2d-standard-1, us-central1-f, project `development-237419`, Debian 13/trixie,
systemd-native, root SSH). Production will be `lxc0`, `lxc1`, ... (FQDN
`lxc0.dev.lan` / `lxc0.prod.lan`) on C4D/N4D + Hyperdisk Balanced, hosting
~28 MQTT broker CTs (consolidating existing 28-VM fleet), MariaDB, OMS CTs.

Working dir: `/opt/cttools` (staging). Deploy targets:
- `ct-create`, `ct-destroy` -> `/usr/local/sbin/`
- `defaults.conf`, `mqtt.conf`, `ctN.service.template` -> `/etc/ct-create/`
- hook scripts -> `/etc/ct-create/scripts/`

## Architecture (all validated working on t2d1)

### Networking - ipvlan L3s
- VPC `default`, subnet `default` us-central1 (10.128.0.0/20), secondary range
  `containers` = 10.130.0.0/24 (pool for ALL lxc hosts; each host claims a
  non-overlapping alias slice on its nic0)
- t2d1 nic0 `ens4` (10.128.0.33) has alias `containers:10.130.0.0/28`
- CT networking: `lxc.net.0.type = ipvlan`, `ipvlan.mode = l3s`, `link = ens4`.
  ZERO host-side per-CT network config (no veth, no bridges, no networkd files)
- Addressing rule: **ctN = 10.130.0.N** (ct1=.1 ... ct15=.15; /28 cap; .1 is
  usable - no gateway IP exists in ipvlan L3)
- In-CT eth0.network: Address=10.130.0.N/32, DNS=169.254.169.254, default
  route `Destination=0.0.0.0/0 Scope=link` (device route, no gateway), MTU
  inherits 1460 from ens4
- Cloud NAT covers secondary range -> CT egress works
- L3s = CT traffic traverses host netfilter in **input/output hooks, NOT
  forward** (verified by counters). Host nftables = enforcement point; CT root
  cannot affect it. Rules must match on CT addrs (saddr/daddr 10.130.0.0/24)
  since input/output also carry host traffic
- Host<->CT networking does NOT work (ipvlan same-interface limitation, by
  design). Management via lxc-attach only
- Known ipvlan L3s quirks: interface renamed ipvl* -> eth0 in kernel log each
  start (noise); TOMOYO log line (noise)
- External exposure: nft DNAT works (e.g. `ip daddr 35.225.59.53 dnat to
  10.130.0.2` in inet nat prerouting) via GCP protocol forwarding/target
  instances. Persisted in /etc/nftables.conf, nftables.service enabled
- Production LB plan: global external TCP proxy LB (keeps anycast IP) ->
  zonal NEGs type GCE_VM_IP_PORT with **CT alias IPs as endpoints** (container-
  native LB). Health checks per-CT (allow 35.191.0.0/16, 130.211.0.0/22).
  PROXY protocol if client IPs needed. Passthrough NLB rejected: no alias-IP
  endpoints, regional IP (loses anycast), DSR needs host DNAT anyway.
  Active/active across 2 hosts via both hosts' endpoints in one backend service

### Containers - unprivileged, idmapped, btrfs
- Rootfs: debootstrap trixie minbase (or btrfs template snapshot). Dir-type
  rootfs at /var/lib/lxc/ctN/rootfs, each ctN is a btrfs subvolume
- /var/lib/lxc is btrfs (dedicated disk), REQUIRED (ct-create hard-fails else)
- idmap: `lxc.idmap = u/g 0 BASE 65536`, BASE = 231072 + (N-1)*65536.
  /etc/subuid+subgid: `root:231072:1048576` (covers ct1-ct16; ct-create
  auto-extends). Other entries exist (ctadmin:100000, flume:165536) - leave
- `lxc.rootfs.options = idmap=container` -> on-disk files owned by REAL root,
  shift at mount time. One template works for any idmap range
- `lxc.mount.auto = proc:mixed sys:ro cgroup:mixed` - sys:ro REQUIRED
  (networkd hangs on udev wait otherwise). No AppArmor userspace on host ->
  `lxc.apparmor.profile = unconfined`. Includes: common.conf + userns.conf
  (debian.* configs don't exist on Debian 13)
- CT contents: systemd-native (networkd/resolved/timesyncd enabled), no udev,
  no sshd, no dbus by default (resolved pulls dbus in -> logind runs; decided
  to leave alone). NTP=169.254.169.254 (GCE metadata NTP). TZ default
  America/Los_Angeles. LANG=C.UTF-8. root password deleted (attach-only)
- **Reboot handling**: in-CT reboot.target symlinked to poweroff.target +
  unit Restart=always RestartSec=2. Reason: lxc-start -F internal reboot
  breaks idmapped rootfs mount (perms show nobody/nogroup). In-CT `reboot` =
  clean stop -> systemd restarts fresh. Host-side restart always safe
- Units: `/etc/systemd/system/ctN.service` (convention: ctN.service, NOT
  lxc-ctN), one file per CT sed-generated from template. Delegate=yes,
  MemoryMax/MemoryHigh (hard cap/soft throttle), CPUQuota (hard ceiling, per
  one core), optional CPUWeight (proportional share under contention).
  Template units (ct@.service) considered and rejected

### SSH console access (no sshd in CTs)
- `ssh ctN@t2d1` -> root shell in ctN. Mechanism:
  - Host users ct1..ctN + ctadmin ALL share uid/gid 1005 (useradd -o), no
    home (-d /), shell /bin/bash (real shell REQUIRED for ForceCommand)
  - sshd_config: `Match User ct*,!ctadmin` -> AuthorizedKeysFile
    /etc/ssh/ct_authorized_keys (shared single key, root:root 644),
    ForceCommand `sudo /usr/bin/lxc-attach -n "$USER" --clear-env --keep-var
    TERM -- /bin/bash --login`, DisableForwarding yes, PermitTTY yes
  - sshd patterns: only * and ? work, NO [0-9] classes (that bug was hit)
  - sudoers /etc/sudoers.d/ctadmin: `%ctadmin ALL=(root) NOPASSWD:
    /usr/bin/lxc-attach -n ct[0-9]* --clear-env --keep-var TERM -- /bin/bash
    --login` (glob IS the security boundary; $USER set by sshd)
  - DenyUsers ctadmin (main sshd_config) - ctadmin not SSH-able
  - /root/.bash_profile in CT: `cd /root` + source .bashrc (login shells skip
    .bashrc otherwise)
- lxc-attach -n ctN from host works always (network-independent)

### DNS / aliases
- Both DNS zones live in dev project: zone name `dev-lan` (domain dev.lan),
  `prod-lan` (prod.lan)
- A record per CT: `ctN.<hostshort>.<domain>` -> 10.130.0.N (upsert via
  gcloud --configuration=$CT_GCLOUD_CONFIG, warn-don't-die on failure)
- Instance alias (--alias / CT_ALIAS): CNAME `<alias>.<domain>` -> CT FQDN,
  e.g. mqtt3.dev.lan -> ct2.t2d1.dev.lan. Also: /etc/hosts line, PS1 via
  /etc/profile.d/ct-alias.sh (root@ct2[mqtt3]:...), stamp file
  /var/lib/lxc/ctN/alias. ct-destroy removes CNAME (keeps A), rm stamp
- Config-vs-alias semantics: --config = role/profile (packages etc., e.g.
  mqtt.conf), --alias = instance name (mqtt3)

## Scripts (in /opt/cttools)

### ct-create [--name ctN] [--config <name>] [--alias <alias>] [--dry-run|-n]
Auto-picks lowest free N (by /var/lib/lxc/ctN existence), IP=10.130.0.N.
Config layering: built-in defaults -> /etc/ct-create/defaults.conf ->
/etc/ct-create/<name>.conf (--config) -> CLI --alias wins over CT_ALIAS.
Validates: root, btrfs, >=10% free, name/unit conflict, hooks executable,
template exists. Auto-extends subuid/subgid. (Step-by-step flow: read the
deployed script.)
Hooks: arrays, entries "script.sh [args...]", relative to
CT_HOOK_SCRIPTS_DIR=/etc/ct-create/scripts (host), run inside CT with args
(eval-parsed, quoted args ok). Template-based creates SKIP fresh-build steps
(templates pre-baked).

### ct-destroy ctN --yes [--force] [--purge] [--dry-run|-n]
Refuses if running w/o --force. Disables unit, deletes CNAME via alias stamp,
btrfs subvolume delete (or rm -rf legacy), removes config+alias. --purge also
removes unit file + host user. Default keeps unit/user for reuse.

### Config files (see /etc/ct-create/ for current contents)
- Default packages: systemd,systemd-sysv,systemd-timesyncd,iputils-ping,nano,
  net-tools,less,lrzsz,rsync,procps,wget,curl,locales,tzdata,iproute2,
  bash-completion (+ systemd-resolved post-installed in chroot - debootstrap
  --include of resolved FAILS in chroot, known issue, hence two-step)

## Host setup already done on t2d1 (replicate on prod hosts)
- ip_forward=1, rp_filter=2; lxc installed --no-install-recommends; lxc-net
  disabled+masked; /etc/systemd/system-generators/systemd-ssh-generator
  masked (vsock error noise on GCE, systemd 256+)
- nftables.conf persisted + service enabled (currently: inet nat prerouting
  DNAT 35.225.59.53 -> 10.130.0.2)
- subuid/subgid root:231072:1048576; ssh users; sudoers; sshd Match block
- btrfs disk mounted /var/lib/lxc (fstab; compress=zstd:1 noatime recommended)
- gcloud installed, config 'dev', SA gce-sa@development-237419 w/
  roles/monitoring.editor added

## Open items / next steps
1. **ct-snap** script + systemd timer: btrfs subvolume snapshot -r into
   /var/lib/lxc/.snapshots/<ct>-<stamp>, retention prune. Recovery: stop CT,
   mv rootfs aside, rw-snapshot from snapshot, start. Cadence TBD
2. **ct-template** script: snapshot existing CT rootfs -> .templates/<name>
   (rw) -> offline sysprep (truncate machine-id, rm eth0.network/hosts line/
   ct-alias.sh/logs/apt cache/history; KEEP bash_profile/reboot symlink/
   enables) -> btrfs property set ro true. Templates independent of source CT
   (CoW; source deletable). Never boot templates
3. Off-host backup: btrfs send/receive incremental + PD snapshots (DR layer)
4. Security punch list (deferred): nftables egress allowlist per CT (input/
   output hooks!), lxc.cap.drop, seccomp verification, read-only rootfs,
   conntrack sizing (nf_conntrack_max ~1M for 300k conns)
5. Monitoring: dashboard "MQTT VM Sizing v3" deployed in prod project
   (mqtt-dashboard.json; agent metrics keyed on metadata.system_labels.name;
   gauge charts use ALIGN_MAX - percentile aligners invalid for plain gauges)
6. Prod sizing (from 30d dashboard data, 28 VMs, ~110k devices, ~106k conns):
   fleet CPU ~2.2-2.9 cores, mem ~40GiB peak (leaky daemons sawtooth 2-3GiB
   then restart; post-restart baseline ~25-28GiB), ~5MiB/s, ~22k pps, ~20-80
   IOPS. Projection 300k devices: ~8 cores, ~110GiB (leak) / ~70-75GiB
   (fixed), ~290k conns. Memory-bound. Candidates: 2x c4d-highmem-8 (63GB)
   active/active post-leak-fix, or c4d-highmem-16 for pre-fix margin; N4D
   custom shape (e.g. 8vCPU/64GB) cheaper alternative - CPU perf not needed.
   One daemon has known unfixed CPU issue + memory leak; source to be shared
   with Claude for fixing
7. gce-create script exists (separate, from earlier sessions) for VM
   provisioning; ct-create modeled on its defaults+named-config pattern

## Conventions / gotchas learned
- nftables: 'fwd' is a reserved keyword (chain name ctfwd); [0-9] globs work
  in sudoers but NOT sshd Match
- debootstrap systemd-resolved --include fails; install in chroot after
- lxc-device exists for hotplug but nothing to pass through on GCE
- systemd-socket-proxyd pattern dead with ipvlan (no host<->CT traffic)
- Snapshot disk math: cost = churn (overwritten/deleted), not additions
- Hyperdisk: C4D/N4D require Hyperdisk (no pd-*); Balanced ~$0.09/GiB +
  3k IOPS/140MiB/s baseline free; live-resizable (disk + btrfs resize max,
  grow-only); t2d does NOT support Hyperdisk
- User (Diane) prefers: brief/concise/direct responses, numbered Q&A for
  decisions, discuss-before-build, dry-run flags on tools
