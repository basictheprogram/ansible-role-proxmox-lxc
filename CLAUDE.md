# Claude Code project notes — proxmox_lxc

Creates and manages Proxmox VE LXC containers via the `community.proxmox`
collection (`community.proxmox.proxmox` for the container itself,
`community.proxmox.proxmox_template` to ensure the OS template is present
on the target node's storage before creation). Sets hostname, static
IP/gateway, cores, memory, swap, disk size, and storage as role variables
with reasonable defaults in `defaults/main.yml`, all overridable per-host
via `host_vars`. First use case driving this role's design: a Debian 13
self-hosted GitHub Actions runner LXC.

---

## Behavioral guidelines

These four rules govern how to work in this repo. They bias toward
caution over speed — for trivial one-liner changes, use judgment.

### 1. Think before writing tasks

**Don't assume. Surface tradeoffs. Ask when uncertain.**

Before adding or changing anything:

* State assumptions explicitly. If a variable could live in `defaults/`,
  `vars/`, or `host_vars`, say which and why before choosing.
* If multiple approaches exist (e.g. `ansible.builtin.command` vs a
  purpose-built module), present the tradeoff — don't pick silently.
* If the request is ambiguous (which task file? which template block?),
  name the ambiguity and ask. Don't guess and implement.
* If a simpler approach solves the problem, say so and push back.
* If something conflicts with `DESIGN.md`, flag it before proceeding.

### 2. Simplicity first

**Minimum tasks, variables, and template logic that solve the problem.**

* No new default variables beyond what the task being added requires.
* No Jinja2 abstraction for logic used in only one template.
* No `when:` conditions for scenarios that have no test coverage.
* No "future-proofing" of the public interface that wasn't asked for.
* If a template block is 30 lines and could be 10, rewrite it.

Ask: would a senior Ansible engineer call this overcomplicated? If yes,
simplify.

### 3. Surgical changes

**Touch only what the request requires. Clean up only your own mess.**

When editing existing tasks, templates, or defaults:

* Don't reformat adjacent YAML, fix unrelated comments, or clean up
  upstream code that wasn't broken by your change.
* Match the existing style — indentation, quoting, bullet character —
  even if you'd do it differently from scratch.
* If you notice unrelated dead code or stale variables, mention it;
  don't delete it without being asked.

When your change creates orphans:

* Remove `vars`, `when` conditions, or template blocks that YOUR change
  made unreachable.
* Don't remove pre-existing orphans unless explicitly asked.

Every changed line should trace directly to the request.

### 4. Goal-driven execution

**Define the success criteria before starting. Verify before declaring done.**

Transform requests into verifiable outcomes:

* "Add a preflight assertion" → `molecule converge` passes,
  `molecule verify` passes, `pre-commit run --all-files` is clean.
* "Fix an idempotency bug" → second `molecule converge` reports zero
  changed tasks.
* "Refactor a template" → rendered output is byte-for-byte identical
  to pre-refactor output on a converged instance.

For multi-step changes, state a brief plan before starting:

    1. Edit template → verify: rendered YAML is valid
    2. Add task       → verify: molecule converge green
    3. Add test       → verify: molecule verify green
    4. Lint           → verify: pre-commit run --all-files clean

Strong success criteria allow independent verification. Weak criteria
("make it work") require constant clarification.

---

## Role-specific notes

### Source of truth

`DESIGN.md` is the authoritative spec. Read it before any non-trivial
change. If code disagrees with `DESIGN.md`, `DESIGN.md` is right —
flag the discrepancy and ask before fixing the design to match the code.

### Design notes

No `DESIGN.md` found — add one. This role is brand new (scaffolded
2026-09-19); nothing has been implemented yet beyond this file and the
standard lint/CI config synced from `_template/`.

### Secrets

Role-specific secret variable names:

None yet — no `tasks/`/`defaults/` written yet. The role will need a
Proxmox API credential (API token strongly preferred over root
password) to authenticate to the Proxmox API; expect a var named
along the lines of `vault_proxmox_api_token_secret`, supplied by the
consumer via its own gitignored `vault.yml` (this repo's actual
secrets convention — see "Settled decisions" below), never hardcoded
here.

### Commit scopes

Role-specific subsystem scopes: `container` (create/update/delete the
LXC via `community.proxmox.proxmox`), `template` (OS template
presence via `community.proxmox.proxmox_template`), `network` (net0/
static IP handling). Anticipatory — no `tasks/` exist yet to infer
these from directly; revise once real task files land.

### Settled decisions

* **Collection: `community.proxmox`**, not `community.general.proxmox`
  — the latter's Proxmox modules were deprecated/removed from
  `community.general` and now live in this dedicated collection.
* **OS template: Debian 13** ("trixie") — current Debian stable as of
  2026-09-19; Debian 12 has moved to oldstable.
* **Both `community.proxmox.*` tasks in `tasks/main.yml` carry
  `delegate_to: localhost` and `become: false`** (added 2026-09-19
  after a real consumer failure). They're API calls, not guest shell
  commands — the guest doesn't exist yet when they run, so they must
  not try to connect to it. This is also what makes it correct for a
  consuming play to target the guest's own inventory hostname (so its
  `host_vars` actually load) rather than the Proxmox node. See
  "Consumer side notes" below and README's "Play Target Note."
