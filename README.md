# server-iac

Ansible setup for personal servers, run via [mise-en-place](https://mise.jdx.dev)
over SSH. Debian 13.

## Layout / tiers

- **Tier 0 — base**: applied to every host (tailscale, mise, git, hardening,
  ufw, fail2ban, unattended-upgrades, admin user, base packages).
- **Tier 1 — zfs**: storage, runs before platform roles that may need it.
- **Tier 2 — platform**: `k3s` (single node) and `haproxy` (separate machine).

`playbooks/site.yml` runs the tiers in dependency order. Each role also has a
standalone playbook.

## Usage

```bash
# one-time: install ansible + collections
mise run setup

# fill in your hosts and secrets first (see below), then:
mise run ping        # connectivity check
mise run site        # everything
mise run base        # just the base role
mise run zfs
mise run k3s
mise run haproxy
mise run upgrade     # full apt dist-upgrade on all hosts
```

## Defining hosts

Edit `inventory/hosts.yml`. Add a host once under `all.hosts`, then tag it into
the groups whose roles you want applied (`zfs`, `k3s`, `haproxy`). Everything in
`all` gets the base role.

## Secrets (from env)

Secrets are pulled from environment variables via `lookup('env', ...)`:

| Variable      | Used by   | Purpose                     |
|---------------|-----------|-----------------------------|
| `TS_AUTHKEY`  | base      | Tailscale auth key          |

```bash
export TS_AUTHKEY="tskey-auth-xxxxxxxx"
mise run base
```

Rotate/revoke auth keys in the Tailscale admin console when done.

## Admin user

`inventory/group_vars/all.yml` defines a dummy `admin_user` +
`admin_ssh_key`. Fill these in before running the base role.
