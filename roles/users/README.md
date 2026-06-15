# Role: users

Manages OS groups, user accounts, SSH authorized keys, and sudoers
drop-ins for the `rhel-oe-build-lab` fleet.

## What This Role Does

| Task | File modified |
|------|---------------|
| Creates `admin` and `operations` groups | `/etc/group` |
| Creates users from `users_list` variable | `/etc/passwd` + home dir |
| Deploys SSH public keys per user | `~/.ssh/authorized_keys` |
| Grants passwordless sudo to `admin` group | `/etc/sudoers.d/ops-admin` |
| Creates (and optionally locks) a break-glass account | `/etc/passwd` |

## Variables

All variables live in `defaults/main.yml` and can be overridden in
`group_vars/` or `host_vars/`.

| Variable | Default | Description |
|----------|---------|-------------|
| `users_groups` | `[admin, operations]` | Groups to create |
| `users_list` | `[]` | Users to create (see structure below) |
| `users_sudo_nopasswd_groups` | `[admin]` | Groups granted NOPASSWD sudo |
| `users_breakglass_enabled` | `false` | Unlock the break-glass account |
| `users_breakglass_name` | `breakglass` | Break-glass account login name |
| `users_breakglass_groups` | `[wheel, admin]` | Break-glass group membership |

### users_list entry structure

```yaml
users_list:
  - name: jsmith                         # required
    comment: "Jane Smith"                # GECOS
    groups: [operations]                 # supplementary groups
    shell: /bin/bash                     # default /bin/bash
    ssh_authorized_keys:                 # list of public key strings
      - "ssh-ed25519 AAAA... jsmith@ws"
    state: present                       # present | absent
```

## Relationship to baseline_hardening

`baseline_hardening` deploys a `wheel` sudoers drop-in
(`/etc/sudoers.d/wheel-nopass`) as part of its baseline posture.
This role adds a **separate** drop-in (`/etc/sudoers.d/ops-admin`)
for the `admin` group. Both can coexist safely.

The target state (CRQ-002) is to:
1. Add an `admin` user via this role with an SSH key.
2. Add that user to `wheel` and `admin`.
3. Disable root SSH in `baseline_hardening` defaults.

## Break-glass Account

The break-glass account (`breakglass`) is created in a locked state
on every host. It is unlocked only by setting
`users_breakglass_enabled: true` — which should itself be tracked as
a change record entry. Revert to `false` and re-apply to re-lock after
the emergency is resolved.

## Tags

| Tag | Tasks covered |
|-----|--------------|
| `users` | All tasks |
| `groups` | Group creation only |
| `ssh_keys` | SSH authorized key management |
| `sudo` | Sudoers drop-in deployment |
| `breakglass` | Break-glass account management |
