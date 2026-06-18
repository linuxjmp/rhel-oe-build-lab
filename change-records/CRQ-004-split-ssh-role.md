# CRQ-004: Split SSH Posture into a Dedicated Role

| Field           | Value                                                              |
|-----------------|--------------------------------------------------------------------|
| Change ID       | CRQ-004                                                            |
| Title           | Extract sshd posture from baseline_hardening into an `ssh` role    |
| Owner           | jaric.poinsette                                                   |
| Date submitted  | 2026-06-17                                                        |
| Risk level      | Medium (touches remote-access policy)                            |
| Status          | Implemented — pending fleet validation                            |

---

## 1. Reason for Change

SSH access policy is the single control most likely to lock an
operator out of a host, and it was buried inside `baseline_hardening`
alongside packages, SELinux, firewalld, auditd and sudo. Splitting it
into its own role:

- lets SSH posture be read, changed, and `--tags ssh` applied on its
  own without re-running the whole baseline,
- makes the dependency on the `users` role explicit and ordered: the
  admin account (CRQ-002) must exist before root login is disabled,
- continues the planned role separation called out in the project's
  CLAUDE.md (users / firewall / auditd already separate; ssh was the
  remaining concern bundled in baseline).

## 2. What Changed

New role `roles/ssh/`:

- `tasks/main.yml` — deploys the validated sshd drop-in (`sshd -t -f`
  before write, `notify: Restart sshd`).
- `templates/sshd_baseline.conf.j2` — moved from baseline_hardening.
- `handlers/main.yml` — `Restart sshd` (moved from baseline_hardening).
- `defaults/main.yml` — `ssh_permit_root_login` (default `"yes"`),
  `ssh_password_authentication` (default `"no"`), `ssh_dropin_path`.
- `README.md`.

`baseline_hardening` had the SSH task, template, handler, and the
`baseline_sshd_*` defaults removed.

Wiring:

- `playbooks/01-baseline.yml` role order is now
  `baseline_hardening → users → ssh → firewall → auditd`. The `ssh`
  role runs **after** `users` so the admin key is present before root
  login is turned off.
- `group_vars/all.yml` renamed `baseline_sshd_permit_root_login` to
  `ssh_permit_root_login` (still `"no"`, the CRQ-002 target state).

## 3. Validation

- `ansible-playbook playbooks/01-baseline.yml --syntax-check` — pass.
- `ansible-playbook playbooks/03-validation.yml --syntax-check` — pass
  (sshd posture is still asserted by the validation playbook via
  `sshd -T`, unchanged).
- Pending: `--check --diff --tags ssh --limit servera` to confirm the
  drop-in renders identically to the pre-split version (no unintended
  change to `/etc/ssh/sshd_config.d/00-baseline.conf`).

## 4. Rollback

Revert this commit. The drop-in path and contents are unchanged, so no
host-side cleanup is required — re-applying the old baseline produces
the same file.
