# CRQ-002: Create Admin User and Disable Root SSH

| Field           | Value                                                              |
|-----------------|--------------------------------------------------------------------|
| Change ID       | CRQ-002                                                            |
| Title           | Create dedicated admin user; disable direct root SSH login         |
| Owner           | linuxjmp                                                           |
| Date submitted  | 2026-06-15                                                        |
| Scheduled start | 2026-06-15                                                        |
| Scheduled end   | 2026-06-15                                                        |
| Risk level      | High                                                               |
| Status          | Completed                                                          |

---

## 1. Reason for Change

The `rhel-oe-build-lab` fleet currently connects to managed hosts as
`root` over SSH, which is documented as a **lab convenience deviation**
in `architecture.md`. This is not acceptable as a long-term posture:

- Direct root SSH login bypasses the separation of privilege that
  `sudo` provides — there is no audit trail for individual operators.
- SSH brute-force attacks targeting `root` are constant background
  noise on any internet-exposed host.
- The STIG and CIS benchmarks for RHEL both require `PermitRootLogin no`.

This change transitions the lab to the intended target state:

1. A dedicated `admin` account is created on all hosts via the `users`
   role with an operator SSH public key.
2. `PermitRootLogin` is changed from `yes` to `no` in
   `roles/baseline_hardening/defaults/main.yml`.
3. The inventory is updated to use `ansible_user=admin`.

---

## 2. Systems Affected

| Host    | Group       | Impact                                    |
|---------|-------------|-------------------------------------------|
| servera | production  | New user created; root SSH disabled        |
| serverb | production  | New user created; root SSH disabled        |
| serverc | production  | New user created; root SSH disabled        |
| serverd | production  | New user created; root SSH disabled        |

Out of scope: `control-node`.

---

## 3. Risk Assessment

| Risk                                    | Likelihood | Impact | Mitigation                                                          |
|-----------------------------------------|------------|--------|---------------------------------------------------------------------|
| SSH key not deployed before root locked | Medium     | High   | Verify key login as admin user on all hosts before disabling root   |
| `sudoers` misconfiguration locks admin  | Low        | High   | Validate sudoers with `visudo -c` before and after; test sudo       |
| One host missed during key deployment   | Low        | High   | `serial: 1` in playbook; validate each host individually            |
| Inventory change breaks other playbooks | Low        | Medium | All playbooks use `hosts: production` group — `ansible_user` change applies fleet-wide |

**Risk rating: High** — any mistake in this sequence can lock the
operator out of a host. The mitigation is strict sequencing and
validation at each step before proceeding to the next host.

---

## 4. Pre-Checks

Before making any changes, confirm the current state:

```bash
# Confirm all hosts are reachable as root
ansible production -m ping

# Confirm root SSH is currently permitted
ansible production -m shell -a 'sshd -T | grep permitrootlogin'

# Capture current sudoers state
ansible production -m shell -a 'visudo -c' --become

# Verify the admin SSH key is ready on the control node
ls -l ~/.ssh/id_*.pub
```

---

## 5. Implementation Steps

Execute in this exact sequence. Do not disable root SSH until step 4
is confirmed clean.

### Step 1: Add the admin user to `group_vars/all.yml`

Add the operator account to `users_list`. Replace the placeholder
public key with the actual key from the control node:

```yaml
# group_vars/all.yml
users_list:
  - name: admin
    comment: "Lab Admin Account"
    groups:
      - admin
      - wheel
    shell: /bin/bash
    ssh_authorized_keys:
      - "ssh-ed25519 AAAA... admin@control-node"
```

Get the public key:

```bash
cat ~/.ssh/id_ed25519.pub   # or id_rsa.pub
```

### Step 2: Apply the `users` role to create the account

```bash
ansible-playbook playbooks/01-baseline.yml --tags users
```

### Step 3: Verify SSH key login as admin on every host

Do this manually before touching sshd config:

