# realtime.proxmox_lxc
Ansible role for creating and managing Proxmox VE LXC containers.

---

<!-- ANSIBLE DOCSMITH MAIN START -->
# Ansible Role realtime.proxmox_lxc

Creates and manages Proxmox VE LXC containers via the `community.proxmox`
collection: ensures the OS container template is present on the target
node's storage, then creates/updates the container (hostname, static
IPv4/gateway, hardware sizing) idempotently.

## Table of Contents <a id="toc"></a>

- [Requirements](#requirements)
- [Supported Platforms](#platforms)
- [Role Variables](#variables)
- [Networking Note](#networking)
- [Task Flow](#taskflow)
- [Limitations](#limitations)
- [Dependencies](#dependencies)
- [Example Playbook](#example)
- [License](#license)
- [Author Information](#author)

---

## Requirements <a id="requirements"></a>

- Ansible core >= 2.20
- The `community.proxmox` collection (see [Dependencies](#dependencies))
- A Proxmox VE API token with permission to create/update containers on
  the target node

---

## Supported Platforms <a id="platforms"></a>

This role runs its tasks against the Proxmox API — its "platform" is the
**guest container template** being provisioned, not an OS family the role
itself branches on (Proxmox nodes are uniformly Debian-based, so there is
no `vars/<OsFamily>.yml` split here; see [Role Variables](#variables)).

Guest templates this role has actually been used with:

| Distribution | Versions |
|-------------|---------|
| Debian      | 13 (trixie) |
| Ubuntu      | 24.04 (noble) — untested, but the module interface is identical |

Debian 12 (bookworm) is EOL as of 2026-09 and was never onboarded here.

---

## Role Variables <a id="variables"></a>

### defaults/main.yml

User-overridable defaults. One file; no OS-specific variants (see
[Supported Platforms](#platforms) for why).

| Variable | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `proxmox_lxc_cores` | `int` | No | `2` | CPU cores |
| `proxmox_lxc_memory` | `int` | No | `2048` | Memory in MB |
| `proxmox_lxc_swap` | `int` | No | `512` | Swap in MB |
| `proxmox_lxc_disk_gb` | `int` | No | `20` | Root disk size in GiB |
| `proxmox_lxc_storage` | `str` | No | `local-lvm` | Storage pool for the root disk |
| `proxmox_lxc_unprivileged` | `bool` | No | `true` | Create as an unprivileged container |
| `proxmox_lxc_onboot` | `bool` | No | `true` | Start automatically when the node boots |
| `proxmox_lxc_state` | `str` | No | `started` | `community.proxmox.proxmox` state — use `present` to create without starting |
| `proxmox_lxc_features` | `list` | No | `[]` | e.g. `["nesting=1"]` — only needed if this LXC runs Docker or another nested-container workload directly (see [Networking Note](#networking) for why the first consumer doesn't need it) |
| `proxmox_lxc_bridge` | `str` | No | `vmbr0` | Proxmox network bridge |
| `proxmox_lxc_net_interface_name` | `str` | No | `eth0` | Interface name inside the guest |
| `proxmox_lxc_hostname` | `str` | **Yes** | `""` | Container hostname |
| `proxmox_lxc_ip_cidr` | `str` | **Yes** | `""` | Static IPv4 + prefix, e.g. `192.0.2.10/24` |
| `proxmox_lxc_gateway` | `str` | **Yes** | `""` | IPv4 gateway |
| `proxmox_lxc_ostemplate` | `str` | **Yes** | `""` | Exact template filename, e.g. `debian-13-standard_13.0-1_amd64.tar.zst` — check `pveam available` on the target node first; this role does not default one since Proxmox revises these periodically |
| `proxmox_lxc_ostemplate_storage` | `str` | No | `local` | Storage holding the OS template |
| `proxmox_lxc_node` | `str` | **Yes** | `""` | Proxmox node to provision on |
| `proxmox_lxc_vmid` | `str`/`int` | No | `""` | Leave empty to let Proxmox auto-assign |
| `proxmox_lxc_root_pubkey` | `str` | No | `""` | SSH public key for `/root/.ssh/authorized_keys` at creation |
| `proxmox_api_host` | `str` | **Yes** | `""` | Proxmox API host |
| `proxmox_api_user` | `str` | **Yes** | `""` | Proxmox API user (e.g. `ansible@pve`) |
| `proxmox_api_token_id` | `str` | **Yes** | `""` | API token ID |
| `proxmox_api_token_secret` | `str` | **Yes** | `""` | API token secret — supply via the consumer's own gitignored `vault.yml` (e.g. `vault_proxmox_api_token_secret`), never hardcoded |

Every variable marked **Yes** is asserted non-empty in `tasks/preflight.yml`
and fails fast with a named-variable error message rather than a raw
module-level failure.

### vars/

Not used by this role — there is no OS-family branching to drive it (see
[Supported Platforms](#platforms)).

---

## Networking Note <a id="networking"></a>

**This role does not use `realtime.network_interface` (netplan), and a
consuming playbook shouldn't either for the guest this role creates.**
Proxmox owns LXC network configuration directly: the `hostname` and
`netif` parameters passed to `community.proxmox.proxmox` are the LXC-native
equivalent of cloud-init, and Proxmox re-applies them into the guest's
`/etc/network/interfaces` **on every container start** — a separately
managed netplan config would either target a file nothing reads (the
stock Debian template doesn't ship netplan) or get fought by Proxmox on
the next restart. Static IP/gateway/hostname belong in this role's own
variables, applied once at the Proxmox API layer, not inside the guest.

`realtime.bootstrap` (hostname confirmation, timezone, `/etc/hosts`) is
still fine to run against the resulting guest — just don't set
`sysctl_conf` for it, since unprivileged containers can't write most
machine-wide sysctl keys.

---

## Task Flow <a id="taskflow"></a>

1. **Preflight** (`tasks/preflight.yml`) — asserts Ansible version, API
   connection vars, node/hostname, an OS template filename, IPv4
   CIDR/gateway shape, and that `proxmox_lxc_features` is a list.
2. **Ensure template present** — `community.proxmox.proxmox_template`
   downloads the named template from the Proxmox appliance catalog onto
   the node's storage if it isn't already there.
3. **Create or update the container** — `community.proxmox.proxmox` with
   `update: true`, so re-running this role against an existing container
   converges its config rather than failing on "already exists."

---

## Limitations <a id="limitations"></a>

- IPv4 static addressing only — no DHCP, no IPv6.
- No molecule coverage yet: exercising this role means actually calling a
  real Proxmox API, which doesn't fit the Docker-container molecule
  pattern this repo's other roles use. Flagged in `TODO.md` rather than
  faked with a scenario that doesn't test anything real.
- Does not create the `gh-runner`-style guest user this role's first
  consumer needs — that's a separate, consumer-side concern (see
  `CLAUDE.md` open questions).

---

## Dependencies <a id="dependencies"></a>

- `community.proxmox` collection — see `requirements.yml`.

---

## Example Playbook <a id="example"></a>

```yaml
- name: Provision a self-hosted GitHub Actions runner LXC
  hosts: pve-node-01.example.com
  become: false # the community.proxmox modules talk to the API, not the node's shell
  vars:
    proxmox_api_host: "pve-node-01.example.com"
    proxmox_api_user: "ansible@pve"
    proxmox_api_token_id: "ansible"
    proxmox_api_token_secret: "{{ vault_proxmox_api_token_secret }}"
    proxmox_lxc_node: "pve-node-01"
    proxmox_lxc_hostname: "gh-runner-example-app"
    proxmox_lxc_ip_cidr: "192.0.2.50/24"
    proxmox_lxc_gateway: "192.0.2.1"
    proxmox_lxc_ostemplate: "debian-13-standard_13.0-1_amd64.tar.zst"
    proxmox_lxc_root_pubkey: "{{ lookup('file', '~/.ssh/id_ed25519.pub') }}"
  roles:
    - role: realtime.proxmox_lxc
```

---

# License <a id="license"></a>

MIT

# Author Information <a id="author"></a>

Bob Tanner, Real Time Enterprises, Inc.