* **A consuming play also needs `gather_facts: false`** — the guest
  doesn't exist yet, so Ansible's default fact-gathering SSH attempt
  fails before any task in this role even runs. Found the same session
  as the `delegate_to` fix; the first real consumer's playbook was
  missing it.
* **`proxmoxer`/`requests` must be importable by the control node's
  Python** (added 2026-09-19, second real-consumer failure the same
  session) — a `community.proxmox` runtime dependency that
  `ansible-galaxy collection install` does not pull in, and easy to
  miss precisely because it only matters for whichever Python actually
  executes the `delegate_to: localhost` tasks. Asserted explicitly in
  `tasks/preflight.yml` (a `command` + `register` + `assert`, since a
  plain `assert` can't probe for an importable module) rather than
  left to surface as a deep, `no_log`-obscured module import error.
* **`proxmox_api_validate_certs` defaults to `true`, not `false`**
  (added 2026-09-19, third real-consumer failure the same session —
  Proxmox VE's stock self-signed cert failed TLS validation). Secure
  by default is the right call for a generic role even though the
  first real consumer immediately needs to flip it — silently
  defaulting to `false` would weaken every future consumer's security
  posture without them asking for it. `proxmox_api_ca_path` exists as
  the more-correct alternative (validate against a real CA) for anyone
  who'd rather not disable validation outright.
* **`| default(omit)` does not do what it looks like it does for
  `proxmox_lxc_vmid`, `proxmox_lxc_root_pubkey`, and
  `proxmox_api_ca_path`** (bug found and fixed 2026-09-19, fourth
  real-consumer issue the same session). `default(omit)` only fires
  when a variable is *undefined* -- these all have a defined
  `defaults/main.yml` value of `""`, so the filter never triggered and
  the module always received a literal empty string when a consumer
  didn't override them. Fixed with `{{ var or omit }}` instead: `""`
  is falsy, so `or` correctly falls through to `omit`; any real value
  is truthy and passes through unchanged. Apply the same `or omit`
  pattern, not `default(omit)`, for any future optional variable whose
  role-default is an empty string rather than genuinely undefined.
* **`tasks/preflight.yml` resolves `proxmox_lxc_root_pubkey` via a
  `block`/`rescue`, not a plain `assert`** (added 2026-09-19, fifth
  real-consumer issue the same session — a `lookup('file', ...)`
  pointing at a key that didn't exist on the control node). A plain
  `assert` referencing that variable would re-trigger the same lookup
  failure at the exact same point in execution, just inside preflight
  instead of the create-container task -- no earlier, and with a
  worse message. `block`/`rescue` is what actually lets the failure be
  caught and turned into a clear, actionable message *before* the
  template-download and container-creation API calls run, rather than
  after burning a template-download call to reach a crash that then
  gets partially hidden behind `no_log: true` on the create task.
* **Hostname, static IP, and gateway are set via the `community.proxmox.proxmox`
  module's `hostname`/`net` parameters at container-creation time** —
  this is the LXC-native equivalent of cloud-init. They are deliberately
  **not** managed by a netplan-based role in the consuming site repo:
  Proxmox owns LXC network config and re-applies it into the guest's
  `/etc/network/interfaces` on every container start, which would fight
  a separately-managed netplan config, and the stock Debian Proxmox
  template doesn't ship netplan anyway.
* **A generic hostname/timezone/`/etc/hosts` bootstrap role from the
  consuming site repo can be reused as-is** against the resulting
  guest — nothing about that kind of role is inherently LXC-hostile.
  The one caveat found in practice: consumers must **not** set a
  machine-wide sysctl list for an LXC host, since unprivileged
  containers can't write most machine-wide sysctl keys (lacking
  `CAP_SYS_ADMIN` over the host namespace) — omit that variable for
  LXC hosts rather than patching the bootstrap role itself.
* **This role assumes the LXC does not need Docker running locally**
  for its first driving use case (a GitHub Actions runner that SSHes
  out to a separate existing host to run `docker compose` rather than
  running it in-container) — so no `nesting=1`/`keyctl=1` Proxmox
  feature flags or local Docker install are defaulted on. A consumer
  that *does* need Docker inside the LXC directly must set those
  feature flags itself at creation time via `proxmox_lxc_features` —
  don't assume they're irrelevant to this role in general, just to
  that first use case.
* **Secrets: no defaults for API credentials, and no assumption about
  vault mechanism.** Consumers supply `proxmox_api_token_secret` (etc.)
  however their own repo manages secrets — this role doesn't assume
  real `ansible-vault encrypt` vs. a plaintext gitignored `vault.yml`
  convention; that's entirely a consumer-side decision, deliberately
  not baked into this generic role.
* **Hardware sizing (cores, memory, swap, disk, storage) lives in
  `defaults/main.yml`** with reasonable defaults, relying on normal
  Ansible variable precedence (`host_vars` > role `defaults`) for
  per-host overrides — no custom override mechanism needed.

### Open questions

If a task touches one of these, leave a `# TODO(open-q):` comment:

* Proxmox API auth method and exact var names — API token (id +
  secret) is strongly preferred over root password, but not yet
  finalized.
* VMID assignment: fixed/hardcoded per host_vars entry, or looked up
  as "next free" via `community.proxmox.proxmox_vm_info`?
* Should `community.proxmox.proxmox_template` be run unconditionally
  (idempotent check-then-fetch) or should template presence be assumed
  pre-staged on the node?
* How does a system user like `gh-runner` get created on the guest —
  via `monolithprojects.github_actions_runner`'s own user-creation
  behavior, or via a consuming site repo's own user-management role
  with a `nologin` shell entry? Deliberately left as a consumer-side
  concern; this role currently assumes it does not need to care —
  container creation only, guest user management is out of scope.
* No real consumer has been wired up yet anywhere — this role has been
  scaffolded and reviewed, but never actually run against a live
  Proxmox node.

### Implementation order

Work one section at a time. Each item = one focused session and one
commit. Stop and verify between items.

1. `meta/main.yml` — `galaxy_info.namespace: realtime`,
   `galaxy_info.role_name: proxmox_lxc`, min_ansible_version 2.20,
   author/company/license matching this repo's other `realtime.*`
   roles (see `realtime.network_interface/meta/main.yml` for the
   pattern).
2. `defaults/main.yml` — hardware sizing vars (cores, memory, swap,
   disk size, storage pool, bridge), Proxmox API connection vars, LXC
   feature flags (default `nesting: false`), `unprivileged: true`.
3. `tasks/main.yml` — preflight assertions (required vars present),
   ensure OS template present (`community.proxmox.proxmox_template`),
   create/update the container (`community.proxmox.proxmox`,
   `state: present` for idempotency).
4. `README.md` — usage example, variable table, the netplan/
   `realtime.network_interface` incompatibility note from "Settled
   decisions" above (this is the thing a future maintainer is most
   likely to get wrong).
5. Molecule scenario — likely impractical to test LXC-creation-against-
   a-real-Proxmox-API in CI; flag as a known limitation rather than
   skipping silently. A lighter scenario (lint + syntax-check only, no
   real `molecule converge` against Proxmox) may be the realistic
   ceiling here — confirm with the user before assuming which.
6. Wire in a first real consumer: an inventory entry + `host_vars/`
   directory in whatever site repo needs the first real LXC, once its
   FQDN/IP are known, plus a playbook invoking this role and (if a
   CI runner is the use case) the `monolithprojects.github_actions_runner`
   install step downstream of it.

### Consumer side notes

**Corrected 2026-09-19 after a real failure**: the play must target the
**guest's own inventory hostname** (`hosts: <guest-fqdn>`), not the
Proxmox node. `host_vars` load based on the play's `hosts:` target —
pointing the play at the Proxmox node while this role's variables live
under `host_vars/<guest-fqdn>/` means none of them get loaded; every
`proxmox_api_*`/`proxmox_lxc_*` var silently falls back to this role's
own empty-string default, and preflight fails with a "must all be set"
message even though the file exists and looks correct. First real
consumer hit exactly this. Fixed by adding `delegate_to: localhost` +
`become: false` to both `community.proxmox.*` tasks in `tasks/main.yml`
— they're API calls, not guest shell commands, so the play can safely
target the guest (for correct `host_vars` resolution) while the actual
API calls run from the control node. See README's "Play Target Note"
for the consumer-facing explanation.

---

## Conventions

* **Commits**: follow the commit message guide in this file exactly.
  Conventional Commits, imperative mood, bodies wrapped at 72,
  asterisk bullets.
* **Lint**: `.ansible-lint`, `.yamllint`, `.pre-commit-config.yaml`
  define the rules. Run `pre-commit run --all-files` before declaring
  work done.
* **Secrets**: never write a credential into a tracked file. Vault
  secrets are consumed on the consumer side; the role templates them
  into config files with restricted permissions. Use `no_log: true`
  on any task that touches them.
* **Modules**: prefer FQCNs (`ansible.builtin.template`, etc.).
  The `.ansible-lint` rules require it.
* **Idempotency**: every task should be safe to re-run.

## Testing locally

* `pre-commit run --all-files` — fast lint/format pass. Run before
  every commit.
* `molecule converge` then `molecule verify` — fast iteration during
  template / task work; skips the destroy/create cycle.
* `molecule test` — full role exercise per platform. Slow; run
  before declaring a change done.

## When in doubt

Read `DESIGN.md`, then ask. The schemas and decisions there are
load-bearing.

---

## Commit message guide

You are an expert DevOps engineer and professional git commit message
writer. When generating a commit message, follow these steps exactly.

### Step 1 — Retrieve changes

Run:

    git diff --cached

Analyze the full staged diff. This is the **single source of truth**
for what will be committed.

### Step 2 — Understand the change

Determine:

* The **primary purpose** of the change
* The **type of change** (feature, bug fix, refactor, etc.)
* The **most relevant scope** within the role
* Whether the change introduces a **breaking change** for role consumers
* Whether multiple changes should be summarized together

Pay special attention to:

* Changes to `defaults/main.yml` — these define the role's public interface
* Changes to handler names, task names, and tags — consumers may pin to them
* Changes to template variables that consumers override
* Changes to config or env file templates that affect service behavior
* Changes to `meta/main.yml` — galaxy metadata, min Ansible version, platforms

If multiple files are modified, identify the **dominant intent** rather
than listing every file.

### Step 3 — Select commit type

Use Conventional Commits:

* `feat` — new task, handler, variable, template, or capability
* `fix` — bug fix or idempotency correction
* `docs` — README, role metadata documentation, inline comments
* `style` — YAML formatting, whitespace, ansible-lint cleanup
* `refactor` — restructure tasks/templates without behavior change
* `perf` — performance improvement (e.g., reduced task runs, fewer handlers)
* `test` — molecule scenarios, lint config, CI tests
* `chore` — galaxy metadata, dependencies, tooling
* `ci` — GitHub Actions, GitLab CI, pre-commit hooks

### Step 4 — Determine scope

Infer a scope from the role layout or the subsystem being changed.

Common Ansible role scopes: `tasks`, `handlers`, `templates`,
`defaults`, `vars`, `meta`, `molecule`, `docker`.

Role-specific subsystem scopes: `container`, `template`, `network`.

Only include a scope when it adds clarity. Prefer a subsystem scope
for feature-driven changes (e.g., `feat(tls): ...`) and a role-layout
scope for structural changes (e.g., `refactor(tasks): ...`).

### Step 5 — Write the commit message

Format exactly as:

    <type>[optional scope]: <short summary (<=50 chars)>

    <body wrapped at 72 characters>

    [optional footer(s)]

**Subject line rules:**

* Use **imperative mood** ("Add", "Fix", "Update", "Remove")
* Maximum **50 characters**
* Describe the **result**, not the implementation
* Prefer role-specific or Ansible terminology over generic phrasing

**Body rules** (required):

Explain **why the change was made**, focusing on:

* What deployment scenario or upstream behavior motivated it
* What downstream role consumers need to know to upgrade safely
* Any Ansible version constraints involved

When helpful, summarize key changes using bullet points.

**Bullet rules:**

* Use `*` (asterisk) for all bullets — never `-` or `•`
* Nested bullets indented with two spaces
* No Markdown formatting of any kind

**Ansible role expectations:**

* Call out new, renamed, or removed default variables
* Note when handler names, tag names, or public task names change
* Mention idempotency improvements when relevant
* Reference supported platforms when adding OS-specific tasks
* Flag changes to `meta/main.yml` (min Ansible version, platforms)
* Note molecule scenario additions or removals

### Breaking changes

A change is breaking when it:

* Renames or removes a default variable
* Renames or removes a handler, tag, or public task name
* Changes a default value in a way that alters runtime behavior
* Drops support for an Ansible version or OS platform
* Restructures generated configuration in a way consumers' overrides
  cannot accommodate

If the diff introduces a breaking change:

* Add `!` after the type/scope in the subject
* Include a footer: `BREAKING CHANGE: <description>`

Examples:

    feat(tasks): add preflight variable assertion block
    fix(handlers): correct service restart trigger condition
    refactor(tasks): split install and configure into files
    chore(meta): bump minimum Ansible version to 2.20
    test(molecule): add scenario for Ubuntu 24.04

    feat(defaults)!: rename primary configuration variable

    BREAKING CHANGE: old_variable_name is now new_variable_name;
    update playbook vars before upgrading.

### Step 6 — Output rules

Return **only the commit message**. Do NOT include:

* explanations or analysis
* the diff
* markdown formatting
* code fences

The output must be a clean commit message ready for `git commit`.
It will be pasted directly into a git commit editor — optimize for
copy/paste fidelity over styling.
