*This project has been created as part of the 42 curriculum by asaadane.*

# Born2beRoot — Linux System Administration & Virtualization

<p align="center">
  <img src="https://img.shields.io/badge/42_Cursus-Born2beRoot-000000?style=for-the-badge&logo=42" alt="42 Cursus Born2beRoot" />
  <img src="https://img.shields.io/badge/Debian-A81D33?style=for-the-badge&logo=debian&logoColor=white" alt="Debian" />
  <img src="https://img.shields.io/badge/Linux-FCC624?style=for-the-badge&logo=linux&logoColor=black" alt="Linux" />
  <img src="https://img.shields.io/badge/VirtualBox-183A61?style=for-the-badge&logo=virtualbox&logoColor=white" alt="VirtualBox" />
  <img src="https://img.shields.io/badge/Bash-4EAA25?style=for-the-badge&logo=gnubash&logoColor=white" alt="Bash Scripting" />
  <img src="https://img.shields.io/badge/MariaDB-003545?style=for-the-badge&logo=mariadb&logoColor=white" alt="MariaDB" />
</p>

<p align="center">
  <a href="#project-overview">Project Overview</a> •
  <a href="#virtualization--operating-system">Virtualization & OS</a> •
  <a href="#storage--partitioning-concepts">Storage & Partitioning</a> •
  <a href="#security-configuration--hardening">Security Hardening</a> •
  <a href="#bonus-service-mariadb">MariaDB Bonus</a> •
  <a href="#terminal-utility-screen">Screen Utility</a> •
  <a href="#monitoring-script--automation">Monitoring & Automation</a> •
  <a href="#peer-to-peer-evaluations">Peer Evaluations</a> •
  <a href="#verification--inspection-commands">Verification</a> •
  <a href="#author--license">Author & License</a>
</p>

---

## Project Overview

**Born2beRoot** is a foundational system administration project in the 42 Common Core curriculum. It introduces virtualized server installation, storage encryption, privilege delegation, network perimeter defense, and automated telemetry in a pure command-line environment.

Running strictly in headless multi-user mode (X.org, Wayland, or desktop environments are forbidden), the objective is to build and harden a minimal Linux server inside **Oracle VirtualBox** adhering to strict security baselines:
- **Encrypted Storage:** Partitioning storage with LUKS volume encryption and Logical Volume Management (LVM).
- **Network Perimeter:** Remapping SSH from the default port (author reported port 4242), disabling root remote access, and enforcing inbound packet filtering via UFW.
- **Identity & Privilege Governance:** Enforcing password aging and complexity via PAM (`pam_pwquality`), alongside strict `sudo` attempt restrictions, TTY enforcement, and I/O auditing.
- **Bonus Service (MariaDB):** Installing, securing, and maintaining a local MariaDB relational database service.
- **Terminal Multiplexing (Screen):** Utilizing `screen` for resilient multi-session command-line workflows.
- **System Telemetry Automation:** Engineering a custom Bash script ([`Monitoring.sh`](Monitoring.sh)) scheduled via `cron` to run every 10 minutes and broadcast server vitals across all active terminal sessions via `wall`.

---

## 42 Cursus Context & Learning Objectives

| Learning Dimension | Competencies Acquired |
| :--- | :--- |
| **Virtualization & Systems** | Hypervisor operations, host vs. guest abstraction, headless Unix lifecycle. |
| **Storage Architecture** | Master Boot Record (MBR) / GUID Partition Table (GPT), LUKS block encryption, dynamic LVM volume management concepts. |
| **Perimeter & Remote Defense** | Port remapping, remote root attack mitigation, stateful firewall rules, Mandatory Access Control (AppArmor). |
| **Authentication & Governance** | Linux-PAM configuration, `/etc/login.defs` password aging, complexity enforcement, `/etc/sudoers.d/` logging. |
| **Database Administration** | Systemd service management, relational database configuration, database security hardening, and local service configuration. |
| **Shell Scripting & Telemetry** | System metric extraction (`/proc`, `free`, `df`, `ss`, `top`), cron scheduling, inter-terminal broadcasting. |
| **Peer Defense & Communication** | Defending architectural choices, performing live reconfiguration under peer scrutiny. |

---

## Virtualization & Operating System

