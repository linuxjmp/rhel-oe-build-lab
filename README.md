# rhel-oe-build-lab

**Quarterly RHEL Operating Environment Build and Release Pipeline**

A portfolio home lab that demonstrates how an infrastructure engineer
builds, patches, validates, and rolls back a Red Hat-compatible
Operating Environment. Ansible enforces a security baseline, applies
quarterly updates, validates end state, and documents every step the
way a real operations team would.

---

## Skills Demonstrated

| Domain | What the lab shows |
|--------|-------------------|
| RHEL system administration | SELinux, firewalld, auditd, chrony, sshd, sudo |
| Ansible automation | Roles, playbooks, templates, handlers, variables, idempotency |
| Baseline hardening | CIS/STIG-aligned controls: SELinux enforcing, key-only SSH, auditd rules |
| User and sudo management | Group creation, SSH authorized keys, sudoers drop-ins, break-glass account |
| Firewall management | Declarative firewalld zone and service management |
| Audit logging | auditd rules covering identity, privilege, SSH config, and time changes |
| Quarterly patch lifecycle | Snapshot → update → reboot check → report → validate |
| Validation and reporting | Ansible assert-based checks with clear PASS/FAIL output |
| Change control | Full CRQ records with risk, pre-checks, rollback plan, and results |
| Rollback planning | Safe single-host rollback playbook with confirmation gates |
| Documentation | Recruiter-ready README, architecture, troubleshooting, lessons learned |

---

## Architecture

```
+---------------------+
| control-node        |
| Ansible + Git       |
+----------+----------+
           |
           | SSH (key-based) → sudo → root
           |
+----------+---+   +----------+   +----------+   +----------+
|  servera     |   |  serverb |   |  serverc |   |  serverd |
|  production  |   |          |   |          |   |          |
+--------------+   +----------+   +----------+   +----------+
```

See [architecture.md](architecture.md) for the component diagram,
data flow, security posture, and failure modes.

---

## Repository Layout

```
rhel-oe-build-lab/
├── ansible.cfg                       # Project-scoped Ansible config
├── inventory                         # [production] group: servera–serverd
├── group_vars/all.yml                # Fleet-wide variable overrides
├── requirements.yml                  # Collection dependencies
│
├── roles/
│   ├── baseline_hardening/           # SELinux, firewalld, auditd, chrony, SSH
│   ├── users/                        # Groups, users, SSH keys, sudoers
│   ├── firewall/                     # Declarative firewalld rule management
│   ├── auditd/                       # Audit rules (99-lab-audit.rules)
│   └── patching/                     # Quarterly update workflow + facts
│
├── playbooks/
│   ├── 01-baseline.yml               # Apply full baseline to fleet
│   ├── 02-quarterly-update.yml       # Patch + report
│   ├── 03-validation.yml             # Assert expected end state
│   ├── 04-rollback-checklist.yml     # Safe guided rollback (single host)
│   └── templates/
│       └── quarterly_report.md.j2   # Per-host update report template
│
├── change-records/
│   └── CRQ-001-quarterly-oe-update.md
│
├── reports/                          # Per-host patch reports (git-tracked)
├── screenshots/                      # Portfolio evidence
└── docs/
    ├── validation-guide.md
    ├── troubleshooting.md
    └── lessons-learned.md
```

---

## Lab Environment Assumptions

- **OS:** RHEL 8/9 or compatible (Rocky Linux, AlmaLinux)
- **Control node:** Any Linux host with `ansible-core` and `git`
- **Managed hosts:** `servera`, `serverb`, `serverc`, `serverd` — reachable by DNS or `/etc/hosts`
- **Access:** SSH key-based auth from the control node; currently connects as `root` (see [architecture.md § Current Deviation](architecture.md))
- **Collections:** `ansible.posix` — installed via `ansible-galaxy collection install -r requirements.yml`

---

## Quick Start

```bash
# 1. Install dependencies on the control node
sudo dnf install -y ansible-core git

# 2. Clone and enter the repo
git clone <repo-url>
cd rhel-oe-build-lab

# 3. Install Ansible collections
ansible-galaxy collection install -r requirements.yml

# 4. Verify connectivity to all managed hosts
ansible production -m ping
```

---

## Running the Baseline

Always follow: **syntax-check → dry-run → apply → validate**

```bash
# Step 1: Confirm YAML is well-formed
ansible-playbook playbooks/01-baseline.yml --syntax-check

# Step 2: Preview changes without touching anything
ansible-playbook playbooks/01-baseline.yml --check --diff

# Step 3: Apply the baseline to all hosts
ansible-playbook playbooks/01-baseline.yml

# Step 4: Validate the result
ansible-playbook playbooks/03-validation.yml
```

To target a single host:

```bash
ansible-playbook playbooks/01-baseline.yml --limit servera
```

To re-apply only one concern after a config change:

