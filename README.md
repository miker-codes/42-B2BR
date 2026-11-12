*This project has been created as part of the 42 curriculum by mruiz-ra.*

# Born2beRoot

## Description

Born2beRoot is an introduction to system administration and virtualization. The goal is to set up a small, secure Linux server inside a virtual machine, following a strict set of rules:

- No graphical interface: only the minimum services a server needs.
- Encrypted disk partitions managed with LVM.
- SSH access on port 4242, with root login disabled.
- A firewall that only leaves port 4242 open.
- A strong password policy and a strict `sudo` configuration.
- A Bash script, `monitoring.sh`, that broadcasts system information to all terminals at startup and every 10 minutes.

The repository only contains this README and `signature.txt`, the SHA1 hash of the virtual machine's disk. The virtual machine itself is not part of the repository.

## Project description

### Operating system choice: Debian

I chose **Debian 13 (trixie)**, the latest stable release.

| | Advantages | Disadvantages |
|---|---|---|
| **Debian** | Simple to administer, very well documented, huge community, stable and predictable releases, recommended by the subject for beginners. | Packages are older than in faster-moving distributions; shorter official support than Rocky. |
| **Rocky Linux** | Binary-compatible with Red Hat Enterprise Linux, long support cycle, common in enterprise environments. | More complex to set up (SELinux, firewalld); smaller community documentation for beginners. |

### Partitioning

Virtual disk: **13 GB**, dynamically allocated (the `.vdi` file only grows as data is written).

```
sda                     13G   disk
├─sda1                 500M   /boot          (not encrypted)
├─sda2                  1K    extended partition
└─sda5                12.5G   LUKS encrypted partition
  └─sda5_crypt                LVM physical volume
    ├─LVMGroup-root      6G   /
    ├─LVMGroup-swap      2G   [SWAP]
    └─LVMGroup-home      3G   /home
```

- **/boot (500 MB, not encrypted)**: holds the kernel and the initramfs. It must stay readable before the disk is unlocked, because the initramfs is the program that asks for the encryption passphrase. 500 MB fits two or three kernels for safe upgrades.
- **LUKS + LVM**: everything else is encrypted with LUKS, and LVM runs on top of it. The three logical volumes are the encrypted partitions required by the subject.
- **/ (6 GB)**: a minimal Debian uses about 2 GB; the rest is room for packages, the apt cache and logs (`/var` is part of `/`).
- **swap (2 GB)**: same size as the VM's RAM, the usual rule for machines with little memory.
- **/home (3 GB)**: the server has very few users and they do not store data.
- **About 2.4 GB left unallocated in the volume group** on purpose: with LVM, `/` or `/home` can be extended later without repartitioning (`lvextend` + `resize2fs`).

### Security policies

**sudo** (`/etc/sudoers.d/sudo_config`):
- 3 password attempts maximum, with a custom error message.
- Every command is logged to `/var/log/sudo/sudo.log`, and its input and output are recorded in `/var/log/sudo/` (replayable with `sudoreplay`).
- `requiretty`: sudo can only be used from a real terminal.
- `secure_path`: sudo ignores the user's `PATH` and only uses trusted directories.

**Password policy**:
- Expiration (`/etc/login.defs` and `chage` for existing users): passwords expire after 30 days, cannot be changed again for 2 days, and users are warned 7 days before expiration.
- Quality (`libpam-pwquality`, `/etc/security/pwquality.conf`): at least 10 characters, with one uppercase letter, one lowercase letter and one digit; no more than 3 identical characters in a row; must not contain the username; at least 7 characters different from the previous password (not applied to root, which does not type the old password); rules also enforced for root.
- All passwords were changed after the policy was configured.

**SSH**: listens on port 4242, and `PermitRootLogin no` blocks root logins. Administration is done with a personal account plus `sudo`, so every action is traced to a user.

**Firewall**: UFW is enabled at boot, denies all incoming traffic, and only allows port 4242 (IPv4 and IPv6).

**AppArmor**: enabled by default in Debian and running at startup. It restricts what each program can access through per-program profiles.

### User management

- `root` and a personal user `mruiz-ra`.
- `mruiz-ra` belongs to the `sudo` and `user42` groups.
- The hostname is `mruiz-ra42`.

### Services installed

Only what the subject requires: **OpenSSH server**, **UFW**, **sudo**, **libpam-pwquality** and **cron** (part of the base system), plus the standard system utilities. No graphical server (X.org or Wayland) is installed.

### Monitoring script