### VirtualBox Environment
The virtual machine runs inside **Oracle VirtualBox**, a Type-2 hypervisor that abstracts physical host resources into virtualized hardware:
- **Host vs. Guest Distinction:** The host system manages the physical CPU, memory, and physical network adapters. The guest virtual machine runs an independent kernel and operating system in an isolated hardware container.
- **Network Addressing:** As confirmed by the author, no static IP address was configured for the virtual machine.

### Operating System: Debian GNU/Linux
Debian was selected for this project based on the following technical considerations:
- **Stability & Minimal Footprint:** Debian provides a predictable, rock-solid operating environment with a minimal base netinst footprint.
- **Package Management:** Native `APT` (`dpkg`, `.deb` packages) provides clear dependency resolution and transparent service management.
- **Mandatory Access Control (MAC):** Debian integrates **AppArmor** by default, enforcing path-based security profiles on system binaries without the administrative overhead of complex multi-level labeling.

#### Technical Choices & Context
* **Debian vs. Rocky Linux:** Debian uses `APT` package management and path-based AppArmor; Rocky Linux (a downstream enterprise rebuild of RHEL) uses `DNF`/`RPM` and inode/label-based SELinux. Debian was chosen for its clean Unix conventions and lower base overhead.
* **AppArmor vs. SELinux:** AppArmor binds security profiles to file paths (e.g., restricting `/usr/sbin/sshd`), making profiles human-readable and straightforward to audit (`aa-status`). SELinux attaches security contexts (`user:role:type:level`) to every object and process, providing granular multi-level security at the cost of higher policy complexity.
* **UFW vs. firewalld:** UFW provides a simplified declarative CLI for `nftables`/`iptables` packet filtering. firewalld operates as a dynamic daemon managing network interfaces through abstract zones (`public`, `dmz`).
* **VirtualBox vs. UTM:** VirtualBox is a mature cross-platform Type-2 hypervisor for x86_64 architectures using `.vdi` disk files. UTM relies on QEMU and Apple's Hypervisor.framework for ARM64 virtualization on Apple Silicon using `.qcow2` images.

---

## Storage & Partitioning Concepts

### 1. Subject Storage Concepts & Requirements
The Born2beRoot subject outlines standard storage and security requirements:
* **Unencrypted Boot Partition (`/boot`):** In common Debian installations and standard Born2beRoot setups, `/boot` is kept unencrypted so the bootloader can load the kernel (`vmlinuz`) and initial RAM disk (`initramfs`) before cryptographic drivers and user-space unlocking mechanisms initialize.
* **LUKS Encrypted Block Storage:** The subject mandates block-level encryption using LUKS (Linux Unified Key Setup via `cryptsetup`). In typical installations, the encrypted block container is unlocked during system startup so that underlying volumes and filesystems can be mounted.
* **Logical Volume Management (LVM):** Decrypted block storage is partitioned using LVM (Physical Volumes, Volume Groups, Logical Volumes) to decouple filesystems from physical disk sector boundaries. Logical volumes can often be extended online while mounted (subject to filesystem capabilities, such as `ext4`), whereas volume reduction generally requires unmounting and filesystem-specific shrink steps.

### 2. Author's Implementation Experience
* The virtual disk was partitioned manually during the Debian installation with guidance from an external system administration article.
* Disk encryption and Logical Volume Management were configured according to project specifications.
* The author remembers completing the setup successfully, though specific boot-unlock hook details or passphrase configurations were not independently recorded.

### 3. Unrecorded Partition Details
Because the virtual machine disk file is stored locally outside Git per 42 submission rules and is offline:
* Exact partition device names (such as `/dev/sda1` or `/dev/sda5`) and exact partition sizes in gigabytes remain unrecorded in the repository.
* Specific volume names and mount-point allocations (such as whether optional bonus volumes like `/var`, `/srv`, `/tmp`, or `/var/log` were created on separate logical volumes) cannot be confirmed from repository evidence.

---

## Security Configuration & Hardening

### 1. SSH Hardening
Remote administration is handled via OpenSSH Server (`openssh-server`). Based on the author's reported configuration in accordance with subject requirements:
* **Custom Port (Port 4242):** Remapped from the default SSH port `22` (the subject mandates port `4242`; the author reported configuring this port).
* **Root Login Prohibition:** Direct root login via SSH was disabled (`PermitRootLogin no`).
* **Access Restrictions:** Access restrictions were applied so that remote sessions authenticate through unprivileged user accounts, requiring privilege escalation via `sudo`.

