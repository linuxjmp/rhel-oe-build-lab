# Troubleshooting Guide

A field guide for the failures you will hit while building and
operating the `rhel-oe-build-lab`. Each section: symptom →
diagnostic → fix.

---

## SSH and Connectivity

### Symptom: `UNREACHABLE!` from `ansible -m ping`

```bash
ansible -i inventory <host> -m ping -vvv
```

Look for the SSH-level error:

- `Permission denied (publickey)` → key not installed on the host
  or the wrong `ansible_user`.
- `Connection refused` → `sshd` is not running, or SSH is on a
  non-standard port not declared in inventory.
- `No route to host` → network or DNS problem.
- `Host key verification failed` → known_hosts mismatch after a
  re-image. Remove the stale entry with `ssh-keygen -R <host>`.

Sanity baseline:

```bash
ping -c 2 <host>
ssh -v root@<host> 'true'
```

### Symptom: `sudo: a password is required`

`become: true` needs passwordless sudo or `--ask-become-pass`.
For passwordless sudo on a managed host:

```
%wheel ALL=(ALL) NOPASSWD: ALL
```

Place in `/etc/sudoers.d/wheel-nopass`, owned `root:root` mode
`0440`, and validate with `visudo -c`.

---

## Ansible Playbook Errors

### Symptom: `ERROR! Syntax Error while loading YAML`

```bash
ansible-playbook playbooks/<file>.yml --syntax-check
```

Common causes:

- Tabs instead of spaces (YAML requires spaces).
- Misaligned indentation.
- Missing colon after a key.
- An unquoted value containing `:` or `#`.

### Symptom: `The conditional check ... failed`

A variable was referenced before it was defined. Print the host's
facts and confirm the variable name:

```bash
ansible -i inventory <host> -m setup -a 'filter=ansible_*'
```

Wrap the reference with `default()` if it might legitimately be
absent.

### Symptom: Task hangs forever

Usually `pipelining = True` combined with `Defaults requiretty`
in sudoers. Confirm with:

```bash
ANSIBLE_PIPELINING=False ansible-playbook ...
```

If that fixes it, remove `Defaults requiretty` from sudoers and
re-enable pipelining.

---

## SELinux

### Symptom: Service won't start after a change

```bash
getenforce
ausearch -m AVC -ts recent
sealert -a /var/log/audit/audit.log
```

If a denial is the cause, prefer one of:

- `semanage fcontext` + `restorecon` (file labels)
- `semanage port` (port labels)
- Targeted policy module via `audit2allow -M`

Avoid `setenforce 0` outside of a deliberate diagnostic window.
Any permissive period must be recorded in the change record.

---

## firewalld

### Symptom: service is up, but port is closed from outside

```bash
firewall-cmd --get-default-zone
firewall-cmd --list-all
ss -tulpn | grep <port>
```

If `ss` shows the service listening but firewalld blocks it, add
the service to the right zone permanently:

```bash
firewall-cmd --permanent --add-service=<svc>
firewall-cmd --reload
```

Then encode the change as an Ansible task so it persists across
re-runs and rebuilds.

---

## Services and systemd

### Symptom: service in failed state after update

```bash
systemctl status <svc>
journalctl -u <svc> -n 100 --no-pager
journalctl -u <svc> -p err --since "1 hour ago"
```

Common patterns:

- Config syntax error → `Invalid` or `cannot bind` in the log.
- Missing dependency → `Failed to start dep`.
- SELinux denial → cross-check with `ausearch`.

---

## Patching

### Symptom: `dnf update` returns "Nothing to do" but updates were expected

```bash
dnf clean all
dnf makecache
dnf check-update
subscription-manager status   # RHEL only
```

### Symptom: System fails to boot after kernel update

From the console (not SSH), select the previous kernel from GRUB.
Then make it the default while the issue is diagnosed:

```bash
grub2-set-default 1
grub2-mkconfig -o /boot/grub2/grub.cfg
```

Capture failed-boot logs:

```bash
journalctl --boot=-1 -p err
```

Document the regression in the change record and execute the
rollback per `playbooks/04-rollback-checklist.yml` (once defined).

---

## Networking

### Symptom: host loses network after a change

From the console (not SSH):

```bash
ip addr
ip route
nmcli connection show
journalctl -u NetworkManager -n 200 --no-pager
```

If an `nmcli` change broke the primary interface, revert to the
previous profile:

```bash
nmcli connection up <previous-profile>
```

---

## Validation Playbook False Positives

### Symptom: disk pressure assert fails on `/mnt/repo` at 100%

`/dev/sr0` (or another block device) is mounted at `/mnt/repo` as
an ISO-based local package repository. Read-only optical and
loop-mounted devices always report 100% capacity — that is not disk
pressure.

Confirm the mount type:

```bash
df -T /mnt/repo
```

Expected output for an ISO mount:

```
Filesystem     Type  1K-blocks  Used Available Use% Mounted on
/dev/sr0       iso9660  ...      100% /mnt/repo
```

**Fix:** the validation playbook excludes iso9660 via
`df -x iso9660`. If you encounter a similar false positive on a
different read-only filesystem type (e.g. `udf`, `squashfs` for a
snap package), add `-x <type>` to the `df` command in
`playbooks/03-validation.yml`.

**Rule of thumb:** before writing a disk-pressure check, run
`df -T` and inspect *every* filesystem type in the output. Any
read-only or ephemeral type should be excluded.

---

## When in Doubt — Capture First, Fix Second

```bash
ansible -i inventory production -m setup > facts-$(date +%F).json
```

Attach the facts dump and the failing command output to the
relevant incident or change record. A reproducible failure is a
solvable failure; an undocumented failure is a recurring one.
