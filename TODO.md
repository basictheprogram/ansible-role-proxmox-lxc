# TODO — realtime.proxmox_lxc

Flagged-but-unresolved items from the initial scaffolding session
(2026-09-19). Not silently dropped — see `CLAUDE.md`'s "Open questions"
for the reasoning behind each.

- [x] **Git remote configured** (2026-09-19):
      `git@github.com:basictheprogram/ansible-role-proxmox-lxc.git`
      (public), matching every other `realtime.*` role's convention.
      `origin` is set locally and `meta/main.yml`'s `issue_tracker_url`
      points at it. **Not yet pushed** — the remote is a fresh, empty
      repo (confirmed via `git ls-remote`, zero refs); nothing has been
      published to it yet. Top-level `requirements.yml` now has a real
      `src`/`scm`/`version: ansible-core-2.20` entry for it, matching
      this repo's branch-per-ansible-core-version convention — but that
      branch doesn't exist on the remote until the first push creates
      it.
- [ ] **Molecule testing skipped entirely** (explicit user decision,
      2026-09-19) — this role's actual work calls the live Proxmox API,
      which doesn't fit the Docker-container molecule pattern this repo's
      other roles use. No scenario exists, not even a lint-only one.
      Revisit if this role grows enough consumers to justify mocking the
      API, or if a real Proxmox test node/cluster becomes available for
      CI.
- [ ] **`monolithprojects.github_actions_runner` has no version pin yet**
      in the top-level `requirements.yml` — added unpinned since I
      couldn't confirm an exact current Galaxy release version. Pin one
      before relying on this in a real deploy.
- [ ] **Proxmox API token itself doesn't exist yet** — needs creating in
      the Proxmox UI (Datacenter → Permissions → API Tokens), scoped to
      just guest-create/update permissions on the target node, before
      `proxmox_api_token_id`/`proxmox_api_token_secret` can be populated
      in any consumer's `vault.yml`.
- [ ] **VMID assignment scheme undecided** — `proxmox_lxc_vmid` currently
      defaults to empty (Proxmox auto-assigns). Fine for a single one-off
      host; revisit if multiple LXCs get provisioned by this role and a
      predictable/looked-up VMID scheme becomes useful.
- [ ] **Guest user creation is explicitly out of scope for this role** —
      it only creates the container. Whether a system user like `gh-runner`
      on the guest gets created by `monolithprojects.github_actions_runner`
      itself or by the consuming site repo's own user-management role
      (with a `nologin` shell entry) is a consumer-side decision; affects
      a downstream playbook, not this role.
- [ ] **Not wired to any real consumer yet.** No inventory entry or
      `host_vars/` directory exists anywhere for a real first consumer —
      this role has been scaffolded and reviewed, but never actually run
      against a live Proxmox node.