`/usr/local/bin/monitoring.sh` collects the following data and broadcasts it with `wall`: architecture and kernel, physical and virtual CPUs, RAM usage, disk usage, CPU load, last boot, LVM status, established TCP connections, number of logged-in users, IPv4 and MAC address, and the number of commands run with `sudo`. Root's crontab runs it at boot (`@reboot`) and every 10 minutes (`*/10 * * * *`).

> Note on Debian 13: `/run/utmp` no longer exists, so `wall` gets the list of sessions from systemd-logind, which does not record the terminal of SSH sessions. The message therefore appears on the VM console but not in SSH sessions.

<details>
<summary><b>monitoring.sh, line by line</b></summary>

Every line follows the same pattern: `variable=$( source | filter | count or format )`. `$( )` stores the output of a command in a variable, and `|` passes the output of one command to the next.

```bash
#!/bin/bash

# System
arch=$(uname -a)
pcpu=$(grep "physical id" /proc/cpuinfo | sort -u | wc -l)
vcpu=$(grep -c "^processor" /proc/cpuinfo)

# Resources
ram=$(free -m | awk '$1 == "Mem:" {printf "%d/%dMB (%.2f%%)", $3, $2, $3 * 100 / $2}')
disk=$(df -m -x tmpfs -x devtmpfs --total | awk '$1 == "total" {printf "%d/%dGb (%d%%)", $3, $2 / 1024, $3 * 100 / $2}')
cpu=$(vmstat 1 2 | tail -1 | awk '{printf "%.1f%%", 100 - $15}')

# Status
lastboot=$(who -b | awk '{print $3 " " $4}')
if lsblk -o TYPE | grep -q "lvm"; then lvm="yes"; else lvm="no"; fi
tcp=$(ss -Ht state established | wc -l)
ulog=$(who | awk '{print $1}' | sort -u | wc -l)

# Network
ip=$(hostname -I | awk '{print $1}')
mac=$(ip link show | awk '/link\/ether/ {print $2; exit}')

# Sudo
sudo_cmds=$(grep -c "TSID=" /var/log/sudo/sudo.log 2>/dev/null)

wall "	#Architecture: $arch
	...
	#Sudo: $sudo_cmds cmd"
```

| Line | Source | Filter / processing | Result |
|---|---|---|---|
| **Architecture** | `uname -a` | none | Kernel name, hostname, kernel version, architecture |
| **Physical CPU** | `/proc/cpuinfo` (one block per CPU) | `grep "physical id"` keeps the socket line of each CPU, `sort -u` removes duplicates | `wc -l` counts the different physical sockets |
| **vCPU** | `/proc/cpuinfo` | `grep "^processor"` keeps one line per logical CPU | `-c` counts them |
| **Memory** | `free -m` (RAM in MB) | `awk` keeps the `Mem:` row: column 3 = used, column 2 = total | `printf` prints `used/total MB (percentage)` |
| **Disk** | `df -m` (space per filesystem, in MB) | `-x tmpfs -x devtmpfs` excludes RAM filesystems, `--total` adds a sum row | `awk` prints `used MB / total GB (percentage)` from the `total` row |
| **CPU load** | `vmstat 1 2` (two samples, one second apart) | `tail -1` keeps the second sample; column 15 is the idle CPU % | `100 - idle` = CPU in use |
| **Last boot** | `who -b` (`system boot 2026-11-05 11:52`) | `awk` keeps columns 3 and 4 | Date and time |
| **LVM use** | `lsblk -o TYPE` (type of each device) | `grep -q "lvm"` only answers yes or no | `if` prints `yes` or `no` |
| **TCP connections** | `ss -t` (TCP connections) | `state established` keeps active ones, `-H` removes the header | `wc -l` counts them |
| **User log** | `who` (one line per session) | `awk '{print $1}'` keeps the username, `sort -u` removes duplicates | `wc -l` counts distinct users |
| **IP** | `hostname -I` (all IP addresses) | `awk '{print $1}'` keeps the first one | IPv4 address |
| **MAC** | `ip link show` (network interfaces) | `awk` finds the first `link/ether` line and prints column 2 | MAC address |
| **Sudo** | `/var/log/sudo/sudo.log` | `grep "TSID="`: only commands that actually ran have a TSID (failed passwords do not) | `-c` counts them; `2>/dev/null` hides errors |
| **wall** | all the variables above | | Sends the message to the terminal of every open session |