```bash
ansible-playbook playbooks/01-baseline.yml --tags auditd
ansible-playbook playbooks/01-baseline.yml --tags ssh
ansible-playbook playbooks/01-baseline.yml --tags firewall
```

---

## Running the Quarterly Update

```bash
# Dry-run the canary host first
ansible-playbook playbooks/02-quarterly-update.yml --check --diff --limit servera

# Apply to the canary
ansible-playbook playbooks/02-quarterly-update.yml --limit servera

# Inspect the per-host report
cat reports/*-servera.md

# Validate the canary before continuing
ansible-playbook playbooks/03-validation.yml --limit servera

# Apply to the remaining fleet
ansible-playbook playbooks/02-quarterly-update.yml
```

To allow automatic reboots when the OS requires one:

```bash
ansible-playbook playbooks/02-quarterly-update.yml -e patching_allow_reboot=true
```

To apply security errata only (instead of all updates):

```bash
ansible-playbook playbooks/02-quarterly-update.yml -e patching_update_mode=security
```

---

## Running Validation

```bash
# Full fleet
ansible-playbook playbooks/03-validation.yml

# Single host
ansible-playbook playbooks/03-validation.yml --limit servera

# Save output as portfolio evidence
ansible-playbook playbooks/03-validation.yml 2>&1 | \
  tee reports/validation-$(date +%Y%m%d).txt
```

A clean run ends with every host showing `ok=N  failed=0`.

---

## Running the Rollback Checklist

The rollback playbook targets exactly **one host** and requires typed
confirmation before performing any destructive action.

```bash
# 1. Identify the bad dnf transaction
ansible servera -m command -a 'dnf history list'

# 2. Run the rollback against one host
ansible-playbook playbooks/04-rollback-checklist.yml --limit servera
# You will be prompted for the transaction ID and a confirmation word

# 3. Validate the rolled-back host
ansible-playbook playbooks/03-validation.yml --limit servera
```

---

## Role Summary

| Role | Manages |
|------|---------|
| `baseline_hardening` | Packages, SELinux, firewalld, auditd, chrony, SSH drop-in, wheel sudoers |
| `users` | Groups, user accounts, SSH authorized keys, admin sudoers drop-in, break-glass account |
| `firewall` | Declarative firewalld zone, services, and ports |
| `auditd` | Audit rules: identity, sudo, SSH config, time, hostname, failed access |
| `patching` | Pre/post snapshots, dnf update, reboot check, published facts for reports |

---

## Expected Validation Output

```
PLAY RECAP *****
servera : ok=19  changed=0  unreachable=0  failed=0
serverb : ok=19  changed=0  unreachable=0  failed=0
serverc : ok=19  changed=0  unreachable=0  failed=0
serverd : ok=19  changed=0  unreachable=0  failed=0
```

Each `assert` task prints either `PASS —` or `FAIL —` with a
specific message so deviations are immediately actionable.

---

## Portfolio Evidence

| File | What it shows |
|------|---------------|
| `screenshots/ansible-ping-success.txt` | Control node reaches all four managed hosts |
| `screenshots/validation-output.txt` | Validation playbook — all checks passed, 0 failures |
| `reports/20260606T145955-*.md` | Per-host quarterly update reports (before/after snapshots) |
| `reports/validation-20260611.txt` | Full fleet validation run with timing |
| `change-records/CRQ-001-quarterly-oe-update.md` | Complete change record with risk table, results, and lessons learned |

---

## What This Project Proves

1. **Role design:** Five roles with clear responsibilities, sane defaults, and complete documentation.
2. **Idempotency:** Every role and playbook can be re-applied without side effects.
3. **Change discipline:** Syntax-check → dry-run → apply → validate is the only workflow.
4. **Safety by default:** Destructive operations (reboot, rollback) require explicit opt-in variables or typed confirmation.
5. **Audit trail:** Change records, patch reports, and validation output are first-class deliverables, not afterthoughts.
6. **Operations thinking:** The validation, troubleshooting, and lessons-learned docs show how an engineer thinks about failure, not just success.

---

## Resume Bullet

> Built a simulated Red Hat Operating Environment release pipeline
> using Ansible, Git, and RHEL-compatible Linux VMs to automate
> baseline hardening, quarterly patch deployment, post-change
> validation, and rollback documentation across four managed hosts.

---

## Planned Improvements

- **CRQ-002:** Create a dedicated `admin` user via the `users` role and disable root SSH in `baseline_hardening`
- **CRQ-003:** Add inline post-host validation to `02-quarterly-update.yml` to gate serial rollout
- **Separate roles:** Split `users/`, `firewall/`, and `auditd/` concerns further (e.g., separate `ssh` role)
- **Scheduled runs:** systemd timer or cron job to run `03-validation.yml` weekly for drift detection


---

## Docs

- [Architecture](architecture.md)
- [Validation Guide](docs/validation-guide.md)
- [Troubleshooting](docs/troubleshooting.md)
- [Lessons Learned](docs/lessons-learned.md)
