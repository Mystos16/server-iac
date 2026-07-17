# server-iac

Ansible setup for my personal home server (`kingkiller`, Debian 13), run via
[mise-en-place](https://mise.jdx.dev) over SSH.

Everything is layered into tiers: a **base** layer applied to every host, a
**storage** layer (ZFS), and a **platform** layer (k3s, HAProxy). Roles are
applied to a host based on which inventory groups it belongs to.

---

## Do I need Ansible installed on my laptop?

**Yes.** Ansible runs *from* your laptop (the control node) and connects to the
server over SSH. The server itself needs nothing pre-installed except an SSH
daemon and Python (Debian has both).

You don't install it by hand though — `mise` does it for you:

```bash
mise run setup     # installs Ansible + the required Galaxy collections
```

This uses the pinned Python from `.mise.toml` and installs Ansible plus the
`community.general` and `ansible.posix` collections into that environment. After
that, all the other `mise run ...` tasks work.

---

## Repository layout

```
server-iac/
├── .mise.toml               # Task runner. Defines `mise run <task>` commands.
├── ansible.cfg              # Ansible config: inventory path, SSH options, roles path.
├── requirements.yml         # Galaxy collections to install (community.general, ansible.posix).
├── .gitignore
│
├── inventory/
│   ├── hosts.yml            # WHICH hosts exist and WHICH groups they belong to.
│   └── group_vars/
│       ├── all.yml          # Variables for every host (admin user, packages, tailscale, hardening).
│       ├── zfs.yml          # ZFS pool/dataset definition (the tank pool).
│       ├── k3s.yml          # k3s install options.
│       └── haproxy.yml      # HAProxy frontends/backends/stats.
│
├── host_vars/               # Per-host overrides (empty for now; single host).
│
├── playbooks/
│   ├── site.yml             # Orchestrator: runs all tiers in order.
│   ├── base.yml             # Tier 0 — base role on all hosts.
│   ├── zfs.yml              # Tier 1 — zfs role.
│   ├── k3s.yml              # Tier 2 — k3s role.
│   ├── haproxy.yml          # Tier 2 — haproxy role.
│   └── upgrade.yml          # On-demand full apt dist-upgrade (+ reboot if needed).
│
└── roles/
    ├── base/                # Tier 0 role (see below).
    ├── zfs/                 # Tier 1 role.
    ├── k3s/                 # Tier 2 role.
    └── haproxy/             # Tier 2 role.
```

### How it fits together

1. `inventory/hosts.yml` lists the host `kingkiller` and puts it in the `zfs`
   and `k3s` groups. Every host is implicitly in `all`.
2. `group_vars/*.yml` supply variables — `all.yml` to everyone, the others only
   to hosts in that group.
3. `playbooks/*.yml` map a group of hosts to a role.
4. `roles/*` contain the actual tasks.
5. `.mise.toml` wraps `ansible-playbook` calls so you run short commands.

---

## Commands

```bash
mise run setup      # one-time: install Ansible + collections
mise run ping       # test SSH connectivity to all hosts
mise run site       # run everything (base -> zfs -> k3s -> haproxy)
mise run base       # just the base layer
mise run zfs        # just the storage layer
mise run k3s        # just k3s
mise run haproxy    # just haproxy
mise run upgrade    # full apt dist-upgrade on all hosts (reboots if required)
```

---

## Tiers / layers explained

### Tier 0 — `base` (every host)

The foundation applied to all machines. Task files live in
`roles/base/tasks/`:

| File               | What it does |
|--------------------|--------------|
| `packages.yml`     | Installs the base package set (curl, git, htop, tmux, jq, rsync, etc. — defined in `group_vars/all.yml`). |
| `user.yml`         | Ensures my SSH public key is authorized for my existing user `gijs`. Does not create or modify users. |
| `mise.yml`         | Installs mise-en-place system-wide from its official apt repo. |
| `tailscale.yml`    | Installs Tailscale from its apt repo and runs `tailscale up` only if not already connected. Auth key comes from the `TS_AUTHKEY` env var. |
| `hardening.yml`    | Drops an sshd hardening config: root login disabled, key auth on, `MaxAuthTries` lowered. Password auth is left enabled (commented switch to disable). |
| `firewall.yml`     | Installs and enables `ufw`: default deny inbound, allow SSH and the `tailscale0` interface. |
| `fail2ban.yml`     | Installs fail2ban and templates `jail.local` (bans SSH brute-force; ignores localhost and the Tailscale CGNAT range). |
| `unattended.yml`   | Installs and enables `unattended-upgrades` for automatic security patches. |

Handlers (`roles/base/handlers/main.yml`) restart `ssh` and `fail2ban` when
their configs change.

### Tier 1 — `zfs` (storage)

`roles/zfs/`. Runs before the platform layer so storage is ready first.

- `tasks/main.yml`: installs kernel headers + `zfs-dkms` + `zfsutils-linux`
  (enabling the Debian `contrib` component), loads the `zfs` module, and makes
  it load at boot.
- `tasks/pool.yml`: creates each pool defined in `group_vars/zfs.yml` if it
  doesn't already exist, attaches cache (L2ARC) devices, and creates datasets.

The pool for this host is **`tank`**: raidz1 across the four 3 TB HDDs, with the
60 GB Intel SSD as L2ARC read cache. The 120 GB Samsung SSD (boot/OS disk) is
deliberately excluded. ARC is left at ZFS defaults (grows to ~50% RAM, reclaimed
under memory pressure).

### Tier 2 — `k3s` (platform)

`roles/k3s/`. Single-node k3s using the official install script (default
config). Installs only if `/usr/local/bin/k3s` is absent, ensures the service is
running, and waits for the node to report `Ready`.

### Tier 2 — `haproxy` (platform, separate machine)

`roles/haproxy/`. Present for a future dedicated load-balancer host — the
`haproxy` inventory group is currently empty. Installs HAProxy, templates
`haproxy.cfg` from frontends/backends/stats defined in `group_vars/haproxy.yml`,
and validates the config before reloading.

---

## Secrets

Pulled from environment variables (no vault, no external secret store):

| Variable                   | Used by | Purpose                        |
|----------------------------|---------|--------------------------------|
| `TS_AUTHKEY`               | base    | Tailscale auth key             |
| `HAPROXY_STATS_PASSWORD`   | haproxy | HAProxy stats page password    |

```bash
export TS_AUTHKEY="tskey-auth-xxxx"
mise run base
```

Rotate/revoke Tailscale auth keys in the admin console once used.

---

## First-run checklist

1. `mise run setup`
2. Ensure SSH works: `ssh kingkiller` (config uses user `gijs` + your key).
3. `export TS_AUTHKEY=...` (a fresh key).
4. Confirm the 4 HDDs + Intel SSD are empty — `zpool create -f` overwrites them.
5. `mise run ping` then `mise run site`.