Common pieces: `grep` keeps lines containing a text (`-c` counts, `-q` only answers yes/no), `wc -l` counts lines, `awk '{print $N}'` keeps column N, `sort -u` removes duplicates, `printf` formats numbers (`%d` integer, `%.2f` two decimals, `%%` a literal `%`).

</details>

### Comparisons

**Debian vs Rocky Linux**

| | Debian | Rocky Linux |
|---|---|---|
| Origin | Community project since 1993 | RHEL rebuild, created in 2020 after CentOS changed direction |
| Package manager | `apt` / `.deb` | `dnf` / `.rpm` |
| Release cycle | New stable release about every 2 years, ~5 years of support (with LTS) | Follows RHEL, ~10 years of support |
| Default security module | AppArmor | SELinux |
| Default firewall tool | None (UFW installed here) | firewalld |
| Typical use | Servers and desktops, very wide adoption | Enterprise servers |

**AppArmor vs SELinux**

Both are Mandatory Access Control (MAC) systems: they limit what a program can do even if it runs as root or is compromised.

| | AppArmor | SELinux |
|---|---|---|
| How rules are defined | By file paths, one profile per program | By labels: every file, process and port has a security context |
| Complexity | Easier to read and write | More powerful and fine-grained, but harder to configure |
| Modes | enforce / complain | enforcing / permissive / disabled |
| Default in | Debian, Ubuntu | RHEL, Rocky, Fedora |

**UFW vs firewalld**

| | UFW | firewalld |
|---|---|---|
| Approach | Simple list of allow/deny rules | Zones (trust levels per network interface) and services |
| Interface | `ufw allow 4242` | `firewall-cmd --add-port=4242/tcp --permanent` |
| Backend | iptables/nftables | nftables |
| Runtime changes | Reloads its rules | Applies changes without restarting the firewall |
| Default in | Ubuntu (available on Debian) | RHEL, Rocky, Fedora |

**VirtualBox vs UTM**

| | VirtualBox | UTM |
|---|---|---|
| Developer | Oracle (open source) | Open-source project based on QEMU |
| Host systems | Windows, Linux, Intel macOS | macOS and iOS (Apple Silicon and Intel) |
| How it works | Type 2 hypervisor for x86 guests | Uses Apple's Hypervisor framework for same-architecture guests, QEMU emulation for others |
| When to use | Standard choice on x86 machines | Macs with Apple Silicon, where VirtualBox support is limited |

## Instructions

### Requirements

- VirtualBox installed on the host.
- The virtual machine (`born2beroot.vbox` and `born2beroot.vdi`), which is not included in this repository.

### Verify the disk signature

Before starting the VM, compare the hash of the disk with `signature.txt`:

```bash
sha1sum born2beroot.vdi
```

Starting the VM modifies the disk, so the evaluation must start from an untouched copy and with no existing snapshots.

### Start and connect

1. Start the VM in VirtualBox and type the disk encryption passphrase.
2. Log in on the console, or connect via SSH from the host. The VM uses NAT networking, so a VirtualBox port forwarding rule is needed (Settings → Network → Advanced → Port Forwarding). Here, host port 2222 is forwarded to guest port 4242:

```bash
ssh mruiz-ra@localhost -p 2222
```

## Verification commands

Commands to check every requirement of the project, grouped by topic. Click a section to expand it.

<details>
<summary><b>General checks</b></summary>

```bash
head -n 2 /etc/os-release                      # Debian GNU/Linux 13 (trixie)
dpkg -l | grep -E "xorg|wayland"               # empty output = no graphical server
sudo aa-status | head -1                       # "apparmor module is loaded."
systemctl is-enabled apparmor ufw ssh cron     # "enabled" x4: everything starts at boot
```

</details>

<details>
<summary><b>Users and groups</b></summary>

```bash
groups mruiz-ra                        # mruiz-ra is in sudo and user42
getent group sudo user42               # members of each group
sudo adduser newuser                   # create a user (home directory + password, policy applies)
sudo addgroup evaluating               # create a group
sudo usermod -aG evaluating newuser    # add the user to the group (-a = append, keep other groups)
groups newuser                         # check
sudo chage -l newuser                  # expiration: min 2 / max 30 / warning 7 days
sudo deluser --remove-home newuser     # delete the user and its home
sudo delgroup evaluating               # delete the group
```

Password policy files: `/etc/login.defs` (expiration for new users), `chage` (existing users), `/etc/security/pwquality.conf` (password quality).

</details>

<details>
<summary><b>Hostname and partitions</b></summary>

