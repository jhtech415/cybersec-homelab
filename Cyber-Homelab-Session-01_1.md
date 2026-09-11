---
title: "Cyber Homelab — Session 01: Building the Attack Lab"
tags: [homelab, proxmox, kali, metasploitable, nmap, networking, troubleshooting]
created: 2026-08-29
status: complete
---

# Cyber Homelab — Session 01: Building the Attack Lab

> **What this note is:** a full write-up of my first real lab session — building a virtual "attacker vs target" setup and running my first scan. Written so beginner-me can re-read it, understand *why* each step happened, and do it again from scratch. 
---

## The Big Picture (what I actually built)

I built a small, self-contained network of virtual computers running on one physical machine. Three virtual machines:

| VM ID | Name | Role | What it is |
|-------|------|------|------------|
| 100 | Kali | **Attacker** | A Linux built for security testing — comes loaded with hacking/analysis tools |
| 101 | pfSense | **Firewall/router** | Controls and filters network traffic (set up, not used yet this session) |
| 102 | Metasploitable | **Target** | A Linux left broken *on purpose* so I can safely practice attacking it |

All three run on **Proxmox** (the hypervisor — see glossary) at `https://10.0.0.50:8006`.

The most important design choice: the target lives on an **isolated network** so it can never reach the real internet. (See [Caging the Lab]

```
        [ My MacBook ] ---> browser + terminal
               |
        [ Proxmox host  10.0.0.50 ]
         |            |            |
      Kali (100)  pfSense (101)  Metasploitable (102)
         |________________________________|
                 vmbr1 = sealed lab network
                 (no wire to the internet)
```

---

## Glossary

Plain-English first, precise second.

- **VM (Virtual Machine):** a whole computer that exists as software instead of hardware. You can run several on one physical box.
- **Hypervisor:** the program that creates and runs VMs. Mine is **Proxmox**.
- **Proxmox:** the hypervisor software I use; I control it from a web page in my browser.
- **ISO:** a single file that acts like a CD/DVD — usually an *installer* you boot to install an operating system.
- **VMDK:** a file that *is* a virtual hard drive. Unlike an ISO, it's not an installer — the machine is already built inside it.
- **.gz / gzip:** a compression format. `.gz` = a file squeezed smaller (like a vacuum-sealed bag). Must be decompressed before use.
- **Terminal / CLI:** the text window where you type commands instead of clicking. CLI = Command-Line Interface.
- **SSH:** Secure Shell — logging into another computer's terminal over the network, encrypted.
- **SCP:** Secure Copy — copying a file to/from another computer over SSH.
- **root:** the all-powerful admin account on Linux. Can do anything.
- **sudo:** "do this one command as root." Asks for *your* password, not root's.
- **NIC:** Network Interface Card — a computer's network port (real or virtual).
- **Bridge (vmbr0, vmbr1):** a virtual network switch inside Proxmox. VMs plugged into the same bridge can talk to each other.
- **IP address:** a computer's address on a network, e.g. `192.168.50.10`.
- **Subnet / `/24` / netmask `255.255.255.0`:** defines the "neighborhood." Same first 3 numbers = same street; the last number = the house.
- **DHCP:** a service that hands out IP addresses automatically. No DHCP = you set the address by hand (**static IP**).
- **ping:** a tool that asks "are you there?" — sends tiny packets and waits for replies.
- **nmap:** the Network Mapper — the go-to tool for finding what's running on a target.
- **port:** a numbered "door" on a computer; each network service listens on its own port (e.g. web = 80).
- **service:** a program listening on a port (ftp, ssh, http, mysql...).
- **initramfs:** a tiny temporary system Linux uses early in boot; its job is to find and mount the real disk.
- **BusyBox:** a stripped-down toolkit with only a few commands — what the initramfs emergency shell uses.
- **fsck:** File System Check — Linux's "scan and repair a disk" tool.
- **SCSI vs IDE (disk bus):** two ways a virtual disk connects to the VM. SCSI is modern; IDE is old-but-universal. Old operating systems may only understand IDE.
- **Snapshot:** a saved "moment in time" of a VM you can roll back to.
- **Host key fingerprint / TOFU:** a computer's ID card, shown the first time you SSH in. TOFU = "Trust On First Use" — you verify once, trust after.
- **Attack surface:** the total number of ways into a system. Fewer open services = smaller attack surface.

---

## Part 1: Getting the Images Ready

### pfSense came as `.iso.gz` and Proxmox refused it
Proxmox's ISO uploader only accepts real `.iso`/`.img` files. pfSense downloads **compressed** (`.iso.gz`), so it had to be unzipped first.

**Mac fix (Terminal):**
```bash
cd ~/Downloads
gunzip pfSense-CE-<version>.iso.gz
```
- `cd ~/Downloads` = go into my Downloads folder (`~` = my home folder).
- `gunzip <file>` = decompress it; leaves a clean `.iso` behind.

**Verify it worked:**
```bash
ls -lh ~/Downloads
```
- `ls` = list files, `-l` = detailed, `-h` = human-readable sizes. Should show a `.iso` around 700 MB–1 GB.

> 💡 **Lesson:** always read the *full* file extension. `.iso.gz`, `.tar.gz`, `.img.gz` all have a wrapper to remove before the file is usable.

### Metasploitable came as a `.zip` — and it's NOT an installer
Kali and pfSense are `.iso` *installers*. **Metasploitable is a pre-built VM** shipped as a `.vmdk` (virtual disk) inside a `.zip`. Nothing to install — the broken machine already exists inside that disk. So instead of "boot installer," the job is "**import the disk**."

Unzip on Mac = just double-click. Inside was `Metasploitable.vmdk` (~1.8 GB).

---

## Part 2: Moving the File & Getting Into Proxmox

The import command runs *on the Proxmox host*, so the disk had to be copied there first.

**Copy the file (run on Mac):**
```bash
scp ~/Downloads/Metasploitable2-Linux/Metasploitable.vmdk root@10.0.0.50:/root/
```
- `scp` = secure copy.
- Left = the file on my Mac. Right = "log in as **root** on 10.0.0.50 and drop it in `/root/`."
- **Tip:** type `scp ` then *drag the file from Finder into Terminal* to auto-fill the exact path.

**First-connection prompt (host key / TOFU):**
First SSH/SCP to a new machine shows its **fingerprint** and asks to continue. Type the full word `yes`. This makes my Mac *remember* that machine; if the fingerprint ever changes later, SSH warns me — that's the defense against a **man-in-the-middle** attack.

**Log into Proxmox's command line:**
```bash
ssh root@10.0.0.ip
```
Prompt changes to `root@pve:~#` = I'm now typing on the Proxmox box.

**Confirm the file arrived:**
```bash
ls -lh /root/
```

---

## Part 3: Building the Metasploitable VM

### Create an empty VM shell (Proxmox web UI)
`Create VM` → General: name it, note the **VM ID** (mine = **102**) → OS: **"Do not use any media"** (no installer!) → accept defaults through the wizard → Memory `1024` MiB → **do NOT** start after creation.

### Import the real disk (Proxmox CLI)
```bash
qm importdisk 102 /root/Metasploitable.vmdk local-lvm
```
- `qm` = "QEMU Manage," Proxmox's VM control command.
- `102` = which VM. `local-lvm` = where to store the disk.
- Finishes as `unused0` = disk is *in* the VM but not plugged in yet.

### Plug it in and remove the throwaway disk (web UI)
1. Hardware → double-click **Unused Disk 0** → Bus = **SCSI** → Add.
2. The wizard forced a default 32 G disk earlier — remove it: select it → **Detach** → it becomes Unused → select → **Remove**.
   - **How I knew which to delete:** the real one was **8 G / disk-1**; the throwaway was **32 G / disk-0**. Size + disk number both agreed. *(This SCSI disk later had to become IDE — see [Troubleshooting](#the-initramfs--missing-disk-fight).)*

### Boot order
Options → Boot Order → put the Metasploitable disk **at the top and checked**, uncheck everything else.

> 💡 **Lesson:** a computer tries each boot device *in order* until one gives it a working OS. Wrong order = it boots nothing and hangs.

---

## Part 4: Caging the Lab (network isolation)

**The rule:** a deliberately-vulnerable machine must **never** reach the internet. If it can reach out, attackers can reach in — and it becomes a doorway into my real home network.

**How isolation works:** a bridge is a virtual switch. Whether it's "isolated" depends on one thing — **does it have a physical network port attached?**
- `vmbr0` → has my real Ethernet port → reaches the internet. **NOT safe for the target.**
- `vmbr1` → **Bridge ports = empty** → no wire out → sealed cage. VMs on it can talk to each other but nothing leaves.

**How I checked:** Datacenter → pve → System → Network → select `vmbr1` → **Edit** → confirmed **Bridge ports** was blank.

**Assigning it:** each VM → Hardware → Network Device (net0) → **Edit** → Bridge = **vmbr1**.

> ⚠️ Both Kali and the target must be on `vmbr1` to talk to each other. (I forgot this and it caused a "ping gets nothing" scare — see Troubleshooting.)

---

## Part 5: Snapshots (the safety habit)

A **snapshot** saves the entire state of a VM so I can roll back in ~10 seconds.

- Take one **before** any experiment. Break things fearlessly, then roll back.
- Keep one **golden `baseline-clean`** snapshot of the freshly-working machine forever. Never delete it.
- **Snapshots are NOT backups** — they live on the same disk; if the disk dies, they die too.
- Don't hoard them — they eat disk space and can slow a VM.

**Where:** select VM → Snapshots → Take Snapshot → name it clearly (`baseline-clean`, `pre-nmap`, etc.).

---

## Part 6: Booting & the Troubleshooting Fights

*( The story:)*

1. **Black screen** on the console → not broken. The text console goes blank when idle. **Click in the console, press Enter a few times.** It came back. *(Deleting/rebuilding to "fix" a blank screen is wrong — it isn't the cause.)*
2. **`(initramfs)` prompt** → boot got interrupted before the real OS loaded; dropped into an emergency shell.
3. **`fsck: not found`** → the emergency shell is BusyBox (tiny toolset); `fsck` isn't included.
4. **`ALERT! /dev/mapper/metasploitable-root does not exist`** → the real problem: the kernel couldn't *see* its own disk. Cause = **old 2008 kernel + modern VirtIO SCSI controller = missing driver.** Fix below.

### The fix that solved the boot
Change the disk from SCSI to **IDE** (a controller the ancient kernel understands):
1. Stop the VM (Shutdown ▾ → **Stop**).
2. Hardware → select the disk → **Detach** → becomes Unused.
3. Double-click Unused → change **Bus/Device** to **IDE** → Add.
4. Options → Boot Order → check **ide0**, drag to top, uncheck the rest.
5. Start → Console → it booted to the login prompt. ✅

> 💡 **Lesson:** when a VM boots but can't find its own disk, suspect **wrong controller type** — old OS + modern virtual hardware = missing driver.

### Login
Default creds: **`msfadmin` / `msfadmin`**.
> 💡 **Lesson:** default/weak credentials are one of the most common real-world ways in. This box ships wide open on purpose.

---

## Part 7: Networking the Machines

The lab bridge (`vmbr1`) has no DHCP, so no IPs are handed out automatically — I set them **by hand** (static). Same street `192.168.50.x`, different house numbers.

**On Metasploitable (target) — set `.10`:**
```bash
sudo ifconfig eth0 192.168.50.10 netmask 255.255.255.0
```
**On Kali (attacker) — set `.20`:**
```bash
sudo ip addr add 192.168.50.20/24 dev eth0
```
- `/24` and `255.255.255.0` mean the same thing: "everyone sharing the first three numbers is on my street."
- `sudo` asks for the **logged-in user's** password (on Kali that was my user account, not root).

**Verify each machine took its address:**
```bash
ip addr show eth0        # Kali
ifconfig eth0            # Metasploitable  (look for "inet addr:")
```

**Prove they can talk (from Kali):**
```bash
ping 192.168.50.10
```
- `ping` runs **forever** — stop it with **Ctrl+C**.
- Lines printing (`64 bytes from ...`) = success, they see each other. ✅
- Cursor sits blank then "100% packet loss" = packets sent, nothing answered → keep debugging.
- Instant `Network is unreachable` = wrong interface/route.

> ⚠️ **Reboots wipe these static IPs.** Next session I'll likely need to re-run them. (Making them permanent is a future task.)

---

## Part 8: The First Scan — nmap

**Run from Kali:**
```bash
nmap -sV 192.168.50.10
```
- `nmap` = network mapper. `-sV` = **s**ervice **V**ersion → identify what's running on each open port *and its version*.

**Reading the output** — four columns:
| Column | Meaning |
|--------|---------|
| PORT | the numbered door (e.g. 21/tcp) |
| STATE | `open` = something is listening/answering |
| SERVICE | nmap's guess at the service type |
| VERSION | exact software + version — **where attacks come from** |

The scan found **23 open ports** (a normal server shows 1–2). Notable ones:

- **1524 – bindshell / "Metasploitable root shell":** a root shell already listening on the network — instant admin, no password. Real-world equivalent = an attacker's backdoor.
- **21 – vsftpd 2.3.4:** famous backdoored version — a `:)` in the username opens a root shell. Lesson = **supply-chain compromise** (the software itself was tampered with).
- **139 / 445 – Samba (SMB):** file sharing with a command-injection hole. SMB flaws power huge real attacks (e.g. WannaCry).
- **512 / 513 / 514 – rexec / rlogin / rsh:** ancient "r-services" that trust *where* you connect from, not a password. Easily spoofed.
- **23 – telnet:** sends everything, including passwords, in **plaintext**. This is *why* SSH replaced it.
- Others (MySQL 3306, PostgreSQL 5432, VNC 5900, Tomcat 8180, UnrealIRCd 6667) mostly run default/blank passwords or have their own backdoors.

---

## Part 9: Attacker vs Defender Mindset

Same scan, two jobs:
- **Attacker (red team):** "which door do I go through?" → picks the weakest service.
- **Defender (blue team):** reads the *same list* as a to-do list → "why is telnet on? turn it off. Why 23 services? shut down what isn't needed → shrink the **attack surface**. A root shell on 1524? that box is compromised — isolate it."

Most real security work is that **defender** column. nmap is the first tool *both* sides open.

---

## Command Cheat Sheet

| Command | What it does |
|---------|--------------|
| `cd ~/Downloads` | Change into my Downloads folder |
| `gunzip file.gz` | Decompress a `.gz` file |
| `ls -lh` | List files with detailed, readable sizes |
| `scp file root@IP:/root/` | Copy a file to another machine over SSH |
| `ssh root@IP` | Log into another machine's terminal |
| `qm importdisk <id> <file> local-lvm` | Import a virtual disk into a Proxmox VM |
| `sudo ifconfig eth0 <ip> netmask <mask>` | Set a static IP (older style) |
| `sudo ip addr add <ip>/24 dev eth0` | Set a static IP (modern style) |
| `ip addr show eth0` / `ifconfig eth0` | Check an interface's IP |
| `ping <ip>` | Test if another machine is reachable (Ctrl+C to stop) |
| `nmap -sV <ip>` | Scan a target for open ports + service versions |

---

## Troubleshooting Log

| Symptom | Real cause | Fix |
|---------|-----------|-----|
| Proxmox won't accept pfSense image | It's `.iso.gz` (compressed) | `gunzip` it first |
| Console shows black screen | Idle console blanked; not broken | Click in it, press Enter |
| `(initramfs)` prompt on boot | Kernel couldn't mount its disk | See next row |
| `fsck: not found` | Emergency shell is BusyBox (limited) | Don't rely on fsck here |
| `ALERT! ...-root does not exist` | Old kernel can't see VirtIO **SCSI** disk (no driver) | Change disk bus **SCSI → IDE** |
| `ping` gets no replies | Duplicate IPs / VM on wrong bridge | Give unique IPs; put both NICs on `vmbr1` |
| noVNC "Failed to connect to server" | Browser console proxy glitch (not the VM) | Click off/on Console, or `Cmd+Shift+R`, or re-login |

---

## Key Concepts I Learned

- Compressed images must be decompressed before a hypervisor can use them.
- ISO = installer; VMDK = a ready-made disk you *import*.
- SSH/SCP move files and shells securely; the first-connection fingerprint is a real security control (TOFU / anti-MITM).
- Boot order matters; a computer tries devices in sequence.
- Network isolation = a bridge with no physical uplink. This is what makes a malware/target lab safe.
- Snapshots = fearless experimentation + fast recovery (but not backups).
- Old OS + modern virtual hardware = missing driver (the IDE/SCSI fix).
- Static IPs on an isolated net; `/24` = same subnet.
- nmap reveals attack surface; the *version* column is where vulnerabilities live.
- Every recon is read two ways — attacker and defender.

**The troubleshooting loop I keep reusing:** symptom → hypothesis → check evidence → fix → verify.

---

## What's Next

- [ ] Take/confirm `baseline-clean` snapshots on VM 100 and VM 102.
- [ ] Make the static IPs **persistent** (survive reboot).
- [ ] First hands-on exploit: **port 21 / vsftpd 2.3.4 backdoor** (clean, instant root, easy to understand).
- [ ] Bring **pfSense (101)** into the picture as the lab's router/firewall.
- [ ] Practice the **destroy-and-rebuild** drill: delete VM 102 and rebuild it from the `.vmdk` with no notes.

---