```bash
ssh -i ~/.ssh/id_ed25519 admin@servera 'id; sudo -l'
ssh -i ~/.ssh/id_ed25519 admin@serverb 'id; sudo -l'
ssh -i ~/.ssh/id_ed25519 admin@serverc 'id; sudo -l'
ssh -i ~/.ssh/id_ed25519 admin@serverd 'id; sudo -l'
```

Expected output: `uid=NNNN(admin)` and `(ALL) NOPASSWD: ALL` in `sudo -l`.
**Do not proceed past this step if any host fails.**

### Step 4: Disable root SSH in `group_vars/all.yml`

```yaml
# group_vars/all.yml
ssh_permit_root_login: "no"
```

> Note: the SSH posture was originally a `baseline_hardening` default
> but was later split into a dedicated `ssh` role under CRQ-004. The
> variable is now `ssh_permit_root_login`, consumed by the `ssh` role.

Apply the SSH change:

```bash
ansible-playbook playbooks/01-baseline.yml --tags ssh
```

### Step 5: Update the inventory to use the admin user

```ini
# inventory
[all:vars]
ansible_user=admin
ansible_become=true
```

### Step 6: Verify the control node can still reach all hosts

```bash
ansible production -m ping
ansible production -m shell -a 'whoami; sudo whoami'
```

Expected: `admin` for the first command, `root` for the second.

### Step 7: Confirm root SSH is now blocked

```bash
ansible production -m shell -a 'sshd -T | grep permitrootlogin'
```

Expected: `permitrootlogin no` on all hosts.

---

## 6. Validation Steps

Run the full validation playbook:

```bash
ansible-playbook playbooks/03-validation.yml 2>&1 | \
  tee reports/validation-crq002-$(date +%Y%m%d).txt
```

Additional manual checks:

```bash
# Confirm root SSH is rejected (will fail — that is the expected result)
ssh root@servera 'true'
# Expected: "Permission denied (publickey)" or "root login not permitted"

# Confirm admin SSH works
ssh admin@servera 'sudo systemctl status sshd'
```

---

## 7. Rollback Plan

If any host loses SSH access:

1. Use console access (IPMI, iDRAC, VM console) to log in as root.
2. Set `PermitRootLogin yes` temporarily in `/etc/ssh/sshd_config.d/00-baseline.conf`.
3. Restart sshd: `service sshd restart`
4. Reconnect from the control node and diagnose.
5. Once access is restored, identify the root cause before re-applying.

The rollback is manual because the automation itself is what requires
the operator SSH key. Keep console access available throughout this change.

---

## 8. Communication Plan

Notify any other operators or team members:

- The inventory `ansible_user` will change from `root` to `admin`.
- Anyone who has SSH configs pointing to these hosts as `root` must
  update their configs after this change.
- The control node SSH key must be in place before the change window opens.

---

## 9. Results

| Field            | Value |
|------------------|-------|
| Actual start     | 2026-06-15 |
| Actual end       | 2026-06-15 |
| Outcome          | Success — `admin` created fleet-wide, inventory switched to `ansible_user=admin`, direct root SSH and SSH password auth disabled. |
| Hosts succeeded  | servera, serverb, serverc, serverd |
| Hosts deferred   | None |
| Attached output  | [`reports/validation-20260617.txt`](../reports/validation-20260617.txt) — `sshd: PasswordAuthentication no`, `ok=21 failed=0` per host |
| Git commit / tag | `9b6a82b` (admin user + root disable); SSH posture later split to the `ssh` role in `c0d911c` (CRQ-004) |

The latest validation run confirms the end state holds: every host
reports `PasswordAuthentication no` and passes all 21 checks with zero
failures. Direct root SSH login is rejected.

---

## 10. References

- [architecture.md — Access Model (CRQ-002)](../architecture.md)
- [roles/baseline_hardening/README.md — Root Login Default](../roles/baseline_hardening/README.md)
- [roles/users/README.md](../roles/users/README.md)
- [docs/lessons-learned.md](../docs/lessons-learned.md)