```bash
hostnamectl                                  # current hostname
sudo nano /etc/hosts                         # 1. change the 127.0.1.1 line to the new name
sudo hostnamectl set-hostname newname42      # 2. change the hostname (writes /etc/hostname)
sudo reboot                                  # show that the change persists
hostname                                     # check
lsblk                                        # partitions, LUKS and LVM volumes
```

Editing `/etc/hosts` first avoids the `sudo: unable to resolve host` warning.

</details>

<details>
<summary><b>sudo</b></summary>

```bash
dpkg -l sudo                                    # "ii" = installed
sudo usermod -aG sudo newuser                   # give sudo rights to a user
getent group sudo                               # check
sudo cat /etc/sudoers.d/sudo_config             # sudo rules
sudo ls /var/log/sudo                           # sudo.log, recordings (00/) and counter (seq)
sudo tail -n 4 /var/log/sudo/sudo.log           # last commands run with sudo
sudo sudoreplay -d /var/log/sudo -l             # list recorded sessions (TSID)
sudo sudoreplay -d /var/log/sudo <TSID>         # replay one session (input and output)
```

</details>

<details>
<summary><b>UFW</b></summary>

```bash
dpkg -l ufw                       # installed
sudo systemctl status ufw         # enabled at boot; "active (exited)" is normal (rules live in the kernel)
sudo ufw status numbered          # active, only port 4242 allowed
sudo ufw allow 8080               # open a new port (IPv4 and IPv6)
sudo ufw delete allow 8080        # remove both rules
sudo ufw delete <number>          # or remove one rule by number (numbers shift after each delete)
```

</details>

<details>
<summary><b>SSH</b></summary>

```bash
sudo systemctl status ssh                                  # enabled + active (running)
sudo ss -tunlp | grep 4242                                 # sshd listening on 4242 (IPv4 and IPv6)
grep -E "^(Port|PermitRootLogin)" /etc/ssh/sshd_config     # Port 4242 / PermitRootLogin no
```

From the host (port 2222 is forwarded to the VM's port 4242):

```bash
ssh newuser@localhost -p 2222     # log in with a new user
ssh root@localhost -p 2222        # must fail: Permission denied
```

</details>

<details>
<summary><b>Monitoring script and cron</b></summary>

```bash
cat /usr/local/bin/monitoring.sh                  # show the script
sudo /usr/local/bin/monitoring.sh                 # run it manually (visible on the VM console)
sudo crontab -l                                   # @reboot and */10 * * * *
sudo crontab -e                                   # change the schedule (e.g. * * * * * = every minute)
sudo systemctl stop cron                          # stop the script without modifying it
sudo systemctl status cron                        # inactive (dead)
sudo systemctl start cron                         # start it again
sudo journalctl -b -u cron | grep monitoring      # proof that @reboot ran at startup
```

Cron format: `minute hour day-of-month month day-of-week command`.

</details>

<details>
<summary><b>Cleanup after an evaluation</b></summary>

```bash
sudo deluser --remove-home newuser     # users created during the evaluation
sudo delgroup evaluating               # groups created during the evaluation
sudo ufw status numbered               # only 4242 must remain
sudo crontab -l                        # schedule back to */10
hostnamectl                            # hostname back to mruiz-ra42 (and /etc/hosts)
```

</details>

## Resources

- [Debian Administrator's Handbook](https://www.debian.org/doc/manuals/debian-handbook/)
- [Debian Wiki: LVM](https://wiki.debian.org/LVM)
- [Debian Wiki: AppArmor](https://wiki.debian.org/AppArmor)
- [Rocky Linux documentation](https://docs.rockylinux.org/)
- [VirtualBox User Manual](https://www.virtualbox.org/manual/)
- Manual pages ([manpages.debian.org](https://manpages.debian.org/)): `sudoers(5)`, `pwquality.conf(5)`, `login.defs(5)`, `chage(1)`, `sshd_config(5)`, `ufw(8)`, `crontab(5)`, `wall(1)`

### Use of AI

I used Claude (Anthropic) as a guide throughout the project:

- **Explanations**: concepts such as LUKS, LVM, PAM, AppArmor, systemd-logind and how cron works.
- **Step-by-step guidance**: the installation and configuration commands, each explained flag by flag. I ran and checked every step myself on the VM.
- **Troubleshooting and review**: checking my configuration from screenshots and command output (for example, a missing `:` in `secure_path`, and why `wall` does not reach SSH sessions on Debian 13).
- **Monitoring script**: drafting `monitoring.sh`, with a line-by-line explanation of how it works.
- **Documentation**: drafting this README, which I reviewed.
