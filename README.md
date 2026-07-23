# AdmKit

A set of bash utilities currently covering three areas of Linux system administration: user/group lifecycle management, file integrity verification, and system diagnostics. I created this repo to eliminate the manual labor of chaining together multiple commands for tasks that come up constantly in sysadmin and support roles. Another reason for this repo is that there weren't any simple scripts online for these tasks.

## Tools

- [Tool 1: ugm (User-Group Manager)](#tool-1-user-group-manager-ugmsh)
- [Tool 2: vein (Verify and Integrity)](#tool-2-veinsh-verify-and-integrity)
- [Tool 3: srm (System Resource Monitor)](#tool-3-srmsh-system-resource-monitor)

---

## Tool 1: ugm (User-Group Manager)

`ugm` is a wrapper script around standard Linux account utilities (`useradd`, `groupadd`, `userdel`). I made it specifically to solve the hidden dangers of ID recycling, interactive mistakes, and bulk provisioning failures. The features are as follows:

### 1. The ID Recycling Problem (The Ledger System)

When a user or group is deleted, their ID returns to the available system pool. If a new user/group is later assigned that ID, they instantly inherit ownership of any orphaned files left behind by the previous owner — a latent security vulnerability.

Scanning the entire filesystem for orphaned files on every deletion is slow and I/O intensive. Instead, `ugm` implements an append-only ledger at `/var/lib/ugm/deleted_ids.db`. During any provisioning event, the ledger is queried. If a UID or GID is marked as historical, the tool explicitly blocks its assignment, permanently preventing ID recycling.

### 2. Safe Offboarding Sequence (`-d` / `--delete`)

Running `userdel -r` on an account with active processes will fail. Similarly, running `groupdel` on a group that acts as the primary group for existing users leaves those users orphaned and breaks their permissions.

- The `-d` (delete user) flag executes a controlled offboarding sequence: escalating from `SIGTERM` to `SIGKILL` for all active user sessions, purging crontabs, and optionally archiving the home directory before deletion is authorized.
- **Group migration:** when deleting a group (`-gd`), the script scans `/etc/passwd`. If the targeted group is acting as a primary group for any users, the script halts deletion and forces the admin to interactively migrate those users to a new primary group before deletion can proceed.

### 3. Bulk Provisioning & "Skip-and-Continue"

Standard `set -e` bash scripts will crash mid-import if a single CSV row is malformed, leaving the admin to figure out who was provisioned and who wasn't.

The `--bulk` import uses a two-pass architecture. Pass one is a strict validation dry-run that checks bounds (`UID_MIN`/`GID_MIN`), duplicate IDs, and ledger collisions. Invalid rows are skipped and logged; valid rows continue. One bad row will never crash the batch.

### 4. Secure Password Handling

To avoid printing generated passwords to stdout (which leaks credentials into terminal scrollback logs), the script writes temporary bootstrap passwords to a locked-down (`600`) directory. It dynamically detects if cron is installed and configures a system task (falling back to a dynamically generated systemd timer) to automatically shred these files after 24 hours. Provisioned users are forced to reset their password on first login (`chage -d 0`).

### Interface & Usage

The tool operates headless for automation, or via a TTY-driven interactive menu. The interactive menu supports sequential multi-select (e.g., typing `4,7,12` to sequentially List Users, List Groups, and Exit without relaunching the script).

```bash
# Provisioning & Safe Offboarding
sudo ./ugm --create johndoe
sudo ./ugm --delete johndoe

# Group Management & Migration
sudo ./ugm --group-create dev_team
sudo ./ugm --group-delete dev_team --migrate-to legacy_devs

# Panic Button (Kill active sessions & lock account)
sudo ./ugm --kick compromised_user

# Skip-and-Continue Bulk Import
sudo ./ugm --bulk /path/to/users.csv
```

The interactive menus also eliminate the friction of group building: when creating a new group, the script automatically lists active system users and prompts you to add existing users into the new group in one seamless workflow.

---

## Tool 2: vein (Verify and Integrity)

Verifying downloaded ISOs and binaries manually is a tedious chore. You have to look up the developer's public key, run `gpg --recv-keys`, run `gpg --verify`, parse the output, and finally run `sha256sum -c`. Because it requires chaining four or five commands, most admins just skip it.

`vein` is a wrapper script that automates this entire process into a single command. You pass it the files you downloaded, and it figures out the rest. While automating the manual labor, it also enforces a few strict security checks that raw GPG doesn't do by default.

### Why use this over manual GPG commands?

**1. No more remembering flags:**
Whether a vendor distributes checksums as `.sha256` sidecar files, `SHA256SUMS` manifests, or detached `.sig` files, `vein` auto-detects the format. Just pass the files to the script and it routes them to the correct validation tool automatically.

**2. Strict trust check:**
If you manually run `gpg --verify` on a file signed by an attacker's key, GPG will just say "Good Signature," because the math is valid. `vein` intercepts this: after validating the signature, it extracts the signer's fingerprint and cross-references it against a local `known_fingerprints.conf` file. If the fingerprint isn't on your approved list, the script throws a warning and halts.

**3. Keeping your keyring clean:**
Running `gpg --recv-keys` permanently saves untrusted, unverified keys from public keyservers into your personal `~/.gnupg` keyring. To prevent host pollution, `vein` performs all downloads and verifications inside a temporary, ephemeral keyring (`mktemp -d`). Once verification is complete, the temporary keyring is destroyed.

**4. Automation ready:**
Instead of just printing a generic pass/fail, it translates errors into 11 specific exit codes. This makes it easy to drop into a bash provisioning script to automatically halt a deployment if a downloaded binary is corrupted or tampered with.

### Interface & Usage

`vein` is entirely non-interactive. You just pass it the files you downloaded:

```bash
# 1. All-in-One Flow (Validates Signature -> Checks Trust -> Hashes ISO)
./vein SHA256SUMS.gpg SHA256SUMS ubuntu-22.04.3-desktop-amd64.iso

# 2. Batch Directory Verification (Checks an entire folder against a manifest)
./vein SHA256SUMS ./downloads/

# 3. Sidecar File Verification
./vein firmware-update.bin.sha256

# 4. Direct Hash Comparison
./vein 843938DF228D22F7B3742BC0D94AA3F0EFE21092 target-binary.tar.gz

# 5. Overriding the weak-hash guard for legacy systems (MD5/SHA1)
./vein legacy-app.md5 --allow-weak-hashes
```

---

## Tool 3: srm (System Resource Monitor)

`srm` is a read-only diagnostic script used to get a quick snapshot of a system's health. I built this for situations where you log into a server that doesn't have monitoring agents (like Prometheus or Datadog) installed, and you need to see what's going on without manually running diagnostic commands and parsing their output.

The script organizes standard Linux diagnostic commands into a single report, but uses a few specific approaches to avoid common pitfalls:

### 1. Handling "Loaded but Idle" States

Sometimes a system is unresponsive but standard CPU/RAM usage looks normal. Instead of just listing processes, the script checks for specific failure states. It uses `ps` to filter for D-state (uninterruptible sleep) processes to spot stuck I/O, and it samples `/proc/interrupts` twice over a two-second window to calculate deltas and catch hardware interrupt storms. It also groups processes by name to spot cron jobs that are hanging and stacking up over time.

### 2. Safe Inode Profiling

A disk can show plenty of free space and still refuse to create new files if inodes are exhausted. The script checks both and flags this specifically when it happens.

### 3. Filesystem-Specific Checks

ext4, XFS, Btrfs, and ZFS each expose health differently. The script parses `/proc/mounts` first, then runs the relevant tool for that specific filesystem. For example, it runs `zpool status` for ZFS mounts, `btrfs device stats` for Btrfs, and `tune2fs` to check overdue fsck counts on ext4.

### 4. Correlating Error Logs

Reading through `dmesg` during an outage can be noisy. The script pulls recent `err` through `emerg` level kernel events via `journalctl` (or `dmesg`) and parses them against common hardware failure signatures like `ata`, `scsi`, or `xfs`. If it finds a match, it flags it in the output so you know to cross-reference it with the hardware section of the report.

### Interface & Usage

The script is non-interactive. You can run the full report, or pass flags to only run specific sections if you want to limit execution time.

```bash
# Full default report (Summary, CPU, RAM, Disk, Hardware, Network, Logs)
sudo ./srm --full

# Targeted report
sudo ./srm -c   # CPU diagnostics
sudo ./srm -m   # Memory diagnostics

# Targeted infrastructure checks
sudo ./srm -d   # Disk, inode, and filesystem health
sudo ./srm -w   # Hardware health (RAID/SMART/Sensors)
sudo ./srm -n   # Network snapshot (Ports & connections)
```
