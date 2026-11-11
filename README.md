# Born2beRoot

## Description

Born2beRoot is a system administration project. The goal is to build a small, secure Linux server inside a virtual machine, from the installer up to the last configuration file,
and to understand every choice made along the way.

What I built:

- A **Debian 13 "trixie" (stable)** server, installed from the netinst image in **VirtualBox**, with no graphical interface.
- An **encrypted disk** (LUKS) with **LVM** on top, using the partition layout from the bonus part.
- A hardened **SSH** service on port **4242**, with root login disabled.
- A **UFW** firewall that only allows port 4242.
- **AppArmor** active at boot.
- A strong **password policy** and a strict **sudo** configuration with logging.
- A bash script, `monitoring.sh`, that broadcasts system information to all terminals at boot and every 10 minutes.

The hostname of the machine is `dalmoman42`, and my user `dalmoman` belongs to the `sudo` and `user42` groups.

## Instructions

This repository only contains this `README.md` and `signature.txt`. The virtual machine itself is not part of the repository.

`signature.txt` holds the SHA-1 hash of the virtual disk (`.vdi`). It was generated on the host with:

```
sha1sum Born2beRoot.vdi
```

To check the project:

1. Open VirtualBox and start the `Born2beRoot` machine (do not start it before comparing the signature, because starting it changes the hash; work on a copy of the disk instead).
2. Type the disk encryption passphrase at boot, then log in as `dalmoman`.
3. Connect through SSH from the host. The VM uses NAT with port forwarding (host `2222` to guest `4242`):

```
ssh dalmoman@127.0.0.1 -p 2222
```

4. Useful checks:

```
lsblk
sudo aa-status
sudo ufw status
sudo systemctl status ssh
sudo crontab -l
sudo cat /var/log/sudo/sudo.log
```

## Project description

### Operating system choice

I chose **Debian**. It is the one I was most comfortable with for a first server: the documentation is large, the community is big, `apt` is simple to use,
and a stable release does not change under my feet. It is also the system the subject recommends for beginners.

### Design choices

**Partitioning.** The disk has a small unencrypted `/boot` partition (500M) and one big encrypted partition. Inside the encrypted partition, LVM gives separate logical volumes:

| Volume | Mount point | Size |
|---|---|---|
| root | `/` | 10G |
| swap | swap | 2.3G |
| home | `/home` | 5G |
| var | `/var` | 3G |
| srv | `/srv` | 3G |
| tmp | `/tmp` | 3G |
| var--log | `/var/log` | 4G |

Splitting the system like this means that a full `/var/log` or `/tmp` cannot fill the root volume and crash the machine. LVM also lets me resize volumes later without reinstalling.

**Security policies.**

- SSH listens on port 4242 and `PermitRootLogin` is set to `no`.
- UFW denies everything by default and only allows port 4242.
- Passwords expire after 30 days (`PASS_MAX_DAYS 30`), can be changed at most every 2 days (`PASS_MIN_DAYS 2`), and a warning appears 7 days before expiry (`PASS_WARN_AGE 7`).
- Password quality is enforced with `pam_pwquality`: at least 10 characters, one uppercase letter, one lowercase letter, one digit, no more than 3 identical consecutive characters, the username is not allowed inside the password, and at least 7 characters must differ from the old password (this last rule does not apply to root). The rules also apply to root.
- `sudo` is limited to 3 password attempts, shows a custom error message, requires a TTY, uses a restricted `secure_path`, and logs every command (input and output) in `/var/log/sudo/`.

**User management.** I use a normal user, `dalmoman`, who is a member of `sudo` and `user42`. Root cannot log in through SSH, so administration goes through `sudo`, which leaves a trace of what was done. For the evaluation I can create another user and add them to a group (for example `evaluating`).

**Services installed.**

- `openssh-server` for remote access.
- `ufw` for the firewall.
- `sudo` for privilege management.
- `libpam-pwquality` for password rules.
- `cron` to run the monitoring script.
- AppArmor (enabled by default on Debian).

**Monitoring script.** `/usr/local/bin/monitoring.sh` collects the information with `uname`, `grep`, `awk`, `free`, `df`, `top`, `who`, `lsblk`, `ss`, `hostname` and `ip`, then prints it to every terminal with `wall`. It shows: architecture and kernel, physical CPUs, virtual CPUs, RAM usage, disk usage, CPU load, last boot, whether LVM is used, active TCP connections, number of logged users, IPv4 and MAC address, and the number of `sudo` commands. It runs from the root crontab:

```
@reboot /usr/local/bin/monitoring.sh
*/10 * * * * /usr/local/bin/monitoring.sh
```

To stop it without touching the script, I just comment out or edit the line in `sudo crontab -e`.

### Comparisons

**Debian vs Rocky Linux**

| | Debian | Rocky Linux |
|---|---|---|
| Origin | Community project, independent | Free rebuild of Red Hat Enterprise Linux |
| Package manager | `apt` / `dpkg` | `dnf` / `rpm` |
| Release style | Stable release with very tested packages | Long enterprise support cycles |
| Default security module | AppArmor | SELinux |
| Advantages | Easy to learn, huge documentation and package list, lightweight | Very close to what companies run in production, strong security defaults |
| Disadvantages | Packages can be older than in other distributions | SELinux and `dnf` have a steeper learning curve for a beginner |

**AppArmor vs SELinux**

AppArmor controls programs through profiles tied to file paths. It is easier to read and to configure. SELinux labels every file and process and applies much stricter and finer rules, but it is harder to understand and to debug. SELinux is more powerful, AppArmor is simpler. Debian ships with AppArmor, so I kept it.

**UFW vs firewalld**

UFW ("Uncomplicated Firewall") is a simple front end for iptables/nftables, with short commands such as `ufw allow 4242`. firewalld works with zones and services and can change rules without restarting the firewall, which is better for complex networks, but it is heavier to configure. UFW is the default choice on Debian and fits this project well.

**VirtualBox vs UTM**

VirtualBox is free, open source, and works on Linux, Windows and Intel Macs. It is what the campus computers provide. UTM is built on QEMU and is the choice for Apple Silicon Macs, where VirtualBox does not run well. Since I work on a campus Linux machine, I used VirtualBox.

## Resources

**References**

- Debian documentation: https://www.debian.org/doc/
- Debian Administrator's Handbook: https://debian-handbook.info/
- Debian wiki, AppArmor: https://wiki.debian.org/AppArmor
- UFW on the Debian wiki: https://wiki.debian.org/Uncomplicated%20Firewall%20%28ufw%29
- `man sudoers`, `man sshd_config`, `man crontab`, `man pam_pwquality`, `man wall`
- LVM and LUKS: https://wiki.archlinux.org/title/LVM and https://wiki.archlinux.org/title/Dm-crypt
- VirtualBox manual: https://www.virtualbox.org/manual/

**How AI was used**

I used Claude (an AI assistant) as a tutor while doing this project. It was used for:

- Explaining the subject and the concepts (LVM, LUKS, AppArmor, UFW, SSH, sudo, cron, PAM).
- Guiding me step by step through the installation and the configuration, and helping me read error messages (for example a full disk, a bad port forwarding rule, and a wrong crontab line).
- Explaining the commands used in the monitoring script and helping me check its output.
- Helping me write this README from the setup I actually built.

I typed and applied all the configuration myself on my own virtual machine, and I tested each part (reboot, firewall rules, SSH, sudo log, cron). I did not use AI to bypass any part of the project, and I can explain every part of it.
