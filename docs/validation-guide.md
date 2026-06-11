# Validation Guide

Standard checks to run before, during, and after any change to the
`rhel-oe-build-lab` Operating Environment. Validation is the line
between a script and a change.

> A change is not done until it has been validated.
> A validation is not done until its output has been captured.

Every playbook run should be bracketed by the checks below.
Capture the output to `screenshots/` or to the relevant change
record under `change-records/`.

---

## Pre-Change Validation

### 1. Confirm the control node sees every host

```bash
ansible -i inventory production -m ping
```

Expected pattern:

```
servera | SUCCESS => {
    "changed": false,
    "ping": "pong"
}
```

An `UNREACHABLE!` result means the control node cannot SSH to the
host. See [troubleshooting.md](troubleshooting.md).

### 2. Syntax-check the playbook

```bash
ansible-playbook -i inventory playbooks/01-baseline.yml --syntax-check
```

Expected: `playbook: playbooks/01-baseline.yml` with no error.

### 3. Dry-run with diff

```bash
ansible-playbook -i inventory playbooks/01-baseline.yml --check --diff
```

Expected: a task list marked `ok`/`changed`/`skipped` and no red
`failed` entries. Tasks reported as `changed` in `--check` mode
are the ones that will mutate state on the next real run — review
them carefully before applying.

### 4. Snapshot current state

```bash
ansible -i inventory production -m shell -a 'getenforce; \
  systemctl is-active firewalld auditd chronyd sshd; \
  rpm -qa | sort > /tmp/pkgs-pre.txt; \
  uname -r'
```

Save the output to the change record under "Pre-checks".

---

## Post-Change Validation

### 1. Service state

```bash
ansible -i inventory production -m shell \
  -a 'systemctl is-active firewalld auditd chronyd sshd'
```

Every entry should print `active`.

### 2. SELinux

```bash
ansible -i inventory production -m shell -a 'getenforce'
```

Expected: `Enforcing` on every host.

### 3. firewalld zones and ports

```bash
ansible -i inventory production -m shell \
  -a 'firewall-cmd --get-default-zone; firewall-cmd --list-all'
```

Expected: default zone matches the baseline (e.g. `public`); only
approved services listed.

### 4. SSH posture

```bash
ansible -i inventory production -m shell \
  -a 'sshd -T | grep -E "permitrootlogin|passwordauthentication"'
```

Target state:

```
permitrootlogin no
passwordauthentication no
```

### 5. Time sync

```bash
ansible -i inventory production -m shell \
  -a 'chronyc tracking | grep -E "Leap status|System time"'
```

Expected: `Leap status : Normal` and a small `System time` offset
(milliseconds, not seconds).

### 6. Disk pressure

```bash
ansible -i inventory production -m shell \
  -a 'df -h --output=source,pcent,target | awk "NR==1 || \$2+0 > 70"'
```

Expected: no filesystem above the threshold defined by the baseline
(default 70%).

### 7. Package drift

```bash
ansible -i inventory production -m shell \
  -a 'rpm -qa | sort > /tmp/pkgs-post.txt; \
      diff /tmp/pkgs-pre.txt /tmp/pkgs-post.txt | head'
```

Attach the diff to the change record.

---

## Continuous Validation

`playbooks/03-validation.yml` encodes all of the post-change checks
above as idempotent Ansible `assert` tasks. Run it after every
change and whenever you want a posture snapshot:

```bash
# Full fleet
ansible-playbook playbooks/03-validation.yml

# Single host
ansible-playbook playbooks/03-validation.yml --limit servera

# Save the output as portfolio evidence
ansible-playbook playbooks/03-validation.yml 2>&1 | \
  tee reports/validation-$(date +%Y%m%d).txt
```

Every assert either passes with a `success_msg` or fails with a
`fail_msg` that names the exact control that deviated. A clean run
ends with:

```
servera : ok=13  changed=0  unreachable=0  failed=0
serverb : ok=13  changed=0  unreachable=0  failed=0
serverc : ok=13  changed=0  unreachable=0  failed=0
serverd : ok=13  changed=0  unreachable=0  failed=0
```

### Known False Positive: iso9660 Mounts

The disk pressure check excludes `tmpfs`, `devtmpfs`, `squashfs`,
and `iso9660` filesystem types. Without the iso9660 exclusion, a
RHEL ISO mounted at `/mnt/repo` (used as the local package
repository) reports 100% capacity and trips the assert — even
though the device is read-only by design.

**Lesson:** When writing disk checks, always review the full `df`
output on your target fleet before assuming 100% means a problem.
Read-only mounts (optical media, loop-mounted ISOs, snap packages)
always report full; they must be excluded or checked separately.

---

## Why These Checks Matter

A table that maps each validation check to the real-world risk it
prevents:

| Check | Why it matters |
|-------|----------------|
| SELinux Enforcing | Permissive mode silently allows denials — attackers can exploit unlabeled paths. Enforcing is the only production-safe state. |
| firewalld active | A disabled firewall exposes all ports. The check confirms the daemon is running, not just installed. |
| auditd active | Without auditd, privileged-command logging stops. Compliance frameworks (STIG, CIS) require an audit trail for `su`, `sudo`, and identity changes. |
| chronyd active | Time sync is required for Kerberos, TLS certificate validation, and meaningful log correlation. A drifted clock can break authentication or make incidents impossible to reconstruct. |
| sshd PasswordAuthentication off | Password SSH is vulnerable to brute force. Key-only auth means a stolen password is not enough — an attacker also needs the private key. |
| chrony Leap status Normal | Confirms the daemon has synced, not just that it's running. A running-but-unsynced chrony is as bad as no chrony. |
| Disk pressure ≤ 70% | Filesystem saturation causes service failures, log loss, and kernel panics. The 70% threshold leaves headroom for log rotation and kernel dumps. |
| firewalld services | Confirms the *policy* is correct, not just that the service is running. A firewall running with no rules is not protecting anything. |