*(Note: Because the virtual machine is preserved offline and not accessible from this repository host, the guest VM's runtime SSH configuration has not been independently verified; the command below is provided for inspection inside the running VM).*

```bash
# Verify effective SSH configuration (to be run inside the guest VM)
sudo sshd -T | grep -E "port|permitrootlogin"
```

### 2. UFW Firewall
Network access is governed by **UFW** (Uncomplicated Firewall):
* **Subject Mandate:** The subject requires a default-deny policy for incoming traffic, allowing only authorized ports.
* **Author Configuration:** UFW was enabled and rules were added to allow SSH on the configured custom port (port 4242).
* **Startup Enforcement:** Configured to start on boot to enforce packet filtering.

```bash
# Inspect active firewall rules and policies (to be run inside the guest VM)
sudo ufw status verbose
```

### 3. Password Policy
Configured via system defaults and PAM (Pluggable Authentication Modules):
* **Password Aging (`/etc/login.defs`):** The subject mandates a maximum password lifetime of 30 days (`PASS_MAX_DAYS 30`), a minimum interval between changes of 2 days (`PASS_MIN_DAYS 2`), and an expiration warning period of 7 days (`PASS_WARN_AGE 7`).
* **Password Complexity (`pam_pwquality`):** The subject mandates a minimum length of 10 characters containing uppercase, lowercase, and numeric characters, no more than 3 consecutive identical characters, username exclusion, and a 7-character difference from previous passwords (enforced for root as well).
* **Author Configuration:** Minimum password length, expiration, complexity requirements, warning period, and history restrictions were configured in system and PAM configurations.
* **Scope Note:** PAM configuration applies to authentication stacks invoking `pam_pwquality.so`; exact parameter syntax depends on whether rules are placed in `/etc/login.defs`, `/etc/security/pwquality.conf`, or `/etc/pam.d/common-password`.

```bash
# Inspect aging and PAM complexity rules (to be run inside the guest VM)
grep -E "^PASS_MAX_DAYS|^PASS_MIN_DAYS|^PASS_WARN_AGE" /etc/login.defs
grep -E "pam_pwquality.so" /etc/pam.d/common-password
```

### 4. Sudo Governance & Audit Logging
Administrative commands are executed through `sudo`, configured via a custom drop-in file in `/etc/sudoers.d/`:
* **Author Configuration:** Sudo group permissions were configured, authentication attempts were restricted to 3 tries, custom failure messaging was created, and sudo logging was configured under `/etc/sudoers.d/`.
* **Subject Requirements:** Authentication attempts limited to 3 tries (`passwd_tries=3`), custom warning message on failure (`badpass_message`), TTY enforcement (`requiretty`), and restricted executable lookup path (`secure_path`).
* **Input/Output Logging Mechanics:** The subject requires logging all sudo actions (inputs and outputs) in `/var/log/sudo/`. Note that in `sudoers`, setting `iolog_dir=/var/log/sudo` specifies the destination directory, while `log_input` and `log_output` must be enabled to record session I/O.

```bash
# Verify active sudo settings (to be run inside the guest VM)
sudo sudo -V | grep -E "Authentication attempts|Bad password message|requiretty"
```

---

## Bonus Service: MariaDB

As part of the project's bonus criteria, **MariaDB** was installed and configured as a local relational database management system.

### Rationale & Confirmed Implementation
* **System Role:** Operates as a background relational database daemon (`mariadb.service`) accessible locally on the virtual machine.
* **Author Experience:** The author confirmed installing the MariaDB server package, enabling it to start automatically at boot, creating a database and user, securing the installation, and testing local connectivity.

### Service Verification Commands
```bash
# Inspect MariaDB daemon status (to be run inside the guest VM)
sudo systemctl status mariadb

# Verify database service responsiveness without exposing credentials
sudo mariadb-admin ping

# Verify listening addresses and sockets
sudo ss -tunlp | grep -E "3306|mariadb|mysqld"
```

---

## Terminal Utility: Screen

In addition to base system packages, **Screen** was installed for command-line productivity in a headless environment.

### Accurate Technical Role
* **Terminal Multiplexer:** `screen` is **not a background daemon or network service**. It is a command-line terminal multiplexer that allows a single terminal connection to manage multiple pseudo-terminal sessions.
* **Session Persistence:** When administering a remote server over SSH, sudden network disconnects terminate running foreground processes. Running tasks inside a `screen` session decouples execution from the active SSH connection, allowing processes to continue uninterrupted.
* **Basic Operational Commands:**
  - `screen -S <name>`: Create a new named session.
  - `Ctrl + A`, then `D`: Detach from the session while processes continue running.
  - `screen -ls`: List active screen sessions.
  - `screen -r <name>`: Reattach to an existing session.

---

## Monitoring Script & Automation

### Telemetry Script (`Monitoring.sh`)
The repository contains a custom Bash monitoring script, [`Monitoring.sh`](Monitoring.sh), designed to collect and format real-time system metrics.

Example of the script's broadcast format (illustrative example based on subject specifications):
```text
Broadcast message from root@asaadane42 (tty1) (Sun Mar 12 18:20:00 2023):

	#Architecture: Linux asaadane42 5.10.0-19-amd64 #1 SMP Debian x86_64 GNU/Linux
	#CPU physical: 1
	#vCPU: 1
	#Memory Usage: 145/987MB (14.69%)
	#Disk Usage: 1530/8Gb (18%)
	#CPU load: 1.2%
	#Last boot: 2023-03-12 17:40
	#LVM use: yes
	#Connections TCP: 1 ESTABLISHED
	#User log: 1
	#Network: IP 10.0.2.15 (08:00:27:51:9b:a5)
	#Sudo: 12 cmd
```

### Telemetry Categories vs. Output Metrics

The script produces **12 output metrics** across **12 lines** in the `wall` broadcast, grouped into **10 broad functional categories** evaluated during the defense:

| # | Functional Category | Metric Line(s) in `wall` | Command / Mechanism in Script | Technical Behavior & Notes |
| :-: | :--- | :--- | :--- | :--- |
| **1** | **Architecture & Kernel** | `#Architecture` | `arc=$(uname -a)` | Gathers kernel name, hostname, kernel release, build timestamp, and operating architecture. |
| **2** | **Processor Information** | `#CPU physical`<br>`#vCPU` | `pcpu=$(grep "physical id" /proc/cpuinfo \| wc -l)`<br>`vcpu=$(grep "^processor" /proc/cpuinfo \| wc -l)` | `vcpu` counts virtual processors. `pcpu` counts occurrences of `"physical id"` lines in `/proc/cpuinfo`; without filtering for unique values (e.g., via `sort -u`), this counts socket record occurrences rather than necessarily distinct physical sockets. |
| **3** | **Memory & Storage Usage** | `#Memory Usage`<br>`#Disk Usage` | `free -m` and `free \| awk`<br>`df -BG` and `df -BM` (excluding `/boot$`) | `#Memory Usage` extracts total and occupied RAM from `free -m` and computes utilization percentage. `#Disk Usage` aggregates used and total block storage across mounted `/dev/` filesystems (excluding `/boot`). |
| **4** | **Processor Load** | `#CPU load` | `top -bn1 \| grep '^%Cpu' \| awk` | Samples instantaneous CPU utilization combining user (`%us`) and system (`%sy`) loads. |
| **5** | **Last Boot Time** | `#Last boot` | `who -b \| awk` | Extracts system boot date and time from system logs. |
| **6** | **LVM Operational Status** | `#LVM use` | `lsblk \| grep "lvm"` check | Returns `yes` if active LVM partitions are detected by `lsblk`; `no` otherwise. |
| **7** | **TCP Connections** | `#Connections TCP` | `ss -neopt state established \| wc -l` | Counts established TCP sockets (including the header row unless `-H` is used). Note: general socket inspection tools like `ss -ta` list TCP sockets across various states (LISTEN, SYN-SENT, ESTABLISHED, etc.) rather than exclusively established connections. |
| **8** | **Logged-In Users** | `#User log` | `users \| wc -w` | Counts active login-session entries across terminal devices, rather than necessarily distinct unique users. |
| **9** | **Network Addressing** | `#Network` | `hostname -I` and `ip link show \| grep "ether"` | Discovers the guest VM's IPv4 address and physical MAC address. |
| **10** | **Sudo Command Auditing** | `#Sudo` | `journalctl _COMM=sudo \| grep COMMAND \| wc -l` | Audits executed sudo commands recorded in the systemd journal. Requires root privileges to read the system journal. |

### Broadcasting via `wall`
The formatted output is broadcast across all open terminal devices (`/dev/pts/*` and `/dev/tty*`) using `wall`:
```bash
wall "	#Architecture: $arc
	#CPU physical: $pcpu
	#vCPU: $vcpu
	...
	#Sudo: $cmds cmd"
```

### Cron Scheduling & Service Interruption
The script is scheduled in root's crontab to execute every 10 minutes:
```text
# Example crontab entry (using placeholder path):
*/10 * * * * /bin/bash /path/to/Monitoring.sh
```

*(Note: While the 10-minute interval is confirmed, the exact filesystem path where the script was installed inside the virtual machine is unrecorded in the repository; `/path/to/` represents this local VM path).*

> [!IMPORTANT]
> **Cron Service Interruption During Peer Review:**
> During the 42 evaluation defense, you must demonstrate how to interrupt the broadcast without modifying the script code:
> ```bash
> sudo systemctl stop cron
> ```
> *Operational Note:* Stopping the cron service halts **all** cron-scheduled jobs across the system, not solely the monitoring broadcast. Once the defense verification is complete, the daemon must be restarted:
> ```bash
> sudo systemctl start cron
> ```

---

## Peer-to-Peer Evaluations

### 42 Evaluation Model
Evaluation in the 42 curriculum is conducted through peer-to-peer (P2P) reviews. Each student must explain their architectural choices, demonstrate practical commands live on the virtual machine, and submit their work to peer scrutiny.

### Evaluation Experience
* **Defending the Project:** The virtual machine was defended in a single evaluation session before **three independent student peer evaluators**.
* **Defense Score:** The evaluation was validated with a historical score of **125/100**, with all three peer evaluations recording 125% and full marks across all evaluated categories (4/4).
* **Evaluating Others:** Participated in the P2P evaluation exchange by evaluating three other students on multiple occasions, inspecting their VM configurations, partition structures, firewall rules, and security policies.

<p align="center">
  <img src="assets/validation/historical-evaluation-125.png" alt="Historical Peer Evaluation Evidence (125/100)" width="850" />
  <br />
  <em>Historical 42 Intra evaluation record showing the validated 125/100 score across three independent peer evaluations (anonymized for personal data privacy).</em>
</p>

### Live Defense Tasks
During the evaluation, the student must demonstrate practical administrative tasks directly in the VM:
1. **Changing the VM Hostname:**
   ```bash
   sudo hostnamectl set-hostname <new_hostname>
   sudo sed -i "s/old_hostname/<new_hostname>/g" /etc/hosts
   sudo hostnamectl status
   ```
2. **Creating a New User & Group Assignment:**
   ```bash
   sudo adduser <new_username>
   sudo addgroup user42
   sudo usermod -aG user42 <new_username>
   id <new_username>
   ```
3. **Interrupting the Monitoring Broadcast:** Suspending the cron daemon via `sudo systemctl stop cron` without altering the script code.

---

## Verification & Inspection Commands

The following non-destructive commands can be executed directly inside the Debian virtual machine terminal (not on the host) to inspect the environment:

```bash
# -----------------------------------------------------------------------------
# 1. System Identity & Resource Allocation (Run inside guest VM)
# -----------------------------------------------------------------------------
hostnamectl                               # Configured hostname and OS release
uname -a                                  # Kernel release and hardware architecture
lscpu | grep -E "Model name|CPU\(s\):"   # Virtual CPU count
free -h                                   # Allocated RAM capacity and utilization

# -----------------------------------------------------------------------------
# 2. Storage, LUKS Encryption & LVM State (Run inside guest VM)
# -----------------------------------------------------------------------------
lsblk -f                                  # Block device hierarchy, LUKS containers, mount points
sudo pvs                                  # LVM Physical Volumes
sudo vgs                                  # LVM Volume Groups
sudo lvs                                  # LVM Logical Volumes

# -----------------------------------------------------------------------------
# 3. Network, SSH & Firewall Status (Run inside guest VM)
# -----------------------------------------------------------------------------
sudo ss -tunlp                            # Active listening sockets
sudo sshd -T | grep -E "port|permitrootlogin"  # Effective OpenSSH runtime settings
sudo ufw status verbose                   # UFW firewall status and active rules

# -----------------------------------------------------------------------------
# 4. Security, Sudo & Password Policies (Run inside guest VM)
# -----------------------------------------------------------------------------
grep -E "^PASS_MAX_DAYS|^PASS_MIN_DAYS|^PASS_WARN_AGE" /etc/login.defs
grep -E "pam_pwquality.so" /etc/pam.d/common-password
sudo sudo -V | grep -E "Authentication attempts|Bad password message|requiretty"
sudo aa-status                            # AppArmor profile enforcement status

# -----------------------------------------------------------------------------
# 5. MariaDB Bonus Service Verification (Run inside guest VM)
# -----------------------------------------------------------------------------
sudo systemctl status mariadb             # MariaDB daemon status
sudo mariadb-admin ping                   # Database responsiveness check

# -----------------------------------------------------------------------------
# 6. Monitoring & Cron Automation (Run inside guest VM)
# -----------------------------------------------------------------------------
sudo /bin/bash /path/to/Monitoring.sh      # Test manual execution (replace with actual VM path)
sudo crontab -u root -l                   # Inspect root's scheduled monitoring jobs
```

---

## Evidence & Verification Matrix

| Technical Component | Classification | Evidence Source | Status & Notes |
| :--- | :--- | :--- | :--- |
| **Evaluation Score (125/100)** | Verified Repository Evidence | `born2beroot-125.png` / `assets/validation/` | Official 42 Intra screenshot confirms 125/100 across 3 evaluations. |
| **Monitoring Script Logic** | Verified Repository Evidence | `Monitoring.sh` | 30-line Bash script gathering telemetry and broadcasting via `wall`. |
| **Git Repository State** | Verified Repository Evidence | Git commit history and working tree | Repository files, commit history, and scripts verified intact. |
| **Debian & VirtualBox Choice** | Personal Experience | Author recollection | Debian minimal install hosted inside Oracle VirtualBox. |
| **Network IP Configuration** | Personal Experience | Author recollection | No static IP address was configured in the virtual machine. |
| **MariaDB Bonus Deployment** | Personal Experience | Author recollection | Installed MariaDB server, enabled auto-start, created database/user, secured installation, and tested local connectivity. |
| **Screen Utility Usage** | Personal Experience | Author recollection | Installed as a terminal multiplexer for headless session management. |
| **Exact Partition Sizes** | Unrecorded / Unavailable | None (VM disk not in Git) | Exact byte sizes unrecorded; conceptual LUKS + LVM architecture documented. |
| **Virtual Disk Signature** | Subject Requirement | `signature.txt` | Chapter VIII requires generating `signature.txt` containing the SHA-1 hash of the virtual disk. No signature file is preserved in this repository, as virtual machine disks are excluded from Git submission. |

---

## Known Limitations & Operational Considerations

1. **Network Addressing:** No static IP address was configured in the virtual machine. In environments where dynamic IP addressing is used, the machine's IP address can change across reboots depending on the hypervisor and network environment.
2. **Monitoring Script Journal Access:** In `Monitoring.sh`, `journalctl _COMM=sudo` requires systemd journal access, meaning accurate sudo command counts require executing the script with root privileges (as configured in root's crontab).
3. **Virtual Disk SHA-1 Signature:** Chapter VIII of the subject mandates submitting a `signature.txt` file containing the SHA-1 hash of the virtual machine disk (`.vdi`). Because booting the guest OS alters disk block states, this hash must be computed while the VM is completely powered down. No historical signature file was committed to this repository.

---

## Learning Outcomes

Completing Born2beRoot provided hands-on, foundational experience in system administration, security hardening, and systems operations:
* **Operating Systems Fundamentals:** Gained practical understanding of headless Unix environments, the Linux boot process (GRUB, initrd, systemd targets), and system telemetry.
* **Storage & Cryptographic Architectures:** Mastered full-disk encryption with LUKS and flexible partition management through LVM logical volumes.
* **Network & Defense in Depth:** Learned how to reduce attack surfaces by eliminating unused services, altering default ports, disabling root remote access, and enforcing default-deny firewall policies.
* **Security & Compliance Auditing:** Configured Linux-PAM modules for strict password aging/complexity and audited administrative actions using sudo I/O logging.
* **Database & Utility Management:** Administered MariaDB as a secure local service and adopted terminal multiplexing with Screen for headless productivity.
* **Automated Operations:** Wrote maintainable shell scripts to parse `/proc` and system utilities, automating alerts across multi-user environments.

---

## Author & License

### Author
* **Ayman Saadane** ([@Shifuuu31](https://github.com/Shifuuu31))
* 42 Intra: `asaadane`

### License
This repository does not currently contain a LICENSE file. All rights are reserved by the author. Academic evaluation materials and subject guidelines are property of 42 Network. If you wish to use or reference this work for educational purposes, please credit the author.
