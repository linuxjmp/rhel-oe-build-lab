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

## Python / Interpreter Issues

### Symptom: `The module failed to execute ... Unable to find Python`

Ansible requires Python on the managed host to run most modules.
RHEL 8/9 ships Python 3 by default, but a minimal install may omit it.

```bash
ansible -i inventory <host> -m raw -a 'python3 --version'
```

If the command returns `command not found`, install Python:

```bash
# Using raw (no Python needed):
ansible -i inventory <host> -m raw \
  -a 'dnf install -y python3' --become
```

The project `ansible.cfg` sets `interpreter_python = auto_silent` to
suppress the multi-Python warning. If you still see interpreter errors,
pin the interpreter explicitly in the inventory:

```ini
servera ansible_python_interpreter=/usr/bin/python3
```

### Symptom: `SyntaxError` or `import error` in a module

The managed host is running Python 2 but the module requires Python 3.
RHEL 7 ships Python 2 by default. If you must support RHEL 7, set:

```ini
[all:vars]
ansible_python_interpreter=/usr/bin/python3
```

Or install Python 3 on the managed host first (see above).

---

## auditd

### Symptom: `service restart failed — cannot restart auditd`

Under RHEL 8+, `systemctl restart auditd` is blocked by design because
stopping the audit daemon creates an unmonitored window. The correct way
to reload rules is:

```bash
augenrules --load   # compile and load rules from /etc/audit/rules.d/
auditctl -R /etc/audit/audit.rules   # load compiled rules
```

The `auditd` role handles this via a handler that runs `augenrules --load`
instead of restarting the service. If you need to restart auditd manually
(e.g., after changing `auditd.conf`), use:

```bash
service auditd restart   # NOTE: use 'service', not 'systemctl'
```

The `service` command (SysV compatibility) bypasses the systemd guard that
prevents `systemctl restart auditd`.

### Symptom: `auditctl -l` shows no rules after deploying the rule file

The rule file exists but has not been loaded. Run the loader manually:

```bash
augenrules --load
auditctl -l   # should now show the compiled rules
```

If rules are still missing, check the file syntax:

```bash
auditctl --file /etc/audit/rules.d/99-lab-audit.rules
```

A rule syntax error prevents the entire file from loading. Fix the error
and re-run `augenrules --load`.

---

## Package Manager

### Symptom: `Another app is currently holding the yum/dnf lock`

A dnf/yum process is running in the background (update, install, or
`dnf makecache` triggered by a timer). Wait for it or inspect:

```bash
ps aux | grep -E 'dnf|yum'
ls -l /var/run/dnf.pid   # or /run/dnf.pid
```

If the lock is stale (no dnf process running):

```bash
rm -f /var/run/dnf.pid /var/cache/dnf/*.lock
```

The Ansible `dnf` module will wait up to `lock_timeout` seconds
(default 10) before failing. For long updates, increase it:

```yaml
- name: Apply updates
  ansible.builtin.dnf:
    name: '*'
    state: latest
    lock_timeout: 120
```

### Symptom: `dnf update` downloads packages but fails with GPG error

The package signature does not match the trusted key for the repository.
This usually means either a tampered package or an outdated GPG key.

```bash
dnf check-update --nogpgcheck   # confirm the update is available
rpm -q gpg-pubkey               # list installed GPG keys
rpm --import /etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release
```

Never use `--nogpgcheck` in production permanently — it bypasses the
integrity verification that protects the package supply chain.

### Symptom: `dnf` reports `Error: Failed to download metadata`

The package repository is unreachable. Check network and repo config:

```bash
curl -I http://your-repo-host/path/to/repo/repodata/repomd.xml
dnf repolist
cat /etc/yum.repos.d/<your-repo>.repo
```

For RHEL systems using a local ISO mount as the repository, confirm
the mount is active:

```bash
df -T /mnt/repo
ls /mnt/repo/Packages/
```

If the ISO unmounted (e.g., after a reboot), remount it:

```bash
mount /dev/sr0 /mnt/repo
```

---

## DNS and Repository Problems

### Symptom: host is unreachable by name but reachable by IP

The control node cannot resolve the managed host's name.

```bash
dig <hostname>          # or: nslookup <hostname>
cat /etc/hosts          # check for a static entry
ping <hostname>
ping <ip-address>
```

**Fix:** add an entry to `/etc/hosts` on the control node, or configure
DNS to resolve the managed host names. For a lab using RHEL-based VMs,
adding entries to `/etc/hosts` on the control node is the fastest fix:

```
192.168.1.10  servera
192.168.1.11  serverb
```

Test from the control node before trusting Ansible:

```bash
ssh root@servera 'hostname'
```

### Symptom: host resolves but `ansible -m ping` fails with timeout

The managed host may be filtering ICMP (ping) but that does not affect
SSH. Ansible uses SSH, not ICMP. The relevant check is:

```bash
ssh -v root@<host> 'true'
```

Look for the SSH-level error in the verbose output.

---

## Inventory Mistakes

### Symptom: `Could not match supplied host pattern`

The host or group name in the `--limit` or `hosts:` field does not match
anything in the inventory.

```bash
ansible-inventory --list   # shows the parsed inventory as JSON
ansible-inventory --graph  # shows the group hierarchy
ansible <bad-pattern> --list-hosts   # shows what matches
```

Common causes:
- Typo in the `--limit` value (e.g., `Servera` instead of `servera`)
- The group is `[production]` but the playbook says `hosts: managed`
- The inventory file has Windows line endings (CRLF) from a text editor

Fix CRLF line endings:

```bash
sed -i 's/\r//' inventory
```

### Symptom: `Warning: provided hosts list is empty, only localhost will be affected`

The `hosts:` pattern in the playbook matched zero hosts. The play
runs against localhost (the control node) instead of managed hosts,
which is almost never what you want.

Check:
1. The inventory file path: `ansible.cfg` sets `inventory = ./inventory`.
   Run playbooks from the project root.
2. The `hosts:` field in the playbook matches the inventory group name.
3. The host/group is not accidentally commented out in the inventory file.

### Symptom: Ansible connects to the wrong user

`ansible_user` in inventory takes precedence over the value in
`ansible.cfg`. Check all three places:

```bash
ansible-inventory --host servera   # shows all effective variables
```

If the same variable is defined in `group_vars/all.yml`, `inventory`,
and `host_vars/servera.yml`, `host_vars/` wins (highest precedence).

---

## When in Doubt — Capture First, Fix Second

```bash
ansible -i inventory production -m setup > facts-$(date +%F).json
```

Attach the facts dump and the failing command output to the
relevant incident or change record. A reproducible failure is a
solvable failure; an undocumented failure is a recurring one.
