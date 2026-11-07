# HA K3s + Argo CD on Oracle VirtualBox — from empty PC to GitOps

> **Based on:** Christian Lempa — *"Using \*HA\* Kubernetes at home, was never so simple!"*
> <https://www.youtube.com/watch?v=UdjhFLV1yt8> and his k3s cheat-sheet from the same period.
>
> **The video's design:** an external **MariaDB** datastore, an **NGINX** TCP load balancer, **two K3s
> server nodes** (installed with `--node-taint` and `--tls-san`) and **agent (worker) nodes** that join through the load balancer.
>
> **What this guide adds:** it runs that design as **Oracle VirtualBox** VMs running **Kali Linux** on a Windows 11 host and installs **Argo CD** on top for GitOps.
>
> **Written for:** Windows 11 host · VirtualBox **7.1.x** · **Kali Linux** (rolling) on every VM: a minimal, headless install for the cluster nodes plus one Kali desktop workstation · K3s *stable* channel · Argo CD *stable*.
>
> **Kali as a node OS:** Kali is a rolling distro based on Debian testing. K3s runs on it the same way it runs on Debian, but Kali is not one of the OSes K3s is tested on, and a rolling `apt full-upgrade` can change packages without warning. Pin `INSTALL_K3S_VERSION` (Step 9), and take a snapshot before you upgrade a node.

---

## Table of contents

0. [Architecture, IP plan, and order of operations](#0-architecture)
1. [Windows host: install the tools](#step-1--windows-host-tools-path-ssh-key)
2. [Windows host: host-only network and DHCP pool](#step-2--windows-host-host-only-network--dhcp-pool)
3. [Windows host: create the template VM and install Kali](#step-3--windows-host-create-the-template-vm-and-install-kali)
4. [kali-template: prepare the golden image](#step-4--kali-template-prepare-the-golden-image)
5. [Windows host: clone the 7 cluster VMs](#step-5--windows-host-clone-the-7-cluster-vms)
6. [Console or Windows: give each clone its identity](#step-6--console-or-windows-give-each-clone-its-identity)
7. [k3s-db: MariaDB datastore and secrets](#step-7--k3s-db-mariadb-datastore--secrets)
8. [k3s-lb: NGINX TCP load balancer](#step-8--k3s-lb-nginx-tcp-load-balancer)
9. [k3s-srv-1: first K3s server](#step-9--k3s-srv-1-first-k3s-server)
10. [k3s-srv-2: second K3s server](#step-10--k3s-srv-2-second-k3s-server)
11. [k3s-agent-1/2/3: join the workers](#step-11--k3s-agent-123-join-the-workers)
12. [Windows host: kubectl through the load balancer](#step-12--windows-host-kubectl-through-the-load-balancer)
13. [Windows host: test HA failover](#step-13--windows-host-test-ha-failover)
14. [Windows host: install Argo CD](#step-14--windows-host-install-argo-cd)
15. [Windows host: expose Argo CD through Traefik](#step-15--windows-host-expose-argo-cd-through-traefik)
16. [Windows host: Argo CD CLI login and admin password](#step-16--windows-host-argo-cd-cli-login--admin-password)
17. [Windows host: first Argo CD app (guestbook)](#step-17--windows-host-first-argo-cd-app-guestbook)
18. [Windows host and GitHub: your own GitOps repo (app-of-apps)](#step-18--windows-host--github-your-own-gitops-repo-app-of-apps)
19. [Day-2: start/stop, backups, upgrades](#step-19--day-2-startstop-backups-upgrades)
- [Appendix A: embedded etcd variant (no external DB)](#appendix-a--variant-embedded-etcd-no-external-db)
- [Appendix B: make the load balancer HA (keepalived)](#appendix-b--make-the-load-balancer-ha-keepalived)
- [Appendix C: troubleshooting](#appendix-c--troubleshooting)
- [Appendix D: reset and destroy](#appendix-d--reset--destroy)
- [Appendix E: Kali Linux workstation VM](#appendix-e--kali-linux-workstation-vm)
- [What ended up on each VM](#what-ended-up-on-each-vm)
- [Sources](#sources)

---

## 0. Architecture

### 0.1 What you will build

```mermaid
flowchart TB
    subgraph HOST["Windows 11 Pro host — Oracle VirtualBox 7.1 — every VM runs Kali Linux"]
        YOU["You<br/>PowerShell: kubectl · argocd · ssh<br/>Browser<br/>192.168.99.1"]
        subgraph HON["Host-only network 192.168.99.0/24 — NIC 2 (eth1) of every VM"]
            LB["k3s-lb · 192.168.99.10<br/>NGINX L4 load balancer<br/>:6443 · :80 · :443"]
            subgraph CP["Control plane — tainted CriticalAddonsOnly (no app pods)"]
                S1["k3s-srv-1 · 192.168.99.21<br/>k3s server"]
                S2["k3s-srv-2 · 192.168.99.22<br/>k3s server"]
            end
            DB[("k3s-db · 192.168.99.11<br/>MariaDB :3306<br/>cluster datastore")]
            subgraph WK["Workers — join the API through k3s-lb:6443"]
                A1["k3s-agent-1 · 192.168.99.31"]
                A2["k3s-agent-2 · 192.168.99.32"]
                A3["k3s-agent-3 · 192.168.99.33"]
                PODS["Pods: Traefik ingress · Argo CD · your apps"]
            end
            KALI["kali · 192.168.99.50<br/>Kali desktop workstation<br/>kali.home.lab"]
        end
    end
    NAT{{"VirtualBox NAT — NIC 1 (eth0)<br/>Internet: apt · container images · GitHub"}}
    GH[("GitHub<br/>homelab-gitops repo")]

    YOU -->|"kubectl → :6443"| LB
    YOU -->|"https://argocd.home.lab → :443"| LB
    KALI -.->|"kubectl :6443 · https :443"| LB
    LB -->|":6443"| S1
    LB -->|":6443"| S2
    S1 -->|"datastore :3306"| DB
    S2 -->|"datastore :3306"| DB
    LB -->|":80 / :443 → Traefik"| WK
    PODS -.->|"Argo CD pulls manifests"| NAT
    NAT -.-> GH
```

The same picture in plain text, for viewers that don't render Mermaid:

```text
=================  WINDOWS 11 PRO HOST  -  Oracle VirtualBox 7.1  =================

  [ YOU ]  PowerShell (kubectl, argocd, ssh)  +  Browser
           host-only adapter: 192.168.99.1
  [ KALI ] Kali desktop workstation     kali.home.lab   192.168.99.50
           same paths as YOU: kubectl -> k3s-lb:6443, browser -> k3s-lb:443
                                   |
          kubectl  :6443           |          https :443  (argocd.home.lab)
                                   v
                     +----------------------------+
                     |  k3s-lb      192.168.99.10 |
                     |  NGINX  L4 load balancer   |
                     |  :6443     :80     :443    |
                     +-------------+--------------+
                     :6443         |         :80 / :443
                    +--------------+--------------------------+
                    |                                         |
                    v                                         v
  CONTROL PLANE (tainted, no apps)          WORKERS (Traefik, Argo CD, your apps)
  +----------------+ +----------------+     +-----------------------------------+
  |   k3s-srv-1    | |   k3s-srv-2    |     |  k3s-agent-1      192.168.99.31   |
  | 192.168.99.21  | | 192.168.99.22  |     +-----------------------------------+
  |  k3s server    | |  k3s server    |     |  k3s-agent-2      192.168.99.32   |
  +-------+--------+ +--------+-------+     +-----------------------------------+
          |                   |             |  k3s-agent-3      192.168.99.33   |
          +---------+---------+             +-----------------------------------+
                    |  mysql :3306          agents join API via k3s-lb :6443
                    v                       Argo CD pulls Git via NAT -> GitHub
          +--------------------+
          |  k3s-db            |
          |  MariaDB           |
          |  192.168.99.11     |
          +--------------------+

  NICs on every VM:  eth1 = Host-only 192.168.99.0/24  (cluster + admin traffic)
                     eth0 = NAT -> Internet             (apt, images, GitHub)
  OS on every VM:    Kali Linux (rolling) - minimal on the nodes, Xfce desktop on kali
```

**Why two NICs per VM?**
- **NIC 1 (NAT, `eth0`)** gives each VM internet access. Every VM gets the *same* NAT address, `10.0.2.15`, so this NIC is useless for traffic between VMs.
- **NIC 2 (host-only, `eth1`)** puts every VM and your Windows host on one private network with fixed IPs. All cluster traffic uses it.

K3s therefore gets `node-ip` and `flannel-iface: eth1` explicitly. If you skip that, the nodes advertise `10.0.2.15` and the cluster breaks. This is the classic VirtualBox gotcha.

### 0.2 IP plan and sizing

| VM / thing | Role | Host-only IP (`eth1`) | vCPU | RAM | Disk (dynamic) |
|---|---|---|---|---|---|
| Windows host | admin workstation (kubectl, argocd, ssh, browser) | `192.168.99.1` | – | – | – |
| VirtualBox DHCP | only for the template and fresh clones | `192.168.99.2` (pool `.100–.254`) | – | – | – |
| `k3s-lb` | NGINX L4 load balancer (API + ingress) | `192.168.99.10` | 1 | 1 GB | 25 GB |
| `k3s-db` | MariaDB external datastore | `192.168.99.11` | 2 | 2 GB | 25 GB |
| `k3s-srv-1` | K3s server (control plane) | `192.168.99.21` | 2 | 4 GB | 25 GB |
| `k3s-srv-2` | K3s server (control plane) | `192.168.99.22` | 2 | 4 GB | 25 GB |
| `k3s-agent-1` | K3s agent (worker) | `192.168.99.31` | 4 | 4 GB | 25 GB |
| `k3s-agent-2` | K3s agent (worker) | `192.168.99.32` | 4 | 4 GB | 25 GB |
| `k3s-agent-3` | K3s agent (worker) | `192.168.99.33` | 4 | 4 GB | 25 GB |
| `kali` | Kali desktop admin workstation ([Appendix E](#appendix-e--kali-linux-workstation-vm)) | `192.168.99.50` | 2 | 4 GB | 25 GB |
| **Total** | | | **21** | **27 GB** | ~45 GB actually used |

The variants reserve three more IPs. Appendix A adds `k3s-srv-3` at `192.168.99.23`. Appendix B replaces `k3s-lb` with `k3s-lb-1` at `.8` and `k3s-lb-2` at `.9`, and `.10` becomes their shared virtual IP. All three are already in the `home.lab` names (Step 4) and the SSH config (Step 6).

Every VM runs **Kali Linux** from the same installer ISO. The 7 cluster VMs are clones of `kali-template`, a minimal install with no desktop. `kali` is installed separately, and its Xfce desktop comes from ticking **Xfce** on the installer's **Software selection** screen ([Appendix E](#appendix-e--kali-linux-workstation-vm)).

The lab domain is **`home.lab`**. Each cluster VM gets its FQDN from `set-identity` (for example `k3s-srv-1.home.lab`), and resolves every other lab name through the `/etc/hosts` block it inherits from the template (Step 4). `kali` gets `kali.home.lab` from its own installer and the same block in Appendix E.2. On Windows, the `hosts` file maps `k3s.home.lab`, `argocd.home.lab` and `whoami.home.lab` to `192.168.99.10`, the load balancer (Step 15), and `kali.home.lab` to `192.168.99.50` (Appendix E).

> **Smaller host?** Servers and agents still work with 2 GB each (about 12 GB total). Two agents are enough, unless you want the Argo CD **HA** manifest, which needs 3 schedulable nodes. The `kali` workstation also runs with 2 GB, and can stay powered off when you don't need it.

**How many VMs?**

| Scenario | VMs | Which |
|---|---|---|
| Created in VirtualBox | **9** | `kali-template` + 7 cluster VMs + `kali`, all Kali Linux |
| Installed from the Kali ISO | **2** | `kali-template` (Step 3, no desktop) and `kali` (Appendix E, Xfce ticked in Software selection); the other 7 are clones |
| Running at the same time | **8** | 7 cluster VMs + `kali`; the template stays powered off after Step 4 |
| Needed for the cluster itself | **7** | `k3s-lb`, `k3s-db`, `k3s-srv-1`, `k3s-srv-2`, `k3s-agent-1/2/3` |
| Bare minimum (non-HA Argo CD, no `kali` workstation) | **6** | same, with 2 agents |
| Appendix A: embedded etcd | 7 | `k3s-db` replaced by `k3s-srv-3` |
| Appendix B: HA load balancer | 8 | `k3s-lb` replaced by `k3s-lb-1` + `k3s-lb-2` |
| Appendix A + B together | 8 | 2 LBs + 3 servers + 3 agents, no DB |

### 0.3 Traffic and ports

| From → To | Port | Purpose |
|---|---|---|
| Windows host, agents → `k3s-lb` | 6443/tcp | Kubernetes API (kubectl, agent join) |
| Windows host → `k3s-lb` | 80, 443/tcp | Ingress: Argo CD UI, your apps |
| `k3s-lb` → `k3s-srv-1/2` | 6443/tcp | API upstreams |
| `k3s-lb` → `k3s-agent-1/2/3` | 80, 443/tcp | Traefik (exposed on every node by K3s ServiceLB) |
| `k3s-srv-1/2` → `k3s-db` | 3306/tcp | K3s datastore (kine → MariaDB) |
| all nodes ↔ all nodes | 8472/udp | Flannel VXLAN (pod network) |
| all nodes ↔ all nodes | 10250/tcp | kubelet (logs, exec, metrics) |
| `kali` → `k3s-lb` | 6443, 80, 443/tcp | Same as the Windows host: kubectl, Argo CD UI/CLI, apps |
| Windows host → `kali`; `kali` → all VMs | 22/tcp | SSH |
| Windows host `127.0.0.1:2222` → `kali-template`; `127.0.0.1:220xx` → each cluster VM | 22/tcp, through a VirtualBox NAT port forward on `eth0` | SSH with copy and paste: `ssh -p <port> kali@127.0.0.1`. Works before the host-only IPs exist (Steps 4 and 6), and every later step that runs inside a VM starts with it. `xx` = last number of the VM's IP (Step 5) |

Kali enables **no firewall** by default (no `ufw` or `firewalld`), and this lab leaves it that way. The host-only network is reachable only from your PC.

### 0.4 Order of operations (where you type what)

```mermaid
flowchart TD
    A["Step 1-2 · Windows host<br/>tools, SSH key, host-only DHCP pool"]
    B["Step 3-4 · kali-template VM<br/>install Kali (minimal), prep, set-identity helper"]
    C["Step 5-6 · Windows host + each VM console<br/>clone 7 cluster VMs, give each its name and IP"]
    D["Step 7 · k3s-db<br/>MariaDB, k3s database and user, secrets"]
    E["Step 8 · k3s-lb<br/>NGINX TCP load balancer"]
    F["Step 9 · k3s-srv-1<br/>first K3s server"]
    G["Step 10 · k3s-srv-2<br/>second K3s server"]
    H["Step 11 · k3s-agent-1/2/3<br/>join workers through the LB"]
    I["Step 12-13 · Windows host<br/>kubeconfig, HA failover test"]
    J["Step 14-16 · Windows host<br/>install, expose and log in to Argo CD"]
    K["Step 17-18 · Windows host + GitHub<br/>GitOps apps, app-of-apps"]
    A --> B --> C --> D --> E --> F --> G --> H --> I --> J --> K
```

| Step | Run on | What |
|---|---|---|
| 1–2 | Windows host (PowerShell) | kubectl, argocd, portable Git and the Kali ISO into `C:\Working\tools` (checksums verified), VirtualBox, SSH key; then reset and recreate the VirtualBox networks (host-only adapter + DHCP pool, NAT network) in an Administrator PowerShell |
| 3 | Windows host + template console | create VM with the SSH port forward `127.0.0.1:2222`, install Kali (minimal, no desktop) |
| 4 | `kali-template` (3 lines in the console, then `ssh -p 2222 kali@127.0.0.1`) | SSH, NetworkManager, `eth0`/`eth1` names, updates, swap off, `home.lab` names, `set-identity` helper, pre-flight check, power off |
| 5 | Windows host | clone 7 cluster VMs, give each its own SSH port forward, start them |
| 6 | **each VM's console window**, or Windows through the port forwards | `sudo set-identity <name> <ip>` |
| 7 | `k3s-db` (`ssh -p 22011 kali@127.0.0.1`) | MariaDB, DB and user, generate secrets |
| 8 | `k3s-lb` (`ssh -p 22010 kali@127.0.0.1`) | NGINX stream config |
| 9 | `k3s-srv-1` (`ssh -p 22021 kali@127.0.0.1`) | K3s server #1 |
| 10 | `k3s-srv-2` (`ssh -p 22022 kali@127.0.0.1`) | K3s server #2 |
| 11 | `k3s-agent-1`, `-2`, `-3` (`ssh -p 22031 kali@127.0.0.1`, then port `22032`, `22033`) | K3s agents |
| 12–18 | Windows host | kubectl, HA test, Argo CD, GitOps |
| App. E | Windows host + `kali` console | create `kali`, install Kali with **Xfce** ticked in Software selection, then network + SSH (any time after Step 6); kubectl and argocd (after Steps 12 / 16) |

### 0.5 Conventions

- `powershell` blocks run in **PowerShell on the Windows host**.
- `bash` blocks run **inside the named VM**. Each step that runs inside the template or a cluster VM first shows the command that connects to it from Windows: `ssh -p <port> kali@127.0.0.1`, through that VM's own NAT port forward. Run it in a normal PowerShell window (not ISE), paste the `bash` block into the session, and type `exit` to get back to Windows before the next `powershell` block (or use a second PowerShell window).
  - **The port picks the VM.** `2222` is **only** `kali-template` (Step 3). Each clone has its own port, `22000` + the last number of its IP (Step 5): `22010` = `k3s-lb`, `22011` = `k3s-db`, `22021` / `22022` = `k3s-srv-1` / `k3s-srv-2`, `22031` / `22032` / `22033` = `k3s-agent-1` / `-2` / `-3`. `ssh -p 2222` never reaches a cluster node.
  - The first connection to each port asks you to confirm the VM's host key: type `yes`.
  - `ssh <vm-name>` (the shortcuts from Step 6) reaches the same VM over the host-only network instead. The `kali` workstation has no port forward: use `ssh kali` or a terminal on its desktop (Appendix E).
- Kali's default login shell is **zsh**. Step 4 switches the `kali` user to **bash** in the template, so `bash` blocks paste as-is on every clone. The `kali` workstation keeps zsh. Kali's zsh config allows `#` comments, so its blocks paste as-is too.
- **Run on:** at the start of each step tells you where.
- If your host-only network is not `192.168.99.0/24`, replace `192.168.99.` everywhere, or set it to `.99` in Step 2.

---

## Step 1 — Windows host: tools, PATH, SSH key

Everything this guide needs on Windows is downloaded into **`C:\Working\tools`**. Each download is checked against its publisher's SHA-256 checksum or code signature.

```text
C:\Working\tools\
├── bin\          kubectl.exe, argocd.exe                       (user PATH + PowerShell profile)
├── git\          Git for Windows, portable; git\cmd\git.exe     (user PATH + PowerShell profile)
├── iso\          the Kali installer ISO used in Step 3 and Appendix E
└── downloads\    the installers: VirtualBox, Visual C++ runtime, PortableGit
```

The tools get their own `tools` subfolder, so they never mix with your other work in `C:\Working`. Creating it needs no administrator rights. Folders created at the root of `C:` (such as `C:\Working`) are writable by every signed-in account on this PC, and `tools` inherits that. That is fine on a single-user lab PC.

**Why a PowerShell profile as well as the user PATH.** Windows searches the **system** PATH before the **user** PATH. Docker Desktop puts its own, usually older, `kubectl.exe` on the system PATH (`C:\Program Files\Docker\Docker\resources\bin`), and the Git for Windows installer adds `C:\Program Files\Git\cmd`. A folder on the user PATH can never come before those. Block 1.1 therefore also adds one line to your PowerShell profile. That line puts `C:\Working\tools\bin` and `C:\Working\tools\git\cmd` first in every PowerShell window (console, ISE, Windows Terminal, VS Code). It needs no administrator rights, and Docker and the installed Git keep working.

Two things are deliberately **not** under `C:\Working\tools`:

- **VirtualBox.** It installs kernel drivers. Its manual requires every parent folder of a custom install folder to be locked down: inheritance off, and Users / Authenticated Users only allowed to read. For `C:\Working\tools\VirtualBox`, that would make `C:\Working` itself read-only for your own account, including your other projects there. VirtualBox therefore goes into its default `C:\Program Files\Oracle\VirtualBox`, and its installer is kept in `C:\Working\tools\downloads`.
- **The OpenSSH client** (`ssh`, `scp`, `ssh-keygen`). It is part of Windows (`C:\Windows\System32\OpenSSH`), and block 1.5 makes sure it is there.

**Run on:** Windows host, in a normal PowerShell window. Run blocks **1.1 to 1.6 in the same window**, because they share `$Tools` and two helper functions.

- The VirtualBox and Visual C++ installers show a **UAC prompt**: accept it.
- The steps are safe to run again: anything already present with the right checksum is skipped, so a rerun only updates what changed.

**1.1 The `C:\Working\tools` folders, user PATH, PowerShell profile and helpers**

```powershell
# Where every tool goes: its own folder next to your other work in C:\Working
$Tools = "C:\Working\tools"
foreach ($d in "bin", "git", "iso", "downloads") { New-Item -ItemType Directory -Force "$Tools\$d" | Out-Null }

# Ran an earlier version of this step (tools in C:\tools, or in Desktop\HA_K3S_ARGOCD\Tools)? This moves ONLY this
# guide's own files into C:\Working\tools, so nothing is downloaded twice. Other software in those folders
# (e.g. C:\tools\flutter, C:\tools\bfg) is left alone. On a first run it finds nothing and changes nothing.
$OldRoots = "C:\tools", "C:\Users\USER\Desktop\HA_K3S_ARGOCD\Tools"
foreach ($Old in $OldRoots) {
    foreach ($f in "bin\kubectl.exe", "bin\argocd.exe", "iso\kali-linux-*.iso", "kali-linux-*.iso",
                   "downloads\vc_redist.x64.exe", "downloads\VirtualBox-*-Win.exe", "downloads\PortableGit-*.7z.exe") {
        $to = if ($f -like "kali-linux-*") { "iso" } else { Split-Path $f }
        Get-Item "$Old\$f" -ErrorAction SilentlyContinue | Move-Item -Destination "$Tools\$to" -Force -PassThru | ForEach-Object { "moved here: $($_.FullName)" }
    }
    if ((Test-Path "$Old\git\git-bash.exe") -and -not (Test-Path "$Tools\git\git-bash.exe")) {
        Get-ChildItem "$Old\git" -Force | Move-Item -Destination "$Tools\git" -Force
        "moved here: $Tools\git (portable Git)"
    }
}
# take this guide's old bin / git\cmd folders off the user PATH, but only those that are now empty or gone
$stale = foreach ($Old in $OldRoots) { foreach ($d in "$Old\bin", "$Old\git\cmd") { if (-not (Test-Path $d) -or -not (Get-ChildItem $d -Force)) { $d } } }
$keep  = @(([Environment]::GetEnvironmentVariable("Path", "User") -split ';') | Where-Object { $_ -and $_.TrimEnd('\') -notin $stale })
[Environment]::SetEnvironmentVariable("Path", ($keep -join ';'), "User")
[Environment]::SetEnvironmentVariable("HA_TOOLS", $null, "User")      # set by an earlier version; no longer used

# Windows PowerShell 5.1: allow TLS 1.2, hide the slow progress bar
[Net.ServicePointManager]::SecurityProtocol = [Net.ServicePointManager]::SecurityProtocol -bor [Net.SecurityProtocolType]::Tls12
$ProgressPreference = 'SilentlyContinue'

# Download $Url to $OutFile unless the same file (same SHA-256) is already there; delete it if the checksum is wrong
function Get-Verified([string]$Url, [string]$OutFile, [string]$Sha256) {
    if (-not $Sha256) { throw "no checksum found for $Url" }
    if ((Test-Path $OutFile) -and (Get-FileHash $OutFile -Algorithm SHA256).Hash -eq $Sha256) { "up to date : $OutFile"; return }
    curl.exe -fL --retry 3 -o $OutFile $Url
    if ($LASTEXITCODE -ne 0) { throw "download failed: $Url" }
    if ((Get-FileHash $OutFile -Algorithm SHA256).Hash -ne $Sha256) { Remove-Item $OutFile; throw "SHA-256 mismatch, deleted $OutFile" }
    "verified   : $OutFile"
}

# Put a folder FIRST on the user PATH (permanent) and on this window's PATH
function Add-UserPath([string]$Dir) {
    $Dir  = $Dir.TrimEnd('\')
    $rest = @(([Environment]::GetEnvironmentVariable("Path", "User") -split ';') | Where-Object { $_ -and $_.TrimEnd('\') -ne $Dir })
    [Environment]::SetEnvironmentVariable("Path", ((@($Dir) + $rest) -join ';'), "User")
    $env:Path = (@($Dir) + @(($env:Path -split ';') | Where-Object { $_ -and $_.TrimEnd('\') -ne $Dir })) -join ';'
}
Add-UserPath "$Tools\bin"
Add-UserPath "$Tools\git\cmd"

# New windows search the SYSTEM PATH first, so Docker Desktop's kubectl or an installed Git would still win there.
# One line in your PowerShell profile (console, ISE, Windows Terminal, VS Code) puts this guide's tools first.
$ProfileLine = @'
foreach ($d in 'C:\Working\tools\git\cmd', 'C:\Working\tools\bin') { $env:Path = (@($d) + @(($env:Path -split ';') | Where-Object { $_ -and $_.TrimEnd('\') -ne $d })) -join ';' }   # k3s-argocd-setup: before Docker's kubectl / Program Files\Git
'@
$Prof = $PROFILE.CurrentUserAllHosts                   # Documents\WindowsPowerShell\profile.ps1
if (-not (Test-Path $Prof)) { New-Item -ItemType File -Force $Prof | Out-Null }
if (-not (Select-String -Path $Prof -SimpleMatch 'k3s-argocd-setup' -Quiet)) { Add-Content -Path $Prof -Value $ProfileLine }
# Windows' default execution policy (Restricted) loads no profile at all; RemoteSigned for your own account allows it
if ((Get-ExecutionPolicy -Scope CurrentUser) -eq 'Undefined' -and (Get-ExecutionPolicy -Scope LocalMachine) -in 'Undefined', 'Restricted') {
    try { Set-ExecutionPolicy -Scope CurrentUser RemoteSigned -Force } catch { }
}
```

**1.2 VirtualBox and the Microsoft Visual C++ runtime it needs**

```powershell
$VBoxBranch = "7.1"      # the branch this guide was written for; "7.2" = the newest
$vbDir = [Environment]::GetEnvironmentVariable("VBOX_MSI_INSTALL_PATH", "Machine")
if (-not $vbDir -and (Test-Path "$env:ProgramFiles\Oracle\VirtualBox\VBoxManage.exe")) { $vbDir = "$env:ProgramFiles\Oracle\VirtualBox" }

if ($vbDir -and (Test-Path "$vbDir\VBoxManage.exe")) {
    "VirtualBox already installed in $vbDir ($(& "$vbDir\VBoxManage.exe" --version)) - keeping it"
} else {
    # Visual C++ runtime (x64): no fixed checksum is published, so check Microsoft's code signature instead
    $vc = "$Tools\downloads\vc_redist.x64.exe"
    curl.exe -fL --retry 3 -o $vc https://aka.ms/vs/17/release/vc_redist.x64.exe
    $sig = Get-AuthenticodeSignature $vc
    if ($sig.Status -ne 'Valid' -or $sig.SignerCertificate.Subject -notmatch 'O=Microsoft Corporation') { Remove-Item $vc; throw "vc_redist.x64.exe is not signed by Microsoft" }
    Start-Process -Wait $vc -ArgumentList '/install', '/quiet', '/norestart'

    # newest release of $VBoxBranch, checked against Oracle's SHA256SUMS and Oracle's code signature
    $vbVer = [regex]::Matches(((curl.exe -fsSL https://download.virtualbox.org/virtualbox/) -join "`n"), 'href="(' + [regex]::Escape($VBoxBranch) + '\.\d+)/"') |
             ForEach-Object { [version]$_.Groups[1].Value } | Sort-Object | Select-Object -Last 1
    $line  = (curl.exe -fsSL "https://download.virtualbox.org/virtualbox/$vbVer/SHA256SUMS") | Where-Object { $_ -match '\*VirtualBox-[\d.]+-\d+-Win\.exe$' } | Select-Object -First 1
    $vbExe = "$Tools\downloads\" + $line.Split('*')[1]
    Get-Verified "https://download.virtualbox.org/virtualbox/$vbVer/$($line.Split('*')[1])" $vbExe $line.Split(' ')[0]
    if ((Get-AuthenticodeSignature $vbExe).SignerCertificate.Subject -notmatch 'O=Oracle') { throw "$vbExe is not signed by Oracle" }

    # unattended install into the default C:\Program Files\Oracle\VirtualBox (accept the UAC prompt)
    $p = Start-Process -Wait -PassThru $vbExe -ArgumentList '--silent', '--ignore-reboot', '--msiparams', 'VBOX_INSTALLDESKTOPSHORTCUT=0', '--msiparams', 'VBOX_INSTALLQUICKLAUNCHSHORTCUT=0', '--msiparams', 'VBOX_START=0'
    "VirtualBox installer exit code: $($p.ExitCode)   (0 = OK, 3010 = OK, restart Windows before Step 2)"
    $vbDir = [Environment]::GetEnvironmentVariable("VBOX_MSI_INSTALL_PATH", "Machine")
    if (-not $vbDir) { throw "VirtualBox did not install - run $vbExe by hand and accept the defaults" }
}
Add-UserPath $vbDir
VBoxManage --version
```

**1.3 kubectl and the Argo CD CLI → `C:\Working\tools\bin`**

```powershell
# kubectl: current stable release, checked against the .sha256 file published next to it
$k8s  = (curl.exe -fsSL https://dl.k8s.io/release/stable.txt).Trim()
$ksha = (curl.exe -fsSL "https://dl.k8s.io/release/$k8s/bin/windows/amd64/kubectl.exe.sha256").Trim()
Get-Verified "https://dl.k8s.io/release/$k8s/bin/windows/amd64/kubectl.exe" "$Tools\bin\kubectl.exe" $ksha

# argocd: latest release, checked against the release's cli_checksums.txt
$aline = (curl.exe -fsSL https://github.com/argoproj/argo-cd/releases/latest/download/cli_checksums.txt) | Where-Object { $_ -match '\sargocd-windows-amd64\.exe$' } | Select-Object -First 1
Get-Verified https://github.com/argoproj/argo-cd/releases/latest/download/argocd-windows-amd64.exe "$Tools\bin\argocd.exe" ($aline -split '\s+')[0]
```

**1.4 Git for Windows, portable → `C:\Working\tools\git`** (no installer, nothing in Program Files)

```powershell
$rel   = Invoke-RestMethod https://api.github.com/repos/git-for-windows/git/releases/latest
$asset = $rel.assets | Where-Object { $_.name -match '^PortableGit-[\d.]+-64-bit\.7z\.exe$' } | Select-Object -First 1
$want  = "git version " + $rel.tag_name.TrimStart('v')            # e.g. git version 2.56.0.windows.4
$have  = if (Test-Path "$Tools\git\cmd\git.exe") { & "$Tools\git\cmd\git.exe" --version } else { "" }
if ($have -eq $want) { "up to date : $have" } else {
    $sfx = "$Tools\downloads\$($asset.name)"
    Get-Verified $asset.browser_download_url $sfx ($asset.digest -replace '^sha256:', '')     # GitHub's SHA-256 of the asset
    Start-Process -Wait $sfx -ArgumentList "-o`"$Tools\git`"", '-y'                           # self-extracting archive: unpack only
    & "$Tools\git\cmd\git.exe" --version
}
```

**1.5 OpenSSH client (part of Windows)**

```powershell
if (Get-Command ssh.exe -ErrorAction SilentlyContinue) { "OpenSSH client present: $((Get-Command ssh.exe).Source)" } else {
    Start-Process powershell -Verb RunAs -Wait -ArgumentList '-NoProfile -Command "Add-WindowsCapability -Online -Name OpenSSH.Client~~~~0.0.1.0"'
}
```

**1.6 Kali installer ISO → `C:\Working\tools\iso`** (used in Step 3 and Appendix E)

```powershell
$KaliImage = "installer-netinst"     # small; fetches current packages during the install. "installer" = the full, much larger image
$kline = (curl.exe -fsSL https://cdimage.kali.org/current/SHA256SUMS) | Where-Object { $_ -match "\skali-linux-[\d.]+[a-z]?-$KaliImage-amd64\.iso$" } | Select-Object -First 1
$isoHash, $isoName = $kline -split '\s+'
Get-Verified "https://cdimage.kali.org/current/$isoName" "$Tools\iso\$isoName" $isoHash
```

**1.7 Verify.** Open a **new** PowerShell window so the PATH changes apply, then run:

```powershell
foreach ($cmd in "VBoxManage", "kubectl", "argocd", "git", "ssh", "scp", "ssh-keygen", "curl.exe") {
    $src = (Get-Command $cmd -ErrorAction SilentlyContinue | Select-Object -First 1).Source
    if (-not $src) { $tag = "MISSING" }
    elseif ($src -like "C:\Working\tools\*") { $tag = "OK  (C:\Working\tools)" }
    elseif ($src -like "$env:SystemRoot\*") { $tag = "OK  (Windows)" }
    elseif ($env:VBOX_MSI_INSTALL_PATH -and $src -like "$($env:VBOX_MSI_INSTALL_PATH.TrimEnd('\'))\*") { $tag = "OK  (VirtualBox)" }
    else { $tag = "WARN another copy is found first" }
    "{0,-11} {1,-33} {2}" -f $cmd, $tag, $src
}
VBoxManage --version
kubectl version --client
argocd version --client --short
git --version
Get-ChildItem "C:\Working\tools\iso\*.iso" | ForEach-Object { "{0}  {1:N1} GB" -f $_.Name, ($_.Length / 1GB) }
```

Every tool must show **`OK`**:

```text
VBoxManage  OK  (VirtualBox)                  C:\Program Files\Oracle\VirtualBox\VBoxManage.exe
kubectl     OK  (C:\Working\tools)            C:\Working\tools\bin\kubectl.exe
argocd      OK  (C:\Working\tools)            C:\Working\tools\bin\argocd.exe
git         OK  (C:\Working\tools)            C:\Working\tools\git\cmd\git.exe
ssh         OK  (Windows)                     C:\WINDOWS\System32\OpenSSH\ssh.exe
...
```

- **`MISSING`**: run that tool's block again (1.2–1.5), then open a new window.
- **`WARN`**: another copy is found before `C:\Working\tools`, usually on the **system** PATH, which Windows searches before the user PATH. Typical examples are Docker Desktop's `kubectl` (`C:\Program Files\Docker\Docker\resources\bin`) and an installed Git (`C:\Program Files\Git\cmd`). The profile line from 1.1 fixes this, so run 1.1 again and open a new window. If it still says `WARN`, the profile did not load: `Test-Path $PROFILE.CurrentUserAllHosts` must print `True`, and `Get-ExecutionPolicy` must not print `Restricted`.
  - **Why it matters for `kubectl`:** it is supported only within one minor version of the cluster, and Docker Desktop's copy is usually several versions older. `kubectl version --client` must print the version that 1.3 downloaded.
  - **For `git`**, the other copy works as well. The warning only means it isn't the copy from 1.4.
  - The profile only affects PowerShell. cmd.exe and Git Bash still find Docker's `kubectl` first. To change that too, move `C:\Working\tools\bin` above Docker's entry in the **system** `Path` (*System Properties → Environment Variables*, needs admin).
- **Upgrading later:** run 1.1–1.4 and 1.6 again. Tools that are already current are left alone.

Create an SSH key pair (press Enter to accept the default path; a passphrase is optional):

```powershell
if (-not (Test-Path "$HOME\.ssh\id_ed25519")) { ssh-keygen -t ed25519 }
```

> **Hyper-V note.** Run `(Get-CimInstance Win32_ComputerSystem).HypervisorPresent`. If it prints `True` (Hyper-V, WSL2, or Memory integrity is on), VirtualBox runs on top of Hyper-V and shows a green turtle icon in the VM status bar. It works, just slower.
> For full speed, run `bcdedit /set hypervisorlaunchtype off` in an **admin** PowerShell, turn off *Core isolation → Memory integrity*, and reboot. This disables WSL2 and Docker Desktop. To undo it: `bcdedit /set hypervisorlaunchtype auto`.

---

## Step 2 — Windows host: host-only network and DHCP pool

**Run on:** Windows host, in an **Administrator** PowerShell window, because removing and creating a network adapter needs admin. Accept the UAC prompt if VirtualBox shows one. Every VM must be powered off.

The block always starts from empty, so it gives the same result whatever VirtualBox has now. It removes **every** VirtualBox host-only adapter, NAT network and DHCP server, then creates exactly what this guide needs:

| Network | Created as | Used by |
|---|---|---|
| Host-only adapter | `VirtualBox Host-Only Ethernet Adapter`, Windows side `192.168.99.1/24` | NIC 2 (`eth1`) of every VM |
| Its DHCP server | `192.168.99.2`, pool `.100–.254` | Only the template and fresh clones. `.3–.99` stay free for the static IPs (`.10`–`.33`, `.8`/`.9` for Appendix B, `.50` for Kali) |
| NAT network | `NatNetwork`, `10.0.2.0/24` with DHCP (VirtualBox's default) | Nothing in this guide. NIC 1 uses plain **NAT**, which needs no network here. It is recreated so that *File → Tools → Network Manager* is in a known state |

Run it again whenever the network is broken, for example after **Remove** in Network Manager. VMs refer to the adapter and the NAT network by name, and the names come back the same.

> ⚠️ Other VirtualBox VMs on this PC that use a host-only adapter or a NAT network of their own lose those settings. Plain NAT is not affected.

```powershell
& {   # one script block, so it stops at the first problem even when pasted line by line
$HOA    = "VirtualBox Host-Only Ethernet Adapter"     # the name Windows gives the first host-only adapter
$NatNet = "NatNetwork"                                 # VirtualBox's default NAT network name

if (VBoxManage list runningvms) { throw "Power off every VM first: VBoxManage list runningvms" }

# 1) remove every host-only adapter, NAT network and DHCP server, including leftovers that belong to nothing
foreach ($n in @(VBoxManage list hostonlyifs) -match '^Name:' -replace '^Name:\s+')                { VBoxManage hostonlyif remove $n.Trim() }
foreach ($n in @(VBoxManage list natnets) -match '^(Network)?Name:' -replace '^(Network)?Name:\s+') { VBoxManage natnetwork remove --netname $n.Trim() }
foreach ($n in @(VBoxManage list dhcpservers) -match '^NetworkName:' -replace '^NetworkName:\s+')   { VBoxManage dhcpserver remove --network $n.Trim() }
# host-only adapters that Windows still knows but VirtualBox doesn't; they would turn the new one into "... Adapter #2"
Get-PnpDevice -Class Net -ErrorAction SilentlyContinue | Where-Object FriendlyName -like "VirtualBox Host-Only*" | ForEach-Object { pnputil /remove-device $_.InstanceId | Out-Null }

# 2) 192.168.99.x must not be used by anything else on this PC (home router, VPN, Hyper-V switch)
$clash = @(Get-NetIPAddress -AddressFamily IPv4 -ErrorAction SilentlyContinue | Where-Object IPAddress -like "192.168.99.*")
if ($clash) { throw "192.168.99.x is already used by '$($clash[0].InterfaceAlias)'. Pick another network, see 0.5 Conventions" }

# 3) the host-only adapter, Windows side 192.168.99.1/24
$made = if (((VBoxManage hostonlyif create) -join "`n") -match "Interface '([^']+)' was successfully created") { $Matches[1] }
if ($made -ne $HOA) { throw "hostonlyif create made '$made' instead of '$HOA'. See Appendix C" }
VBoxManage hostonlyif ipconfig $HOA --ip 192.168.99.1 --netmask 255.255.255.0

# 4) its DHCP server hands out only .100-.254 ('create' may already have made one with VirtualBox's default addresses)
$op = if (@(VBoxManage list dhcpservers) -match "^NetworkName:\s+HostInterfaceNetworking-$([regex]::Escape($HOA))\s*$") { "modify" } else { "add" }
VBoxManage dhcpserver $op --interface $HOA --server-ip 192.168.99.2 --netmask 255.255.255.0 --lower-ip 192.168.99.100 --upper-ip 192.168.99.254 --enable

# 5) VirtualBox's default NAT network (the VMs in this guide don't use it)
VBoxManage natnetwork add --netname $NatNet --network "10.0.2.0/24" --enable --dhcp on

# 6) check
VBoxManage list hostonlyifs | Select-String "^Name:|^IPAddress:|^NetworkMask:|^Status:"
VBoxManage list dhcpservers | Select-String "^NetworkName:|^Dhcpd IP:|^LowerIPAddress:|^UpperIPAddress:|^Enabled:"
VBoxManage list natnets     | Select-String "^(Network)?Name:|^Network:|^DHCP|^Enabled:"
$hif  = (VBoxManage list hostonlyifs) -join "`n"
$dhcp = (VBoxManage list dhcpservers) -join "`n"
if ($hif -notmatch "(?m)^IPAddress:\s+192\.168\.99\.1\s*$" -or $dhcp -notmatch "(?m)^LowerIPAddress:\s+192\.168\.99\.100\s*$" -or
    -not (@(VBoxManage list natnets) -match "\b$NatNet\b")) { throw "Step 2 check failed: compare with the expected output below" }
"Step 2 OK"
}
```

Expected output at the end:

```text
Name:            VirtualBox Host-Only Ethernet Adapter
IPAddress:       192.168.99.1
NetworkMask:     255.255.255.0
Status:          Up
NetworkName:    HostInterfaceNetworking-VirtualBox Host-Only Ethernet Adapter
Dhcpd IP:       192.168.99.2
LowerIPAddress: 192.168.99.100
UpperIPAddress: 192.168.99.254
Enabled:        Yes
Name:         NatNetwork
Network:      10.0.2.0/24
DHCP Server:  Yes
Step 2 OK
```

- The `0%...100%` lines before it are VirtualBox removing and creating the adapter. PowerShell ISE shows them in red, but they are not errors.
- There is no `dhcpserver restart`. A DHCP server runs only while a VM uses its network, and it reads these settings when it starts. Before that, `restart` always fails with *not running*.

---

## Step 3 — Windows host: create the template VM and install Kali

You install Kali into `kali-template` as a minimal system with no desktop. Step 5 then clones it 7 times, once per cluster node. The `kali` workstation is a second install from the same ISO, and it is the only VM with a desktop ([Appendix E](#appendix-e--kali-linux-workstation-vm)).

1. Step 1.6 already downloaded and verified the **Kali Linux installer** ISO into **`C:\Working\tools\iso`**. To fetch it by hand instead, download it from <https://www.kali.org/get-kali/#kali-installer-images> into `C:\Working\tools\iso`. Use either the **NetInstaller** (`kali-linux-20xx.x-installer-netinst-amd64.iso`), which is small and fetches current packages through NAT, or the full **Installer** (`kali-linux-20xx.x-installer-amd64.iso`).

2. **Run on:** Windows host:

```powershell
$VM  = "kali-template"
$HOA = "VirtualBox Host-Only Ethernet Adapter"
$ISO = (Get-ChildItem "C:\Working\tools\iso\kali-linux-*-installer*-amd64.iso" | Sort-Object Name -Descending | Select-Object -First 1).FullName
$VDI = "$HOME\VirtualBox VMs\$VM\$VM.vdi"
"ISO: $ISO"     # must print a path under C:\Working\tools\iso - if empty, run Step 1.6

VBoxManage createvm --name $VM --ostype Debian_64 --register
VBoxManage modifyvm $VM --cpus 2 --memory 2048 --vram 16 --graphicscontroller vmsvga --audio-enabled off
VBoxManage modifyvm $VM --nic1 nat --nic2 hostonly --host-only-adapter2 $HOA
VBoxManage modifyvm $VM --natpf1 "ssh,tcp,127.0.0.1,2222,,22"      # NAT port forward: Windows 127.0.0.1:2222 -> template port 22 (Step 4)
VBoxManage modifyvm $VM --boot1 disk --boot2 dvd --boot3 none --boot4 none
VBoxManage createmedium disk --filename $VDI --size 25600 --format VDI
VBoxManage storagectl $VM --name SATA --add sata --controller IntelAhci --portcount 2
VBoxManage storageattach $VM --storagectl SATA --port 0 --device 0 --type hdd --medium $VDI
VBoxManage storageattach $VM --storagectl SATA --port 1 --device 0 --type dvddrive --medium $ISO
VBoxManage startvm $VM --type gui
```

> The boot order is **disk first, then DVD**. The empty disk falls through to the ISO; after the install the VM boots from disk, so there is nothing to eject.

> **Port forward for SSH.** The VM window has no copy and paste, because the template has no desktop. Step 4 therefore works over SSH from a Windows window, where copy and paste work normally. The `--natpf1` line is the command-line form of *Settings → Network → Adapter 1 (NAT) → Advanced → Port Forwarding → +*:
>
> | Name | Protocol | Host IP | Host Port | Guest IP | Guest Port |
> |---|---|---|---|---|---|
> | `ssh` | TCP | `127.0.0.1` | `2222` | *(blank)* | `22` |
>
> It works while the VM is running and its SSH server is on (Step 4 part 1). Host IP `127.0.0.1` means only your own PC can use it. Every clone inherits this rule, so Step 5 gives each clone its own port.
>
> **`kali-template` already exists without the rule?** Add it now: `VBoxManage modifyvm kali-template --natpf1 "ssh,tcp,127.0.0.1,2222,,22"` while it is powered off, or `VBoxManage controlvm kali-template natpf1 "ssh,tcp,127.0.0.1,2222,,22"` while it runs. Check with `VBoxManage showvminfo kali-template --machinereadable | Select-String '^Forwarding'`, which must print `Forwarding(0)="ssh,tcp,127.0.0.1,2222,,22"`.

3. **In the VM window**, go through the Kali installer (it is the Debian installer):

| Screen | Choose |
|---|---|
| Boot menu | **Graphical install** |
| Language / location / keyboard | your choice |
| Primary network interface | the **first** interface in the list, shown as `eth0` (or `enp0s3`), which is Adapter 1 = NAT. Both entries say *Intel 82540EM*, so go by the order: the second one is the host-only NIC and has no internet |
| Hostname | `kali-template` |
| Domain name | **`home.lab`** |
| Full name / username | `kali` / **`kali`** · a password (every clone inherits it) |
| Time zone | your choice (Step 4 sets UTC) |
| Partition disks | **Guided - use entire disk** → the 25 GB disk → **All files in one partition** → Finish → Yes |
| HTTP proxy / mirror | leave empty / defaults |
| Software selection | **untick every entry**, because the installer pre-ticks Xfce and the top10 / default tools: no desktop, no tools. Every clone inherits this, so the nodes stay minimal and headless. The GUI goes only on `kali`, whose own install ([E.1](#e1-install-kali-with-the-xfce-desktop)) ticks **Xfce** on this same screen |
| GRUB boot loader | **Yes** → `/dev/sda` |
| Finish | **Continue** (the VM reboots from disk) |

---

## Step 4 — kali-template: prepare the golden image

Everything you do here is inherited by all 7 clones.

1. **In the VM window**, log in as `kali` and turn on SSH. The VM window has no copy and paste, so type these three lines. They are the only ones you type there; everything after them is pasted over SSH:

```bash
sudo apt update
sudo apt install -y openssh-server
sudo systemctl enable --now ssh
```

`systemctl is-active ssh` must print `active`. Windows can now reach the template through the Step 3 port forward, `127.0.0.1:2222`.

2. **Run on:** Windows host, in a **normal PowerShell window** (or Windows Terminal), **not PowerShell ISE**. Add an SSH shortcut `kali-template` for that port forward, forget the host keys of any earlier template install, copy your SSH key into the template, and connect. The first `ssh` asks for the `kali` password you set in the installer:

> PowerShell ISE can't show the password prompt of `ssh` and can't run an interactive SSH session, so the command just hangs with no output. If that happened, click **Stop** (the red square) in ISE and run the block again in a normal PowerShell window. Every `ssh` command in this guide needs a normal window.

```powershell
if ($psISE) { throw "PowerShell ISE can't ask for the ssh password. Run this block in a normal PowerShell window" }
# SSH shortcut "kali-template" = the Step 3 port forward 127.0.0.1:2222 -> port 22 (added only once)
$cfg = "$HOME\.ssh\config"
if (-not ((Test-Path $cfg) -and (Select-String -Path $cfg -Pattern '^Host kali-template$' -Quiet))) {
@'

Host kali-template
    HostName 127.0.0.1
    Port 2222
    User kali
    HostKeyAlias kali-template
    StrictHostKeyChecking accept-new
'@ | Add-Content -Encoding ascii $cfg
}
# Step 3 installed a NEW system with NEW host keys. Forget the keys of any earlier install,
# otherwise ssh stops with "REMOTE HOST IDENTIFICATION HAS CHANGED" (prints the line it removed, if any)
ssh-keygen -R kali-template 2>$null
ssh-keygen -R "[127.0.0.1]:2222" 2>$null
# copy your Windows SSH key into the template (asks for the kali password once). A PowerShell pipe adds Windows
# line endings and sometimes a BOM: sed removes them, sort -u drops a key that was copied twice
Get-Content "$HOME\.ssh\id_ed25519.pub" | ssh kali-template 'mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && sed -i ''s/^\xEF\xBB\xBF//; s/\r$//'' ~/.ssh/authorized_keys && sort -u -o ~/.ssh/authorized_keys ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys'
# the key must work on its own now: password logins are switched off for this one test
ssh -o PasswordAuthentication=no -o KbdInteractiveAuthentication=no kali-template "echo 'SSH key login: OK'"
ssh kali-template      # logs in with the key now - no password (same VM, same port as: ssh -p 2222 kali@127.0.0.1)
```

The check line must print **`SSH key login: OK`**. If it says *Permission denied (publickey,password)*, your key did not reach the template: run the `Get-Content … | ssh kali-template …` line again in a normal PowerShell window and enter the `kali` password when it asks. Don't continue with only a password login. Every clone inherits the template's `authorized_keys`, and Step 6 needs the key.

You now have the template's shell in a Windows window, where copy and paste work:
- **Paste into the VM:** **Ctrl+V** or right-click.
- **Copy from the VM:** select the text with the mouse, then **Ctrl+C** or right-click.

> `HostKeyAlias kali-template` saves the template's SSH host key under the name `kali-template` instead of `[127.0.0.1]:2222`. The clones keep that same host key until `set-identity` replaces it, which lets Step 6 option B check that it is talking to a fresh clone.

> **Why the two `ssh-keygen -R` lines.** Every Kali install creates new SSH host keys. `~\.ssh\known_hosts` still holds the keys of an earlier install under `kali-template` (and under `[127.0.0.1]:2222` if you ever ran `ssh -p 2222 kali@127.0.0.1`). SSH then refuses with *WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED … Host key verification failed*, even with `accept-new`, which only accepts hosts it has never seen. Removing them is safe here: `127.0.0.1:2222` is a port forward on your own PC to the VM you just installed. To compare fingerprints anyway, run `ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub` in the VM window. It must print the `SHA256:` value that `ssh` shows on its first connection.

Still in that SSH session: hand **both** NICs to NetworkManager and fix the NIC names to `eth0` / `eth1`. Type `sudo -v` and enter the `kali` password first, so no password prompt can interrupt the paste. Then paste:

```bash
sudo apt install -y network-manager
printf 'auto lo\niface lo inet loopback\n' | sudo tee /etc/network/interfaces
sudo rm -f /var/lib/NetworkManager/no-auto-default.state
sudo mkdir -p /etc/default/grub.d
echo 'GRUB_CMDLINE_LINUX="$GRUB_CMDLINE_LINUX net.ifnames=0"' | sudo tee /etc/default/grub.d/90-eth-names.cfg
sudo update-grub
sudo reboot
```

The reboot closes the session (*Connection to 127.0.0.1 closed by remote host*). Wait about 30 seconds, connect again with `ssh -p 2222 kali@127.0.0.1` (or `ssh kali-template`), then run `ip -br a`. `eth0` has `10.0.2.15`, and `eth1` has a DHCP address from the host-only network, for example `192.168.99.108`.

> If `ssh -p 2222 kali@127.0.0.1` gets no answer after the reboot, look at the VM window. `eth0` is probably `UP` but has no address: NetworkManager saw the NAT NIC already configured by the installer, so it never created a profile for it. The port forward runs over `eth0`, so type the fix in the VM window: `sudo rm -f /var/lib/NetworkManager/no-auto-default.state; sudo nmcli connection add type ethernet ifname eth0 con-name nat autoconnect yes; sudo nmcli connection up nat`. The clones are not affected, because `set-identity` later writes its own `eth0` profile.

> `net.ifnames=0` gives every clone the same names: **`eth0` = Adapter 1 (NAT)** and **`eth1` = Adapter 2 (host-only)**. The rest of the guide relies on them.

> The host-only network works for SSH too: `ssh kali@192.168.99.108`, with the `eth1` address that `ip -br a` shows. The port forward is the better choice for the template, because that DHCP address can change, while `127.0.0.1:2222` stays the same.

3. **Run on:** `kali-template`, in the SSH session from part 2. If you closed it, connect again from Windows:

```powershell
ssh -p 2222 kali@127.0.0.1      # kali-template, through the Step 3 port forward   (or: ssh kali-template)
```

Type `bash` first; item 1 makes bash the default shell from then on.

> **Type `sudo -v` and enter the `kali` password before you paste.** The reboot started a new session, so `sudo` asks once more. If its password prompt comes up in the middle of a paste, it swallows the lines pasted after it, and item 7 (`set-identity`) is never written. The clones then say `sudo: set-identity: command not found`. After item 1, `sudo` never asks again.

```bash
# 1) passwordless sudo + bash login shell for 'kali' (lab convenience: copy-paste just works)
echo 'kali ALL=(ALL) NOPASSWD:ALL' | sudo tee /etc/sudoers.d/90-kali-nopasswd
sudo chmod 440 /etc/sudoers.d/90-kali-nopasswd
sudo chsh -s /bin/bash kali
#    Kali's sudo searches only /usr/sbin:/usr/bin:/sbin:/bin. Add /usr/local/sbin, so 'sudo set-identity' (item 7) is found
echo 'Defaults secure_path="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"' > /tmp/91-secure-path
sudo visudo -cqf /tmp/91-secure-path && sudo install -m 440 /tmp/91-secure-path /etc/sudoers.d/91-secure-path

# 2) patch the OS + base tools. noninteractive: a package question can't swallow the rest of the paste.
#    apparmor: containerd needs apparmor_parser when the kernel has AppArmor on
sudo apt update && sudo DEBIAN_FRONTEND=noninteractive apt -y -o Dpkg::Options::=--force-confdef -o Dpkg::Options::=--force-confold full-upgrade
sudo apt install -y curl wget vim jq netcat-openbsd openssl cron apparmor systemd-timesyncd

# 3) timezone + time sync (TLS and datastores hate clock skew)
sudo timedatectl set-timezone UTC          # or your zone, e.g. Europe/Berlin
sudo timedatectl set-ntp true
timedatectl status | grep -E 'synchronized|NTP service'

# 4) disable swap (recommended for Kubernetes nodes; the Kali installer creates a swap partition)
sudo swapoff -a
sudo sed -i '/\sswap\s/ s/^/#/' /etc/fstab

# 5) keep NetworkManager away from the interfaces K3s / flannel create
sudo tee /etc/NetworkManager/conf.d/90-k3s.conf >/dev/null <<'EOF'
[keyfile]
unmanaged-devices=interface-name:cni0;interface-name:flannel*;interface-name:veth*
EOF

# 6) lab domain home.lab: every VM knows every other VM by name (inherited by all clones)
sudo sed -i '/^# BEGIN home.lab/,/^# END home.lab/d' /etc/hosts
sudo tee -a /etc/hosts >/dev/null <<'EOF'
# BEGIN home.lab
192.168.99.10  k3s-lb.home.lab k3s-lb k3s.home.lab argocd.home.lab whoami.home.lab
192.168.99.11  k3s-db.home.lab k3s-db
192.168.99.21  k3s-srv-1.home.lab k3s-srv-1
192.168.99.22  k3s-srv-2.home.lab k3s-srv-2
192.168.99.31  k3s-agent-1.home.lab k3s-agent-1
192.168.99.32  k3s-agent-2.home.lab k3s-agent-2
192.168.99.33  k3s-agent-3.home.lab k3s-agent-3
192.168.99.50  kali.home.lab kali
# only used by Appendix A (k3s-srv-3) and Appendix B (k3s-lb-1/2):
192.168.99.23  k3s-srv-3.home.lab k3s-srv-3
192.168.99.8   k3s-lb-1.home.lab k3s-lb-1
192.168.99.9   k3s-lb-2.home.lab k3s-lb-2
# END home.lab
EOF

# 7) helper that gives each clone its own hostname / FQDN / static IP / machine-id / SSH host keys
#    save the template's own eth0 MAC first: Step 5 gives every clone new MACs, so set-identity can tell the template apart
cat /sys/class/net/eth0/address | sudo tee /etc/k3s-template-mac
sudo tee /usr/local/sbin/set-identity >/dev/null <<'EOF'
#!/usr/bin/env bash
# Usage: sudo set-identity <hostname> <host-only-ip>     e.g. sudo set-identity k3s-srv-1 192.168.99.21
set -euo pipefail
[[ $# -eq 2 ]] || { echo "usage: sudo set-identity <hostname> <host-only-ip>"; exit 1; }
NAME="$1"; IP="$2"; DOMAIN="home.lab"

# never on the template itself: it still has the eth0 MAC saved in Step 4, every clone has a new one
if [[ "$(cat /sys/class/net/eth0/address)" == "$(cat /etc/k3s-template-mac 2>/dev/null)" ]]; then
  echo "this is kali-template itself (eth0 MAC unchanged) - nothing was changed. Run set-identity in a clone from Step 5" >&2; exit 1
fi

# the name/IP pair must match the home.lab block in /etc/hosts (Step 4, item 6): a typo stops here, before anything changes
WANT=$(awk -v n="$NAME" '/^# BEGIN home.lab/ {f=1; next} /^# END home.lab/ {f=0} f { sub(/#.*/, ""); for (i = 2; i <= NF; i++) if ($i == n) print $1 }' /etc/hosts)
[[ -n "$WANT" ]] || { echo "unknown VM name '$NAME' - it is not in the home.lab block of /etc/hosts" >&2; exit 1; }
[[ "$IP" == "$WANT" ]] || { echo "wrong IP for $NAME: the IP plan says $WANT, you typed $IP" >&2; exit 1; }

hostnamectl set-hostname "$NAME"
sed -i '/^127\.0\.1\.1/d' /etc/hosts
echo "127.0.1.1 ${NAME}.${DOMAIN} ${NAME}" >> /etc/hosts

# NetworkManager keyfiles: eth0 = Adapter 1 (NAT, DHCP, default route), eth1 = Adapter 2 (host-only, static)
mkdir -p /etc/NetworkManager/system-connections
rm -f /etc/NetworkManager/system-connections/*
cat > /etc/NetworkManager/system-connections/nat.nmconnection <<NM
[connection]
id=nat
type=ethernet
interface-name=eth0

[ipv4]
method=auto

[ipv6]
method=ignore
NM
cat > /etc/NetworkManager/system-connections/hostonly.nmconnection <<NM
[connection]
id=hostonly
type=ethernet
interface-name=eth1

[ipv4]
method=manual
address1=${IP}/24
never-default=true
dns-search=${DOMAIN};

[ipv6]
method=disabled
NM
chmod 600 /etc/NetworkManager/system-connections/*.nmconnection

# clones must not share machine-id or SSH host keys
rm -f /etc/machine-id
systemd-machine-id-setup
rm -f /etc/ssh/ssh_host_*
ssh-keygen -A

echo ">>> identity set: $NAME / $IP - rebooting"
reboot
EOF
sudo chmod 755 /usr/local/sbin/set-identity
#    '#!/usr/bin/env bash' must be the very first line, or sh runs the script and it fails. Remove any empty
#    lines a paste put above it, and Windows line endings
sudo sed -i -e 's/\r$//' -e '/./,$!d' /usr/local/sbin/set-identity
```

4. **Run on:** `kali-template`, same SSH session (to reconnect: `ssh -p 2222 kali@127.0.0.1`). Before you clone the template, check that it has everything `set-identity` needs. Every line must say **`OK`**:

```bash
chk() { local out; if out=$(eval "$2" 2>&1); then echo "OK       $1"; else echo "MISSING  $1"; [ -z "$out" ] || printf '%s\n' "$out" | head -n 2 | sed 's/^/         -> /'; fi; }
chk "set-identity installed (755)"     'test -x /usr/local/sbin/set-identity'
chk "set-identity runs under sudo"     'u=$(sudo set-identity 2>&1); [ "$u" = "usage: sudo set-identity <hostname> <host-only-ip>" ] || { echo "$u"; false; }'   # no arguments: only its usage line
chk "set-identity refuses the template" 'grep -q k3s-template-mac /usr/local/sbin/set-identity && [ "$(cat /etc/k3s-template-mac)" = "$(cat /sys/class/net/eth0/address)" ]'
chk "home.lab names in /etc/hosts"     'getent hosts k3s-srv-1 | grep -q "^192.168.99.21 "'
chk "NetworkManager running"           'systemctl is-active --quiet NetworkManager'
chk "NetworkManager keyfile folder"    'sudo test -d /etc/NetworkManager/system-connections'
chk "ifupdown leaves the NICs alone"   '! grep -qE "^[[:space:]]*(auto|allow-hotplug|iface)[[:space:]]+(eth|en)" /etc/network/interfaces'
chk "NIC names eth0 + eth1"            'grep -qw net.ifnames=0 /proc/cmdline && ip link show eth0 && ip link show eth1'
chk "hostnamectl works (D-Bus)"        'hostnamectl status'
chk "machine-id + SSH host-key tools"  'command -v systemd-machine-id-setup && command -v ssh-keygen'
chk "eth1 on the host-only network"    'ip -4 -br addr show eth1 | grep -q " 192\.168\.99\."'
chk "Windows SSH key authorized"       'grep -qE "^(ssh|ecdsa|sk)-" ~/.ssh/authorized_keys'
```

Here is what `set-identity` uses, and which command above provides it:

| `set-identity` needs | Provided by | If the check says `MISSING` |
|---|---|---|
| the script `/usr/local/sbin/set-identity`, mode `755`, with `#!/usr/bin/env bash` as its very first line | part 3, item 7 | *installed* `MISSING`: paste item 7 again, from its `# 7)` line down to and including the `sed` line. *runs under sudo* shows `-> …: [[: not found`: `sh` ran the script because the `#!` line is not line 1, so run item 7's `sed` line |
| the template's own `eth0` MAC in `/etc/k3s-template-mac`, so the script refuses to run on the template (*this is kali-template itself*) | part 3, item 7 | *refuses the template* `MISSING`: paste item 7 again, from its `# 7)` line down to and including the `sed` line |
| `/usr/local/sbin` on sudo's search path (Kali's sudo leaves it out) | part 3, item 1: `/etc/sudoers.d/91-secure-path` | *runs under sudo* shows `-> sudo: set-identity: command not found`: paste item 1 again. A password prompt also means item 1 is missing |
| the `home.lab` block in `/etc/hosts`, for the name/IP check and the lab names | part 3, item 6 | paste item 6 again |
| NetworkManager, which reads the two keyfiles the script writes | part 2: `apt install -y network-manager` | `sudo apt install -y network-manager && sudo systemctl enable --now NetworkManager` |
| `/etc/network/interfaces` holding only `lo`, so ifupdown doesn't grab `eth0` | part 2: the `printf ... /etc/network/interfaces` line | run that line again |
| the NIC names `eth0` / `eth1` that the keyfiles use | part 2: `90-eth-names.cfg` + `update-grub` + reboot | run those lines again, then `sudo reboot` and check again |
| `hostnamectl`, which talks to systemd over D-Bus (`network-manager` pulls D-Bus in) | part 2 | `sudo apt install -y dbus && sudo reboot` |
| `systemd-machine-id-setup` and `ssh-keygen -A` | systemd (base system), `openssh-server` (part 1) | `sudo apt install -y openssh-server` |
| Adapter 2 = host-only, so `eth1` reaches `192.168.99.0/24` | Step 3 (`--nic2 hostonly`), copied by every clone | on Windows, `VBoxManage showvminfo kali-template \| Select-String "NIC 2"` must say *Host-only*; check the Step 2 DHCP server, then `sudo reboot` |
| your Windows SSH key in `~/.ssh/authorized_keys`, which every clone inherits (the Step 6 check, Step 6 option B, and every later `ssh <vm>`) | part 2: the `Get-Content ... id_ed25519.pub \| ssh kali-template ...` line | Run that line again on Windows, in a normal PowerShell window (not ISE), and enter the `kali` password. Then the part 2 check line must print `SSH key login: OK` |

5. When every line says `OK`, clean up and power off. The template is never booted again, unless you have to fix it:

```bash
sudo apt clean
sudo poweroff
```

> **Never** run `set-identity` with arguments on the template itself. The check above calls it without arguments, which only prints its usage line. If you run it on the template anyway, it stops with *this is kali-template itself* and changes nothing. A template built before this guard existed doesn't stop: see *The template got a clone's identity* in Appendix C.
>
> `sudo poweroff` is a Linux command for this SSH session or the VM window. In **Windows** PowerShell it only prints *Sudo is disabled on this machine*. To power the template off from Windows, run `ssh kali-template "sudo poweroff"`.

**Already cloned, and a clone says `sudo: set-identity: command not found`?** The clones were made before part 3 finished. Nothing is on them yet, so delete them and fix the template. **Run on:** Windows host:

```powershell
foreach ($vm in "k3s-lb", "k3s-db", "k3s-srv-1", "k3s-srv-2", "k3s-agent-1", "k3s-agent-2", "k3s-agent-3") {
    VBoxManage controlvm $vm poweroff 2>$null
    VBoxManage unregistervm $vm --delete
}
# make sure the template has exactly the Step 3 port forward (127.0.0.1:2222 -> 22)
VBoxManage modifyvm kali-template --natpf1 delete ssh 2>$null
VBoxManage modifyvm kali-template --natpf1 "ssh,tcp,127.0.0.1,2222,,22"
VBoxManage startvm kali-template --type gui
```

Then:

1. Wait for the login prompt in the template window. If SSH isn't on yet (`systemctl is-active ssh` doesn't print `active`), type part 1 there.
2. On Windows, run the PowerShell block of part 2 again (it is safe to run twice). It adds the `kali-template` shortcut if it is missing, copies your SSH key and connects. The `eth1` address doesn't matter, because the connection goes through `127.0.0.1:2222`.
3. Run part 3 again (it is safe to run twice), then the check in part 4 and the power-off in part 5.
4. Repeat Step 5.

---

## Step 5 — Windows host: clone the 7 cluster VMs

**Run on:** Windows host. Keep this PowerShell window open: Steps 5, 6 and 13 reuse `$nodes`.

```powershell
$nodes = @(
    [pscustomobject]@{ Name = "k3s-lb";      Cpus = 1; MemMB = 1024; Ip = "192.168.99.10"; NatPort = 22010 }
    [pscustomobject]@{ Name = "k3s-db";      Cpus = 2; MemMB = 2048; Ip = "192.168.99.11"; NatPort = 22011 }
    [pscustomobject]@{ Name = "k3s-srv-1";   Cpus = 2; MemMB = 4096; Ip = "192.168.99.21"; NatPort = 22021 }
    [pscustomobject]@{ Name = "k3s-srv-2";   Cpus = 2; MemMB = 4096; Ip = "192.168.99.22"; NatPort = 22022 }
    [pscustomobject]@{ Name = "k3s-agent-1"; Cpus = 4; MemMB = 4096; Ip = "192.168.99.31"; NatPort = 22031 }
    [pscustomobject]@{ Name = "k3s-agent-2"; Cpus = 4; MemMB = 4096; Ip = "192.168.99.32"; NatPort = 22032 }
    [pscustomobject]@{ Name = "k3s-agent-3"; Cpus = 4; MemMB = 4096; Ip = "192.168.99.33"; NatPort = 22033 }
)   # NatPort = 22000 + last octet of the IP: the clone's own SSH port forward, 127.0.0.1:<NatPort> -> its port 22

# kali-template must be powered off: a running VM keeps its disk locked ("Failed to lock source media")
$state = "$((VBoxManage showvminfo kali-template --machinereadable) -match '^VMState=')"
if ($state -ne 'VMState="poweroff"') {
    Write-Warning "kali-template is not powered off ($state), so NOTHING was cloned - don't start Step 6. Power it off from this window with: ssh kali-template 'sudo poweroff'  (or type 'sudo poweroff' in the kali-template VM window; in PowerShell it doesn't work). Wait until its window closes, then run this block again."
} else {
    $existing = VBoxManage list vms
    $ok = $true
    foreach ($n in $nodes) {
        if ($existing -match "^`"$($n.Name)`" ") { "$($n.Name) already exists - skipped"; continue }
        VBoxManage clonevm "kali-template" --name $n.Name --groups "/k3s-ha" --register
        if ($LASTEXITCODE -ne 0) { Write-Warning "Cloning $($n.Name) failed - see the error above"; $ok = $false; break }
        VBoxManage modifyvm $n.Name --cpus $n.Cpus --memory $n.MemMB
    }
    if ($ok) {
        foreach ($n in $nodes) {
            # replace the inherited 127.0.0.1:2222 rule (kali-template's) with this clone's own port
            VBoxManage modifyvm $n.Name --natpf1 delete ssh 2>$null
            VBoxManage modifyvm $n.Name --natpf1 "ssh,tcp,127.0.0.1,$($n.NatPort),,22"
            VBoxManage startvm $n.Name --type gui
        }
        $nodes | ForEach-Object { "{0,-12} ssh -p {1} kali@127.0.0.1" -f $_.Name, $_.NatPort }
    }
}
```

> The block refuses to clone while `kali-template` is still running, stops at the first failed clone, and skips clones that already exist, so you can safely run it again. Run it while the clones are powered off: a running VM can't change its port forward, and VirtualBox then reports that the machine is locked.

**Success looks like:** 7 new VM windows, and the block ends with 7 lines such as `k3s-lb       ssh -p 22010 kali@127.0.0.1`. If it printed a **WARNING** instead, nothing was cloned, so don't go on to Step 6. The only VM running is then still `kali-template`, and a Step 6 line typed there would turn the template into that node. For *kali-template is not powered off*, run this in the same window, then run the block again:

```powershell
ssh kali-template "sudo poweroff"     # "Connection ... closed by remote host" is expected
while (-not ((VBoxManage showvminfo kali-template --machinereadable) -match '^VMState="poweroff"')) { Start-Sleep 2 }
"kali-template is powered off"
```

**Port forwards.** Each clone inherits the template's rule `127.0.0.1:2222 → 22`, and only one VM at a time can hold a host port. So the block swaps it for the clone's own port, `NatPort` from `$nodes`. The ports follow the IP plan: `22000` + the last number of the IP, so `k3s-lb` (`.10`) answers on `127.0.0.1:22010` and `k3s-agent-3` (`.33`) on `22033`. The template keeps `2222` for itself.

| VM | `NatPort` (`ssh -p <port> kali@127.0.0.1`) |
|---|---|
| `kali-template` | `2222` |
| `k3s-lb` / `k3s-db` | `22010` / `22011` |
| `k3s-srv-1` / `k3s-srv-2` / `k3s-srv-3` (App. A) | `22021` / `22022` / `22023` |
| `k3s-agent-1` / `-2` / `-3` | `22031` / `22032` / `22033` |
| `k3s-lb-1` / `k3s-lb-2` (App. B) | `22008` / `22009` |

The ports go through each VM's NAT NIC (`eth0`), so they work before Step 6 gives the clones their host-only IPs, and later as well whenever the host-only network is broken (Appendix C). Every later step that runs inside a clone starts with that clone's `ssh -p <NatPort> kali@127.0.0.1` line.

`clonevm` makes full clones with **new MAC addresses**. All 7 VMs are grouped under **k3s-ha** in VirtualBox Manager.

> `kali` is **not** cloned here, because a clone would inherit the template's headless install. [Appendix E](#appendix-e--kali-linux-workstation-vm) installs it from the same ISO and ticks **Xfce** on the Software selection screen. That VM joins the **k3s-ha** group as well.

---

## Step 6 — Console or Windows: give each clone its identity

Every clone currently believes it is `kali-template`. Each one needs one `sudo set-identity <name> <ip>` line, and the VM then reboots by itself.

**First, check that Step 5 really cloned.** **Run on:** Windows host, in the PowerShell window that still has `$nodes` from Step 5:

```powershell
$running = VBoxManage list runningvms
foreach ($n in $nodes) {
    $s = if ($running -match "^`"$($n.Name)`" ") { 'running' } else { 'NOT RUNNING - run Step 5 again (no clone, or not started)' }
    "{0,-12} {1}" -f $n.Name, $s
}
if ($running -match '^"kali-template" ') { Write-Warning "kali-template is running. Power it off first (ssh kali-template 'sudo poweroff') and never type a Step 6 line in its window" }
```

All 7 lines must say `running`, with no warning. Then use **one** of the two options:

- **Option A:** type the line in each VM window.
- **Option B:** run one PowerShell loop on Windows, through the clones' port forwards from Step 5. Nothing to type in the VM windows.

**Option B — Run on:** Windows host, in the PowerShell window that still has `$nodes` from Step 5. Wait until all 7 VM windows show a login prompt:

```powershell
# each clone through its own port forward 127.0.0.1:<NatPort> (Step 5), so each line reaches the right VM
foreach ($n in $nodes) {
    "== $($n.Name)   127.0.0.1:$($n.NatPort)"
    ssh -p $n.NatPort -o HostKeyAlias=kali-template -o BatchMode=yes -o ConnectTimeout=10 kali@127.0.0.1 "sudo set-identity $($n.Name) $($n.Ip)"
}
```

Each VM prints `>>> identity set: <name> / <ip> - rebooting`, followed by *Connection to 127.0.0.1 closed by remote host*. That message is expected, because the VM reboots.

**A VM without a `>>> identity set` line changed nothing.** It was usually still booting. Run the loop again once every window shows a login prompt. VMs that already have their identity are refused with *REMOTE HOST IDENTIFICATION HAS CHANGED*, because their new host key no longer matches `kali-template`, so running the loop twice is safe.
- **`HostKeyAlias=kali-template`:** until `set-identity` runs, every clone still has the template's SSH host key, which part 2 of Step 4 saved under that name. So a clone that answers proves it is still fresh.
- **`BatchMode=yes`:** never asks for a password.
- ***REMOTE HOST IDENTIFICATION HAS CHANGED*** for one VM: that clone already has its own identity, and nothing was changed on it.
- ***Permission denied (publickey,…)***: your SSH key isn't on the template (Step 4 part 4, *Windows SSH key authorized*). Use option A.
- ***Connection reset***, *kex_exchange_identification* or a timeout: the VM is still booting, or its port forward is missing (`VBoxManage showvminfo <vm> --machinereadable | Select-String '^Forwarding'`).

**Option A — In each VM window**, log in as `kali` and type the matching line. Or connect from Windows with the `ssh -p` line from the same row and paste the line there. The port decides which VM answers, so always take both from the same row. The session then closes with *Connection to 127.0.0.1 closed by remote host*, because the VM reboots.

> The prompt says `kali@kali-template` in **every** clone, and in the template itself, so go by the VirtualBox **window title** (for example *k3s-srv-1 [Running]*) to know which VM you are in. A window titled *kali-template* is the template: never type these lines there. `set-identity` refuses there anyway, with *this is kali-template itself*. `set-identity` checks the name/IP pair against the `home.lab` block from Step 4. A wrong pair such as `k3s-srv-1 192.168.99.22` stops with the IP the plan expects, before anything is changed. The Windows check below also catches a correct line typed into the wrong window.

| VM window | Or connect from Windows | Type or paste this |
|---|---|---|
| `k3s-lb` | `ssh -p 22010 kali@127.0.0.1` | `sudo set-identity k3s-lb 192.168.99.10` |
| `k3s-db` | `ssh -p 22011 kali@127.0.0.1` | `sudo set-identity k3s-db 192.168.99.11` |
| `k3s-srv-1` | `ssh -p 22021 kali@127.0.0.1` | `sudo set-identity k3s-srv-1 192.168.99.21` |
| `k3s-srv-2` | `ssh -p 22022 kali@127.0.0.1` | `sudo set-identity k3s-srv-2 192.168.99.22` |
| `k3s-agent-1` | `ssh -p 22031 kali@127.0.0.1` | `sudo set-identity k3s-agent-1 192.168.99.31` |
| `k3s-agent-2` | `ssh -p 22032 kali@127.0.0.1` | `sudo set-identity k3s-agent-2 192.168.99.32` |
| `k3s-agent-3` | `ssh -p 22033 kali@127.0.0.1` | `sudo set-identity k3s-agent-3 192.168.99.33` |

You can then close the VM windows and choose **"Continue running in the background"**.

**Run on:** Windows host. Add SSH shortcuts (only once), forget the old host keys for these IPs and port forwards (`set-identity` gave every clone new ones), then check reachability:

```powershell
if (-not (Select-String -Path "$HOME\.ssh\config" -Pattern '^Host k3s-lb$' -Quiet)) {
@'

Host k3s-lb
    HostName 192.168.99.10
Host k3s-db
    HostName 192.168.99.11
Host k3s-srv-1
    HostName 192.168.99.21
Host k3s-srv-2
    HostName 192.168.99.22
Host k3s-agent-1
    HostName 192.168.99.31
Host k3s-agent-2
    HostName 192.168.99.32
Host k3s-agent-3
    HostName 192.168.99.33
# Kali workstation, installed in Appendix E
Host kali
    HostName 192.168.99.50
# only used by Appendix A / B
Host k3s-srv-3
    HostName 192.168.99.23
Host k3s-lb-1
    HostName 192.168.99.8
Host k3s-lb-2
    HostName 192.168.99.9
Host k3s-* kali
    User kali
    StrictHostKeyChecking accept-new
'@ | Add-Content -Encoding ascii "$HOME\.ssh\config"
}
# new identity = new host keys. Keys saved earlier (an earlier build, or ssh -p <NatPort> before set-identity) would stop ssh
# with "REMOTE HOST IDENTIFICATION HAS CHANGED": forget them for the host-only IP and for the port forward 127.0.0.1:<NatPort>
foreach ($n in $nodes) { ssh-keygen -R $n.Ip 2>$null | Out-Null; ssh-keygen -R "[127.0.0.1]:$($n.NatPort)" 2>$null | Out-Null }

# For each VM in $nodes: whatever answers at its planned IP must report the planned FQDN, exactly that IP
# on eth1, and the MAC of THIS VirtualBox VM's NIC 2 (that proves the line went into the right window)
foreach ($n in $nodes) {
    $mac = ((VBoxManage showvminfo $n.Name --machinereadable | Select-String '^macaddress2=').Line -split '"')[1]
    $mac = ($mac -replace '(..)(?=.)', '$1:').ToLower()          # 080027AABBCC -> 08:00:27:aa:bb:cc
    $out = @(ssh -o ConnectTimeout=5 $n.Name 'hostname -f; cat /sys/class/net/eth1/address; ip -4 -br addr show eth1')
    $ip  = if ($out.Count -eq 3) { ((-split $out[2]) | Select-Object -Skip 2) -join ' ' } else { '' }
    if ($out.Count -eq 0) { $res = 'NO ANSWER' }
    elseif ($out[0] -eq "$($n.Name).home.lab" -and $out[1] -eq $mac -and $ip -eq "$($n.Ip)/24") { $res = 'OK' }
    else { $res = 'MISMATCH  got: ' + (($out -join ' | ') -replace '\s{2,}', ' ') }
    "{0,-12} {1,-14} {2}" -f $n.Name, $n.Ip, $res
}
```

Every line must end in **`OK`**:

```text
k3s-lb       192.168.99.10  OK
k3s-db       192.168.99.11  OK
k3s-srv-1    192.168.99.21  OK
k3s-srv-2    192.168.99.22  OK
k3s-agent-1  192.168.99.31  OK
k3s-agent-2  192.168.99.32  OK
k3s-agent-3  192.168.99.33  OK
```

- **`MISMATCH`** shows what really answered at that IP.
  - A different hostname or MAC means the line was typed into another VM's window. Run the correct line again in the window whose **title** matches.
  - Two addresses on `eth1` mean the keyfile didn't take over (see Appendix C).
- **`NO ANSWER`** (*ssh: connect to host 192.168.99.x port 22: Connection timed out*): nothing holds that IP yet. Most often `set-identity` hasn't run on that VM, so its `eth1` still has the template's setting. Otherwise the VM is still rebooting, or NetworkManager ignored the keyfile. Ask each VM through its NAT port forward, which works without `eth1`:

  ```powershell
  foreach ($n in $nodes) { "{0,-12} {1}" -f $n.Name, ((ssh -p $n.NatPort -o BatchMode=yes -o ConnectTimeout=5 -o StrictHostKeyChecking=no -o UserKnownHostsFile=NUL -o LogLevel=ERROR kali@127.0.0.1 'hostname; ip -4 -br addr show eth1') -join ' | ') }
  ```

  The second column must start with the VM's own name. `kali-template`, or another VM's name, means `set-identity` never ran there: run option B again, or type that VM's line in its window. The right name with no IP on `eth1`: see the Appendix C row *A VM has no `192.168.99.x`*. Then run the **whole** check block again, including its `ssh-keygen -R` line, because the new identity comes with a new host key.

Optional but highly recommended: take a snapshot so you can roll back any later step.

```powershell
foreach ($n in $nodes) { VBoxManage snapshot $n.Name take "01-identity" --live }
```

---

## Step 7 — k3s-db: MariaDB datastore and secrets

**Run on:** `k3s-db`. Connect from Windows, then paste the `bash` blocks below into that session:

```powershell
ssh -p 22011 kali@127.0.0.1      # k3s-db   (or: ssh k3s-db)
```

```bash
# Install MariaDB and listen on the network (not only 127.0.0.1)
sudo apt update && sudo apt install -y mariadb-server
sudo sed -i 's/^bind-address\s*=.*/bind-address            = 0.0.0.0/' /etc/mysql/mariadb.conf.d/50-server.cnf
# [client] ssl = 0: MariaDB 11.4+ tools use TLS even on the local socket, and on Kali the server's built-in TLS
#   fails at random ("ERROR 2026 ... tlsv1 alert internal error"). K3s connects without TLS anyway.
# max_connections: two K3s servers open more than the default 151 connections while they start ("Error 1040")
sudo tee /etc/mysql/mariadb.conf.d/90-k3s.cnf >/dev/null <<'EOF'
[client]
ssl = 0

[mysqld]
max_connections = 500
EOF
sudo systemctl enable mariadb
sudo systemctl restart mariadb
sudo ss -tlnp | grep 3306          # expect 0.0.0.0:3306
sudo mariadb -N -e "SELECT @@max_connections"   # expect 500 (and no ERROR 2026)

# Harden. Answers: [Enter] (no root password), n (unix_socket already), n (keep socket auth),
#                  Y (remove anonymous), Y (no remote root), Y (drop test DB), Y (reload)
sudo mariadb-secure-installation
```

Generate the two cluster secrets and **keep a copy in your password manager**. The K3s token also encrypts the bootstrap data stored in the database; without it you cannot add servers or restore.

```bash
# generated only ONCE: once a K3s server has started, the cluster is bound to this token and this DB password
if [ -s ~/k3s-secrets.env ]; then
  echo "~/k3s-secrets.env already exists - keeping it (new secrets would lock the nodes out of the running cluster)"
else
  DB_PASS=$(openssl rand -hex 24)
  K3S_TOKEN=$(openssl rand -hex 32)
  printf 'DB_PASS=%s\nK3S_TOKEN=%s\n' "$DB_PASS" "$K3S_TOKEN" > ~/k3s-secrets.env
fi
grep K3S_TOKEN ~/k3s-secrets.env > ~/k3s-agent.env          # agents need only the token
chmod 600 ~/k3s-secrets.env ~/k3s-agent.env
cat ~/k3s-secrets.env
```

Create the database and a user that may connect **only from the two servers**:

```bash
source ~/k3s-secrets.env
sudo mariadb <<SQL
CREATE DATABASE IF NOT EXISTS k3s;
CREATE USER IF NOT EXISTS 'k3s'@'192.168.99.21' IDENTIFIED BY '${DB_PASS}';
CREATE USER IF NOT EXISTS 'k3s'@'192.168.99.22' IDENTIFIED BY '${DB_PASS}';
GRANT ALL PRIVILEGES ON k3s.* TO 'k3s'@'192.168.99.21';
GRANT ALL PRIVILEGES ON k3s.* TO 'k3s'@'192.168.99.22';
FLUSH PRIVILEGES;
SQL
sudo mariadb -e "SELECT user, host FROM mysql.user WHERE user='k3s';"
```

**Run on:** Windows host. Hand the secrets to the servers (full file) and the agents (token only):

```powershell
scp k3s-db:k3s-secrets.env "$env:TEMP\k3s-secrets.env"
scp k3s-db:k3s-agent.env   "$env:TEMP\k3s-agent.env"
foreach ($h in "k3s-srv-1", "k3s-srv-2")                    { scp "$env:TEMP\k3s-secrets.env" "${h}:k3s-secrets.env" }
foreach ($h in "k3s-agent-1", "k3s-agent-2", "k3s-agent-3") { scp "$env:TEMP\k3s-agent.env"   "${h}:k3s-agent.env" }
Remove-Item "$env:TEMP\k3s-secrets.env", "$env:TEMP\k3s-agent.env"
```

---

## Step 8 — k3s-lb: NGINX TCP load balancer

This VM is the single entry point. It forwards **:6443** to the K3s servers, and **:80/:443** to Traefik on the agents.

**Run on:** `k3s-lb`. Connect from Windows, then paste the block below:

```powershell
ssh -p 22010 kali@127.0.0.1      # k3s-lb   (or: ssh k3s-lb)
```

```bash
sudo apt update && sudo apt install -y nginx libnginx-mod-stream
sudo cp /etc/nginx/nginx.conf /etc/nginx/nginx.conf.orig

sudo tee /etc/nginx/nginx.conf >/dev/null <<'EOF'
user www-data;
worker_processes auto;
pid /run/nginx.pid;
error_log /var/log/nginx/error.log;
include /etc/nginx/modules-enabled/*.conf;     # loads the stream (L4) module

events {
    worker_connections 1024;
}

stream {
    log_format k3s '$remote_addr -> $upstream_addr [$time_local] $protocol $status $session_time';
    access_log /var/log/nginx/k3s-stream.log k3s;

    # ---------- Kubernetes API -> K3s servers ----------
    upstream k3s_api {
        server 192.168.99.21:6443 max_fails=3 fail_timeout=10s;
        server 192.168.99.22:6443 max_fails=3 fail_timeout=10s;
    }
    server {
        listen 6443;
        proxy_pass k3s_api;
        proxy_connect_timeout 2s;
        proxy_timeout 1h;              # long-lived watches / logs -f / agent tunnels
    }

    # ---------- Ingress HTTP -> Traefik on the agents ----------
    upstream k3s_http {
        server 192.168.99.31:80 max_fails=3 fail_timeout=10s;
        server 192.168.99.32:80 max_fails=3 fail_timeout=10s;
        server 192.168.99.33:80 max_fails=3 fail_timeout=10s;
    }
    server {
        listen 80;
        proxy_pass k3s_http;
        proxy_connect_timeout 2s;
    }

    # ---------- Ingress HTTPS (TLS passthrough) -> Traefik on the agents ----------
    upstream k3s_https {
        server 192.168.99.31:443 max_fails=3 fail_timeout=10s;
        server 192.168.99.32:443 max_fails=3 fail_timeout=10s;
        server 192.168.99.33:443 max_fails=3 fail_timeout=10s;
    }
    server {
        listen 443;
        proxy_pass k3s_https;
        proxy_connect_timeout 2s;
    }
}
EOF

sudo nginx -t
sudo systemctl enable nginx
sudo systemctl restart nginx
sudo ss -tlnp | grep nginx          # expect :6443, :80 and :443
```

> This is the video's "load balancer in front of the servers" piece. Upstreams that are not up yet are simply skipped; that is fine until Steps 9–11 are done.

---

## Step 9 — k3s-srv-1: first K3s server

**Run on:** `k3s-srv-1`. Connect from Windows, then paste the block below:

```powershell
ssh -p 22021 kali@127.0.0.1      # k3s-srv-1   (or: ssh k3s-srv-1)
```

```bash
source ~/k3s-secrets.env
NODE_IP=$(ip -4 -o addr show eth1 | awk '{print $4}' | cut -d/ -f1); echo "node-ip: $NODE_IP"
nc -zv 192.168.99.11 3306                        # DB reachable?

sudo mkdir -p /etc/rancher/k3s
sudo tee /etc/rancher/k3s/config.yaml >/dev/null <<EOF
token: "${K3S_TOKEN}"
datastore-endpoint: "mysql://k3s:${DB_PASS}@tcp(192.168.99.11:3306)/k3s"
node-ip: "${NODE_IP}"
flannel-iface: "eth1"
node-taint:
  - "CriticalAddonsOnly=true:NoExecute"
tls-san:
  - "192.168.99.10"
  - "k3s.home.lab"
EOF
sudo chmod 600 /etc/rancher/k3s/config.yaml

# Optional: pin a version (same on ALL nodes), e.g. INSTALL_K3S_VERSION="v1.xx.y+k3s1"
curl -sfL https://get.k3s.io | sh -s - server

sudo systemctl status k3s --no-pager | head -5
sudo k3s kubectl get nodes -o wide       # k3s-srv-1  Ready  control-plane  INTERNAL-IP 192.168.99.21
```

What each setting does:

| Setting | Why |
|---|---|
| `token` | Shared secret. It must be identical on every server (and agents use it to join). |
| `datastore-endpoint` | Use MariaDB on `k3s-db` instead of embedded SQLite or etcd. This is what makes 2 servers HA. |
| `node-ip` / `flannel-iface` | Force the host-only NIC; otherwise K3s picks the NAT NIC (`10.0.2.15`). |
| `node-taint CriticalAddonsOnly=true:NoExecute` | Dedicated control plane. Your apps (and Argo CD) run only on the agents. |
| `tls-san` | Adds the LB IP and DNS name to the API certificate, so kubectl through `192.168.99.10` has no x509 errors. |

> **The same thing as one-liner flags, the way the video types it:**
> ```bash
> curl -sfL https://get.k3s.io | sh -s - server \
>   --token=YOUR-SECRET \
>   --datastore-endpoint='mysql://user:pass@tcp(ipaddress:3306)/dbname' \
>   --node-taint CriticalAddonsOnly=true:NoExecute \
>   --tls-san your-dns-name --tls-san your-lb-ip-address
> ```
> The config file gives the same result. On VirtualBox it also carries `node-ip` and `flannel-iface`, keeps the DB password out of `ps` and the systemd unit, and survives upgrades (Step 19).

> It is normal for some `kube-system` pods to stay **Pending** now: the servers are tainted, and no agent has joined yet.

---

## Step 10 — k3s-srv-2: second K3s server

**Run on:** `k3s-srv-2`. These are **exactly the same commands** as Step 9; `NODE_IP` detects `.22` by itself. Connect from Windows, then paste the block below:

```powershell
ssh -p 22022 kali@127.0.0.1      # k3s-srv-2   (or: ssh k3s-srv-2)
```

```bash
source ~/k3s-secrets.env
NODE_IP=$(ip -4 -o addr show eth1 | awk '{print $4}' | cut -d/ -f1); echo "node-ip: $NODE_IP"
nc -zv 192.168.99.11 3306

sudo mkdir -p /etc/rancher/k3s
sudo tee /etc/rancher/k3s/config.yaml >/dev/null <<EOF
token: "${K3S_TOKEN}"
datastore-endpoint: "mysql://k3s:${DB_PASS}@tcp(192.168.99.11:3306)/k3s"
node-ip: "${NODE_IP}"
flannel-iface: "eth1"
node-taint:
  - "CriticalAddonsOnly=true:NoExecute"
tls-san:
  - "192.168.99.10"
  - "k3s.home.lab"
EOF
sudo chmod 600 /etc/rancher/k3s/config.yaml

curl -sfL https://get.k3s.io | sh -s - server

sudo k3s kubectl get nodes -o wide       # now BOTH servers are Ready
```

**Run on:** `k3s-db`. Confirm that the cluster state now lives in MariaDB. Connect from Windows, then run the check:

```powershell
ssh -p 22011 kali@127.0.0.1      # k3s-db   (or: ssh k3s-db)
```

```bash
sudo mariadb -e "SELECT COUNT(*) AS kine_rows FROM k3s.kine;"
```

**Run on:** `k3s-lb`. Confirm that the API answers through the load balancer (HTTP `401` is the expected answer here). Connect from Windows, then run the check:

```powershell
ssh -p 22010 kali@127.0.0.1      # k3s-lb   (or: ssh k3s-lb)
```

```bash
curl -sk -o /dev/null -w '%{http_code}\n' https://127.0.0.1:6443/version
```

---

## Step 11 — k3s-agent-1/2/3: join the workers

**Run on:** `k3s-agent-1`, then `k3s-agent-2`, then `k3s-agent-3`: the identical block on each. Connect to one agent from Windows, paste the block, type `exit`, then connect to the next one:

| VM | Connect from Windows |
|---|---|
| `k3s-agent-1` | `ssh -p 22031 kali@127.0.0.1`   (or: `ssh k3s-agent-1`) |
| `k3s-agent-2` | `ssh -p 22032 kali@127.0.0.1`   (or: `ssh k3s-agent-2`) |
| `k3s-agent-3` | `ssh -p 22033 kali@127.0.0.1`   (or: `ssh k3s-agent-3`) |

```bash
source ~/k3s-agent.env
NODE_IP=$(ip -4 -o addr show eth1 | awk '{print $4}' | cut -d/ -f1); echo "node-ip: $NODE_IP"
nc -zv 192.168.99.10 6443                        # LB reachable?

sudo mkdir -p /etc/rancher/k3s
sudo tee /etc/rancher/k3s/config.yaml >/dev/null <<EOF
server: "https://192.168.99.10:6443"
token: "${K3S_TOKEN}"
node-ip: "${NODE_IP}"
flannel-iface: "eth1"
EOF
sudo chmod 600 /etc/rancher/k3s/config.yaml

curl -sfL https://get.k3s.io | sh -s - agent

sudo systemctl status k3s-agent --no-pager | head -5
```

**Run on:** `k3s-srv-1`. Give the workers a role label and check the whole cluster. Connect from Windows, then paste the block below:

```powershell
ssh -p 22021 kali@127.0.0.1      # k3s-srv-1   (or: ssh k3s-srv-1)
```

```bash
for n in k3s-agent-1 k3s-agent-2 k3s-agent-3; do
  sudo k3s kubectl label node "$n" node-role.kubernetes.io/worker=worker --overwrite
done
sudo k3s kubectl get nodes -o wide
sudo k3s kubectl get pods -A -o wide       # coredns, traefik, svclb-*, metrics-server: Running
```

Expected:

```text
NAME          STATUS   ROLES           INTERNAL-IP
k3s-agent-1   Ready    worker          192.168.99.31
k3s-agent-2   Ready    worker          192.168.99.32
k3s-agent-3   Ready    worker          192.168.99.33
k3s-srv-1     Ready    control-plane   192.168.99.21
k3s-srv-2     Ready    control-plane   192.168.99.22
```

Still on `k3s-srv-1`, prepare a kubeconfig copy for Windows:

```bash
sudo cp /etc/rancher/k3s/k3s.yaml ~/k3s.yaml && sudo chown kali: ~/k3s.yaml
```

---

## Step 12 — Windows host: kubectl through the load balancer

**Run on:** Windows host.

```powershell
New-Item -ItemType Directory -Force "$HOME\.kube" | Out-Null
if (Test-Path "$HOME\.kube\config") { Copy-Item "$HOME\.kube\config" "$HOME\.kube\config.bak" -Force }

scp k3s-srv-1:k3s.yaml "$HOME\.kube\config"
ssh k3s-srv-1 "rm -f ~/k3s.yaml"

# point kubectl at the LOAD BALANCER, not at a single server
(Get-Content "$HOME\.kube\config") -replace '127\.0\.0\.1', '192.168.99.10' | Set-Content -Encoding ascii "$HOME\.kube\config"
kubectl config rename-context default k3s-ha

kubectl get nodes -o wide
kubectl cluster-info          # Kubernetes control plane is running at https://192.168.99.10:6443
```

> Was there an older `config.bak` (e.g. from Docker Desktop)? You can merge it later with `KUBECONFIG` and `kubectl config view --flatten`.

---

## Step 13 — Windows host: test HA failover

**Run on:** Windows host.

```powershell
# 1) kill server #1 hard
VBoxManage controlvm k3s-srv-1 poweroff
kubectl get nodes                 # still answers (NGINX -> k3s-srv-2); srv-1 turns NotReady after ~40 s

# 2) bring it back, then kill server #2
VBoxManage startvm k3s-srv-1 --type headless
kubectl get nodes -w              # wait for k3s-srv-1 Ready, Ctrl+C
VBoxManage controlvm k3s-srv-2 poweroff
kubectl get nodes                 # still answers via k3s-srv-1
VBoxManage startvm k3s-srv-2 --type headless
```

> **Single points of failure in the video's design:** `k3s-db` and `k3s-lb`. If either one is down, the API is down (running pods keep running). Appendix A removes the DB from the picture (embedded etcd); Appendix B makes the LB redundant (keepalived VIP).

Snapshot point:

```powershell
foreach ($n in $nodes) { VBoxManage snapshot $n.Name take "02-k3s-ha" --live }
```

---

## Step 14 — Windows host: install Argo CD

**Run on:** Windows host.

```powershell
kubectl create namespace argocd
kubectl apply -n argocd --server-side --force-conflicts -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

kubectl -n argocd wait --for=condition=Ready pod --all --timeout=600s
kubectl -n argocd get pods -o wide        # all pods land on k3s-agent-* (the servers are tainted)
```

- `--server-side --force-conflicts` is the currently documented way. Some Argo CD CRDs are too large for client-side apply.
- **To pin a version** (recommended for anything long-lived), replace `stable` in the URL with a release tag from <https://github.com/argoproj/argo-cd/releases>.
- **To run Argo CD in HA mode**, use `.../manifests/ha/install.yaml` instead. It needs **3 schedulable nodes**, which the three agents provide.

Quick smoke test before exposing it. Open <https://localhost:8080> (self-signed certificate warning), then press Ctrl+C:

```powershell
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

---

## Step 15 — Windows host: expose Argo CD through Traefik

K3s ships **Traefik** as its ingress controller. The path is browser → `k3s-lb:443` → Traefik on an agent → `argocd-server`. Traefik terminates TLS, so `argocd-server` must serve plain HTTP behind it (`server.insecure`). Without that you get a redirect loop.

**Run on:** Windows host.

```powershell
# 1) argocd-server: plain HTTP behind the ingress
@'
data:
  server.insecure: "true"
'@ | Set-Content -Encoding ascii "$env:TEMP\argocd-insecure.yaml"
kubectl -n argocd patch configmap argocd-cmd-params-cm --type merge --patch-file "$env:TEMP\argocd-insecure.yaml"
kubectl -n argocd rollout restart deployment argocd-server
kubectl -n argocd rollout status deployment argocd-server

# 2) Ingress: https://argocd.home.lab  (Traefik's built-in self-signed cert)
@'
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd-server
  namespace: argocd
  annotations:
    traefik.ingress.kubernetes.io/router.entrypoints: websecure
    traefik.ingress.kubernetes.io/router.tls: "true"
spec:
  ingressClassName: traefik
  rules:
    - host: argocd.home.lab
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: argocd-server
                port:
                  number: 80
'@ | kubectl apply -f -
kubectl -n argocd get ingress
```

**Run on:** Windows host, in an **Administrator** PowerShell. Add the DNS names:

```powershell
Add-Content -Path "$env:SystemRoot\System32\drivers\etc\hosts" -Value "`r`n192.168.99.10  k3s.home.lab argocd.home.lab whoami.home.lab"
ipconfig /flushdns
```

Test it from a normal PowerShell (`curl.exe` is the real curl, not the PowerShell alias):

```powershell
curl.exe -skI https://argocd.home.lab | Select-Object -First 1      # HTTP/1.1 200 OK
```

Open **<https://argocd.home.lab>** and accept the self-signed certificate. Use `https://`, because the route is attached only to the `websecure` entrypoint.

> For a trusted certificate later, add **cert-manager** with a Let's Encrypt DNS-01 issuer, or your own CA, and put a `tls:` block on the Ingress.

---

## Step 16 — Windows host: Argo CD CLI login and admin password

**Run on:** Windows host.

```powershell
argocd admin initial-password -n argocd                       # prints the generated admin password
argocd login argocd.home.lab --username admin --grpc-web --insecure
argocd account update-password                                # set your own password
kubectl -n argocd delete secret argocd-initial-admin-secret   # no longer needed

argocd cluster list       # https://kubernetes.default.svc   in-cluster   Successful
```

- `--grpc-web` makes the CLI work through Traefik over HTTP/1.1.
- `--insecure` is needed because of the self-signed certificate.

The web UI uses the same login: user `admin` and your new password.

---

## Step 17 — Windows host: first Argo CD app (guestbook)

This is a zero-setup sanity check that uses the official example repo.

**Run on:** Windows host.

```powershell
@'
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: guestbook
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/argoproj/argocd-example-apps.git
    targetRevision: HEAD
    path: guestbook
  destination:
    server: https://kubernetes.default.svc
    namespace: guestbook
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
'@ | kubectl apply -f -

argocd app get guestbook                  # Sync Status: Synced, Health: Healthy
kubectl -n guestbook get pods -o wide
```

Clean up when you're done: `argocd app delete guestbook --yes`.

---

## Step 18 — Windows host and GitHub: your own GitOps repo (app-of-apps)

From here on, **Git is the source of truth**. You push YAML, and Argo CD makes the cluster match it.

```mermaid
sequenceDiagram
    participant You as You (Windows)
    participant GH as GitHub repo
    participant ACD as Argo CD (in cluster)
    participant K8s as K3s API (via k3s-lb)
    You->>GH: git push (change manifests)
    loop every ~3 min (or Refresh button)
        ACD->>GH: fetch desired state
    end
    ACD->>ACD: diff desired vs live
    ACD->>K8s: apply / prune (auto-sync)
    Note over ACD,K8s: selfHeal reverts manual kubectl changes
```

Repository layout:

```text
homelab-gitops/
├── apps/
│   └── whoami/
│       └── whoami.yaml          # Deployment + Service + Ingress
└── argocd/
    ├── root.yaml                # the ONE app you apply by hand (app-of-apps)
    └── apps/
        └── whoami.yaml          # Argo CD Application -> apps/whoami
```

**Run on:** Windows host. First create an **empty public** repo named `homelab-gitops` on github.com (no README), then:

```powershell
$GhUser = "Saveit29"          # <-- change me
$Repo   = "C:\Users\USER\Desktop\HA_K3S_ARGOCD\ArgoCD-K3S-Setup-Repo"
New-Item -ItemType Directory -Force "$Repo\apps\whoami", "$Repo\argocd\apps" | Out-Null
Set-Location $Repo

# --- the app itself --------------------------------------------------------
@'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: whoami
spec:
  replicas: 3
  selector:
    matchLabels:
      app: whoami
  template:
    metadata:
      labels:
        app: whoami
    spec:
      containers:
        - name: whoami
          image: traefik/whoami        # pin a version tag for real use
          ports:
            - containerPort: 80
          resources:
            requests: { cpu: 10m, memory: 16Mi }
            limits: { memory: 64Mi }
---
apiVersion: v1
kind: Service
metadata:
  name: whoami
spec:
  selector:
    app: whoami
  ports:
    - port: 80
      targetPort: 80
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: whoami
  annotations:
    traefik.ingress.kubernetes.io/router.entrypoints: websecure
    traefik.ingress.kubernetes.io/router.tls: "true"
spec:
  ingressClassName: traefik
  tls:                        # no secretName: Traefik keeps its default cert; Argo CD now links to https://
    - hosts:
        - whoami.home.lab
  rules:
    - host: whoami.home.lab
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: whoami
                port:
                  number: 80
'@ | Set-Content -Encoding ascii "apps\whoami\whoami.yaml"

# --- Argo CD Application for whoami ---------------------------------------
@"
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: whoami
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/$GhUser/homelab-gitops.git
    targetRevision: main
    path: apps/whoami
  destination:
    server: https://kubernetes.default.svc
    namespace: whoami
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
"@ | Set-Content -Encoding ascii "argocd\apps\whoami.yaml"

# --- root app: watches argocd/apps and creates every Application in it ----
@"
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: root
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/$GhUser/homelab-gitops.git
    targetRevision: main
    path: argocd/apps
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
"@ | Set-Content -Encoding ascii "argocd\root.yaml"

# --- push --------------------------------------------------------------------
# (first time only) git config --global user.name "Your Name"; git config --global user.email "you@example.com"
git init -b main
git add .
git commit -m "GitOps: whoami app + app-of-apps root"
git remote add origin "https://github.com/$GhUser/homelab-gitops.git"
git push -u origin main
```

Bootstrap: this is the **only** `kubectl apply` you ever do for apps.

```powershell
kubectl apply -f argocd\root.yaml
argocd app list                                   # root + whoami -> Synced / Healthy
kubectl -n whoami get pods -o wide                # 3 pods spread over the agents

# load-balanced across pods/agents: Hostname changes between calls
1..6 | ForEach-Object { (curl.exe -sk https://whoami.home.lab/whoami) -match '^Hostname' }
```

**See GitOps work:**

```powershell
# a) change in Git -> cluster follows
(Get-Content apps\whoami\whoami.yaml) -replace 'replicas: 3', 'replicas: 5' | Set-Content -Encoding ascii apps\whoami\whoami.yaml
git commit -am "scale whoami to 5"
git push
argocd app get whoami --refresh                   # or click REFRESH in the UI
kubectl -n whoami get pods

# b) manual drift -> Argo CD reverts it (selfHeal)
kubectl -n whoami scale deployment whoami --replicas=1
kubectl -n whoami get deployment whoami -w        # jumps back to 5 within seconds, Ctrl+C
```

**Adding the next app:** put its manifests in `apps/<name>/`, add `argocd/apps/<name>.yaml` (copy `whoami.yaml` and change `name`, `path` and `namespace`), then push.

**Private repo?** Register it once:

```powershell
argocd repo add "https://github.com/$GhUser/homelab-gitops.git" --username $GhUser --password <github-personal-access-token>
```

---

## Step 19 — Day-2: start/stop, backups, upgrades

### Start and stop the lab in the right order

**Run on:** Windows host.

```powershell
# START: datastore -> LB -> servers -> agents
foreach ($vm in "k3s-db", "k3s-lb", "k3s-srv-1", "k3s-srv-2", "k3s-agent-1", "k3s-agent-2", "k3s-agent-3") {
    VBoxManage startvm $vm --type headless
    Start-Sleep -Seconds 15
}
VBoxManage startvm kali --type gui           # Kali desktop workstation (Appendix E)

# STOP (graceful ACPI shutdown, reverse order)
ssh kali "sudo poweroff"                     # Kali desktop: clean shutdown over SSH (the desktop may ignore the ACPI button)
foreach ($vm in "k3s-agent-3", "k3s-agent-2", "k3s-agent-1", "k3s-srv-2", "k3s-srv-1", "k3s-lb", "k3s-db") {
    VBoxManage controlvm $vm acpipowerbutton
}
```

### Backups

**Run on:** `k3s-db`. Nightly dump of the cluster state, keeping 7 days. Connect from Windows, then paste the block below:

```powershell
ssh -p 22011 kali@127.0.0.1      # k3s-db   (or: ssh k3s-db)
```

```bash
sudo mkdir -p /var/backups/k3s
echo '0 3 * * * root mariadb-dump --single-transaction k3s | gzip > /var/backups/k3s/k3s-$(date +\%F).sql.gz && find /var/backups/k3s -name "*.sql.gz" -mtime +7 -delete' | sudo tee /etc/cron.d/k3s-db-backup
```

Also keep:
- `~/k3s-secrets.env`, because the **token is required** to restore;
- VirtualBox snapshots (`VBoxManage snapshot <vm> take <name> --live`);
- your **Git repo**. Argo CD's apps are already backed up there, and `argocd admin export -n argocd > argocd-backup.yaml` exports Argo CD's own settings.

### Upgrade K3s (one node at a time)

`config.yaml` holds all the flags, so an upgrade is just re-running the installer with a new version.

```bash
# on k3s-srv-1 (ssh -p 22021 kali@127.0.0.1), wait until Ready, then on k3s-srv-2 (ssh -p 22022 kali@127.0.0.1):
curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION="v1.xx.y+k3s1" sh -s - server
```

```powershell
# Windows host, for each agent:
kubectl drain k3s-agent-1 --ignore-daemonsets --delete-emptydir-data
```

```bash
# on that agent: ssh -p 22031 kali@127.0.0.1 (k3s-agent-1), port 22032 (k3s-agent-2), port 22033 (k3s-agent-3)
curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION="v1.xx.y+k3s1" sh -s - agent
```

```powershell
kubectl uncordon k3s-agent-1     # repeat drain/upgrade/uncordon for agent-2 and agent-3
```

### Upgrade Argo CD

```powershell
kubectl apply -n argocd --server-side --force-conflicts -f https://raw.githubusercontent.com/argoproj/argo-cd/<vX.Y.Z>/manifests/install.yaml
```

The `server.insecure` patch from Step 15 survives this, because it is owned by a different field manager.

---

## Appendix A — Variant: embedded etcd (no external DB)

This removes `k3s-db` as a single point of failure. It needs an **odd number of servers, at least 3** (3 servers tolerate 1 failure, 5 tolerate 2).

Changes to the plan:
- Skip `k3s-db`.
- Add `k3s-srv-3` at `192.168.99.23` with `NatPort = 22023` to the Step 5 list, then run `sudo set-identity k3s-srv-3 192.168.99.23` in its window (or Step 6 option B). The name is already in the `home.lab` block and the SSH config.
- Add `server 192.168.99.23:6443 ...;` to the `k3s_api` upstream on `k3s-lb`.

**Run on:** `k3s-srv-1` (`K3S_TOKEN` from `openssl rand -hex 32`). Connect from Windows, then paste the block below:

```powershell
ssh -p 22021 kali@127.0.0.1      # k3s-srv-1   (or: ssh k3s-srv-1)
```

```bash
NODE_IP=$(ip -4 -o addr show eth1 | awk '{print $4}' | cut -d/ -f1)
sudo mkdir -p /etc/rancher/k3s
sudo tee /etc/rancher/k3s/config.yaml >/dev/null <<EOF
token: "${K3S_TOKEN}"
cluster-init: true
node-ip: "${NODE_IP}"
flannel-iface: "eth1"
node-taint:
  - "CriticalAddonsOnly=true:NoExecute"
tls-san:
  - "192.168.99.10"
  - "k3s.home.lab"
EOF
curl -sfL https://get.k3s.io | sh -s - server
```

**Run on:** `k3s-srv-2` (`ssh -p 22022 kali@127.0.0.1`) and `k3s-srv-3` (`ssh -p 22023 kali@127.0.0.1`). Same file, but replace `cluster-init: true` with `server: "https://192.168.99.21:6443"`, then run the same `curl ... | sh -s - server`.

Agents are unchanged (Step 11). Backups: `sudo k3s etcd-snapshot save` (stored in `/var/lib/rancher/k3s/server/db/snapshots`).

---

## Appendix B — Make the load balancer HA (keepalived)

Run two NGINX VMs that share a **virtual IP**, `192.168.99.10`. Every other step keeps using `.10` and stays unchanged.

| VM | IP | keepalived role |
|---|---|---|
| `k3s-lb-1` | `192.168.99.8` | MASTER (priority 150) |
| `k3s-lb-2` | `192.168.99.9` | BACKUP (priority 100) |
| VIP | `192.168.99.10` | floats to whichever is healthy |

**Setup:**
1. In the Step 5 list, replace `k3s-lb` with `k3s-lb-1` (`192.168.99.8`, `NatPort = 22008`) and `k3s-lb-2` (`192.168.99.9`, `NatPort = 22009`), both with 1 vCPU / 1 GB, and clone them.
2. In their VM windows (Step 6, or Step 6 option B): `sudo set-identity k3s-lb-1 192.168.99.8` and `sudo set-identity k3s-lb-2 192.168.99.9`. Both names are already in the `home.lab` block and the SSH config, and the Step 6 check covers them.
3. Run **Step 8** on both: connect with `ssh -p 22008 kali@127.0.0.1` for `k3s-lb-1` and `ssh -p 22009 kali@127.0.0.1` for `k3s-lb-2` (or `ssh k3s-lb-1`, `ssh k3s-lb-2`).
4. **Run on:** both LB VMs, connected as in item 3:

```bash
sudo apt install -y keepalived
ME=$(ip -4 -o addr show eth1 | awk '{print $4}' | cut -d/ -f1)
if [ "$ME" = "192.168.99.8" ]; then STATE=MASTER; PRIO=150; PEER=192.168.99.9; else STATE=BACKUP; PRIO=100; PEER=192.168.99.8; fi

sudo tee /etc/keepalived/keepalived.conf >/dev/null <<EOF
global_defs {
    enable_script_security
    script_user root
}
vrrp_script chk_nginx {
    script "/usr/bin/systemctl is-active --quiet nginx"
    interval 2
    fall 2
    rise 2
}
vrrp_instance K3S_VIP {
    state ${STATE}
    interface eth1
    virtual_router_id 51
    priority ${PRIO}
    advert_int 1
    unicast_src_ip ${ME}
    unicast_peer {
        ${PEER}
    }
    authentication {
        auth_type PASS
        auth_pass k3sVIP42
    }
    virtual_ipaddress {
        192.168.99.10/24
    }
    track_script {
        chk_nginx
    }
}
EOF
sudo systemctl enable --now keepalived
ip -4 -br addr show eth1        # the MASTER also shows 192.168.99.10
```

To test, power off `k3s-lb-1`. The VIP moves to `k3s-lb-2`, and `kubectl get nodes` keeps working.

---

## Appendix C — Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `kubectl get nodes -o wide` shows INTERNAL-IP **`10.0.2.15`**, or pods on different nodes can't talk | `node-ip` / `flannel-iface` missing, so K3s used the NAT NIC | Fix `/etc/rancher/k3s/config.yaml`, then `sudo systemctl restart k3s` (or `k3s-agent`) |
| `x509: certificate is valid for ..., not 192.168.99.10` | LB IP not in `tls-san` when the server first started | Add it to `tls-san`, then on each server: `sudo systemctl stop k3s && sudo k3s certificate rotate && sudo systemctl start k3s` |
| `k3s-srv-2` fails: *bootstrap data already found and encrypted with different token* | the token differs from srv-1's | Use the identical `token` (from `k3s-secrets.env`) |
| `Access denied for user 'k3s'@'192.168.99.2x'` | DB user, host or password mismatch | Re-run the Step 7 SQL; check with `sudo mariadb -e "SELECT user,host FROM mysql.user;"` |
| `dial tcp 192.168.99.11:3306: connect: connection refused` | MariaDB still bound to 127.0.0.1 | Set `bind-address = 0.0.0.0` in `50-server.cnf`, then `sudo systemctl restart mariadb` |
| `sudo mariadb …` on `k3s-db` fails with `ERROR 2026 (HY000): TLS/SSL error: tlsv1 alert internal error`, sometimes or every time | MariaDB 11.4+ tools use TLS by default, even over the local socket. On Kali the server's built-in TLS fails on reused worker threads (`FLUSH SSL` reports *SSL_CTX_new failed*). It also breaks `mariadb-dump` (the nightly backup) at random. K3s is not affected, because it connects without TLS | Write the `90-k3s.cnf` file from Step 7. Its `[client] ssl = 0` applies to the next command, with no restart. For one command only: `sudo mariadb --skip-ssl …` |
| K3s logs `Error 1040: Too many connections`; MariaDB logs *Aborted connection … (Too many connections)* | Starting K3s servers open more than MariaDB's default 151 connections | `90-k3s.cnf` from Step 7 (`max_connections = 500`). To apply it without restarting under the running cluster: `sudo mariadb -e "SET GLOBAL max_connections = 500"` |
| Agent hangs or logs `connection refused` to `192.168.99.10:6443` | NGINX not forwarding | On `k3s-lb`: `sudo nginx -t`, `sudo ss -tlnp`, `nc -zv 192.168.99.21 6443`, `tail /var/log/nginx/k3s-stream.log` |
| `nginx: [emerg] unknown directive "stream"` | stream module missing | `sudo apt install libnginx-mod-stream`; keep the `include /etc/nginx/modules-enabled/*.conf;` line |
| Pods **Pending** with *untolerated taint {CriticalAddonsOnly}* (`helm-install-traefik*`, `helm-install-gateway-api-crd`, later `traefik`, `svclb-*`) | no agents yet; servers are tainted by design. Only `coredns`, `metrics-server` and `local-path-provisioner` tolerate the taint and run on the servers | Finish Step 11. When `get nodes` shows only the 2 servers after Step 11, check `sudo journalctl -u k3s-agent -n 5` on an agent (next row) |
| Agents stay `activating`, and `journalctl -u k3s-agent` repeats *failed to retrieve configuration from server: **not authorized*** | The agent's token isn't the cluster's token. Usually the Step 7 secrets block ran again after Step 9 and made new secrets: the cluster keeps the **first** token, and MariaDB keeps the **first** `k3s` password (`CREATE USER IF NOT EXISTS` doesn't change it). The running servers still work from memory, but they **won't start again** with the new files | Compare on a server: `sudo cat /var/lib/rancher/k3s/server/token \| sed 's/.*::server://' \| tr -d '\n' \| sha256sum` with `grep -oP '^K3S_TOKEN=\K.*' ~/k3s-secrets.env \| tr -d '\n' \| sha256sum`. If they differ, keep the cluster's **original token** with the **new DB password**. The original token is still in `/var/lib/rancher/k3s/server/token`, while the old DB password is gone. 1) `k3s-db`: `source ~/k3s-secrets.env`, then `sudo mariadb` with `ALTER USER 'k3s'@'192.168.99.21' IDENTIFIED BY '<DB_PASS>';` and the same for `.22`. 2) Each server: `OLD=$(sudo sed 's/.*::server://' /var/lib/rancher/k3s/server/token)`, put `$OLD` into `token:` in `/etc/rancher/k3s/config.yaml` and into `K3S_TOKEN=` in `~/k3s-secrets.env`, then `sudo systemctl restart k3s`. 3) Put the same token into each agent's `config.yaml` and `~/k3s-agent.env`, and into `k3s-db`'s two files, then `sudo systemctl restart k3s-agent` on the agents |
| `kubectl` on a server: *The connection to the server 127.0.0.1:6443 was refused*; `systemctl status k3s` shows *activating (auto-restart)*; `journalctl -u k3s` shows *Error 1045 … Access denied for user 'k3s'* or *bootstrap data already found and encrypted with different token* | The server's `config.yaml` no longer matches the cluster. Usually Step 7 made new secrets after Step 9, and a later run of the Step 9/10 block wrote them into `config.yaml` and restarted K3s. It now restarts every few seconds, so the load climbs past 20 and SSH over `192.168.99.x` can stall | Get in through the NAT port (`ssh -p 22021 kali@127.0.0.1`), run `sudo systemctl stop k3s` to calm the VM down, then follow the fix in the row above |
| Argo CD **HA** pods Pending | the HA manifest needs 3 schedulable nodes | Use 3 agents, or the non-HA `install.yaml` |
| Browser shows `ERR_TOO_MANY_REDIRECTS` on argocd.home.lab | `server.insecure` not applied | Repeat Step 15 part 1 (patch + rollout restart) |
| `argocd login` hangs, or shows *transport is closing* | gRPC through the ingress | Add `--grpc-web` |
| `404 page not found` (from Traefik) | hosts entry or Ingress host mismatch, or you used `http://` | Check the hosts file and `kubectl get ingress -A`; use `https://` |
| `Clone VM failed` / `Failed to lock source media '…kali-template.vdi'` (`VBOX_E_INVALID_OBJECT_STATE`) in Step 5 | `kali-template` is still running, and a running VM keeps its disk locked | After the Step 4 part 4 check, power the template off: `ssh kali-template "sudo poweroff"` on Windows, or `sudo poweroff` in its VM window. Wait until `VBoxManage list runningvms` no longer lists it, then run Step 5 again |
| Step 5 prints *WARNING: kali-template is not powered off (VMState="running")* | Same cause. The block stopped before cloning, so **no clone exists yet** | Same fix, using the power-off lines under the Step 5 block. Don't start Step 6 before the block prints its 7 `ssh -p` lines |
| *Sudo is disabled on this machine. To enable it, go to the Developer Settings page* | A Linux command such as `sudo poweroff` was typed in **Windows** PowerShell, where `sudo` is a separate Windows feature that is switched off | Run it inside the VM, in its window or over SSH: `ssh kali-template "sudo poweroff"`. Don't turn on Windows sudo: Windows has no `poweroff` command |
| `ssh kali-template` or `ssh -p 2222 kali@127.0.0.1`: *Connection reset*, *kex_exchange_identification: Connection closed by remote host*, or a timeout | the VM is not running or still booting, its SSH server is off (Step 4 part 1), the port forward is missing, or `eth0` has no address | `VBoxManage showvminfo kali-template --machinereadable \| Select-String '^Forwarding'` must print `ssh,tcp,127.0.0.1,2222,,22` (add it as in the Step 3 note); in the VM window, `systemctl is-active ssh` must say `active` and `ip -br a` must show `10.0.2.15` on `eth0` (otherwise see the `eth0` note in Step 4 part 2) |
| *WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED … Host key verification failed* for `kali-template` or `[127.0.0.1]:2222` | The template was reinstalled, so it has new SSH host keys, but `~\.ssh\known_hosts` still holds the old ones | Run Step 4 part 2 again, which removes them first. By hand: `ssh-keygen -R kali-template; ssh-keygen -R "[127.0.0.1]:2222"`, then connect again. In Step 6 option B, the same warning only means that clone already ran `set-identity`. **Template not reinstalled?** Then `set-identity` probably ran on it: see *The template got a clone's identity* below |
| The `kali-template` window or `hostname` shows a node name (e.g. `k3s-lb`), and Step 5 has created no clones | A Step 6 `set-identity` line ran in the template, usually because Step 5 had stopped with its warning and the template was the only VM running | See *The template got a clone's identity* below |
| A clone's port forward doesn't answer, or `VBoxManage startvm` reports *Locked* / port forward errors in Step 5 | the clone was running while Step 5 changed its rule, or another program uses that host port | Power the clone off, then `VBoxManage modifyvm <vm> --natpf1 delete ssh; VBoxManage modifyvm <vm> --natpf1 "ssh,tcp,127.0.0.1,<NatPort>,,22"`; `Get-NetTCPConnection -LocalPort <NatPort>` shows who holds a port |
| Step 1.7 shows `WARN` for `kubectl` or `git`, or `kubectl version --client` prints an older version than Step 1.3 downloaded | Docker Desktop or an installed Git is on the **system** PATH, which Windows searches before the user PATH | Run block 1.1 again (it writes the PowerShell profile line), then open a new window. `Test-Path $PROFILE.CurrentUserAllHosts` must print `True`, and `Get-ExecutionPolicy` must not print `Restricted` |
| *The host network interface named 'VirtualBox Host-Only Ethernet Adapter' could not be found*, or a VM won't start with *Nonexistent host networking interface* | The host-only adapter was removed, for example with **Remove** in *File → Tools → Network Manager* | Power off every VM, then run Step 2 again in an Administrator PowerShell |
| Step 2 stops with *hostonlyif create made '… Adapter #2'* | Windows still holds an old host-only adapter that the block could not remove | *Device Manager → View → Show hidden devices → Network adapters*: uninstall every *VirtualBox Host-Only* entry, then run Step 2 again |
| An `ssh` command hangs with no output and no password prompt | it runs in **PowerShell ISE**, which can't show console prompts or run interactive SSH | Click **Stop** in ISE (if `ssh.exe` stays alive: `Stop-Process -Name ssh`), then run the same block in a normal PowerShell window or Windows Terminal |
| `sudo: set-identity: command not found` on a clone | Kali's `sudo` doesn't search `/usr/local/sbin` (Step 4 part 3 item 1 adds it), the template was cloned before part 3 finished, or a `sudo` password prompt swallowed part of the paste | On that clone, `sudo /usr/local/sbin/set-identity <name> <ip>` works if the file exists. Otherwise follow *Already cloned…* at the end of Step 4: delete the clones, fix the template, run the part 4 check, clone again |
| `set-identity <name> <ip>` only prints its usage line, or `[[: not found` | `#!/usr/bin/env bash` is not the script's first line (an empty line from the paste), so `sh` runs it | On the template, or on each clone before its `set-identity`: `sudo sed -i -e 's/\r$//' -e '/./,$!d' /usr/local/sbin/set-identity`, then run the line again |
| `set-identity` says *unknown VM name* or *wrong IP for …* | a typo in the name or the IP, or the `home.lab` block is missing | Type the line exactly as in the Step 6 table. `grep -A14 '^# BEGIN home.lab' /etc/hosts` must show the block (Step 4 item 6) |
| Two VMs answer on the same IP / *REMOTE HOST IDENTIFICATION HAS CHANGED* | `set-identity` skipped, or an IP was reused | Run `set-identity` on the clone; `ssh-keygen -R 192.168.99.x` |
| VMs very slow, turtle icon | Hyper-V active underneath VirtualBox | See the Hyper-V note in Step 1 |
| Errors after Windows sleep/hibernate (TLS *not yet valid*, DB timeouts) | VM clock drift | `sudo systemctl restart systemd-timesyncd; timedatectl` on each VM |
| NICs have `enp0s…` names instead of `eth0` / `eth1` | `net.ifnames=0` is missing from the kernel command line | `cat /proc/cmdline`; redo the GRUB lines of Step 4 part 2, then reboot |
| A VM has no `192.168.99.x` after `set-identity` | NetworkManager ignored the keyfile, or NIC 2 is not attached | Get in with copy and paste through its NAT port forward: `ssh -p <NatPort> kali@127.0.0.1` (Step 5 table, e.g. `22021` for `k3s-srv-1`). Then `nmcli connection show`; the files in `/etc/NetworkManager/system-connections/` must be owned by `root` with mode `600`; `sudo nmcli connection up hostonly`; on Windows, `VBoxManage showvminfo <vm> \| Select-String "NIC 2"` must say *Host-only* |
| Pods fail to start with an `apparmor` error | AppArmor userspace tools missing | `sudo apt install -y apparmor`, then `sudo systemctl restart k3s` (or `k3s-agent`) |
| `kubectl` on `kali` → *connection refused* to `127.0.0.1:6443` | the raw `k3s.yaml` was copied instead of the Step 12 kubeconfig | Re-run the `scp "$HOME\.kube\config" kali:.kube/config` line from E.3 |
| `kali` boots to a **text login**, no desktop | **Xfce** was not ticked in Software selection; ticking only the *Desktop environment* heading does nothing | `sudo apt update && sudo apt install -y kali-desktop-xfce` (what the Xfce entry installs), or `sudo tasksel` and tick Xfce; then `sudo systemctl set-default graphical.target && sudo reboot` |
| `kali` window doesn't resize, shared clipboard doesn't work | VirtualBox guest utilities missing | `sudo apt update && sudo apt install -y --reinstall virtualbox-guest-x11 && sudo reboot`; check `VBoxManage showvminfo kali \| Select-String Clipboard` says *Bidirectional* |

### The template got a clone's identity

**Symptoms:** `ssh kali-template` stops with *REMOTE HOST IDENTIFICATION HAS CHANGED*, although you didn't reinstall the template. The template's `hostname` is a node name such as `k3s-lb`, with that node's IP on `eth1`.

**Cause:** a Step 6 `sudo set-identity …` line ran in `kali-template`. Templates built with the current Step 4 item 7 refuse this with *this is kali-template itself*. Older ones go ahead, give themselves the node's name, IP, machine-id and SSH host keys, and reboot.

**Fix:** you don't need to reinstall. Change the template back, then continue from the Step 4 check.

1. **Run on:** Windows host. Forget the template's old host key and log in again. `accept-new` from the Step 4 shortcut saves the new key under `kali-template`, which Step 6 option B checks against:

   ```powershell
   ssh-keygen -R kali-template
   ssh-keygen -R "[127.0.0.1]:2222"
   ssh kali-template      # the shortcut here, not ssh -p 2222 kali@127.0.0.1: only the shortcut saves the key as kali-template
   ```

2. **Run on:** `kali-template`, in that SSH session. Restore the template's name, and put `eth1` back on DHCP so the clones don't all boot with the node's IP:

   ```bash
   sudo hostnamectl set-hostname kali-template
   sudo sed -i '/^127\.0\.1\.1/d' /etc/hosts
   echo '127.0.1.1 kali-template' | sudo tee -a /etc/hosts
   sudo sed -i 's/^method=manual$/method=auto/; /^address1=/d' /etc/NetworkManager/system-connections/hostonly.nmconnection
   hostname     # must print: kali-template
   ```

   The new machine-id and SSH host keys can stay, because `set-identity` gives every clone its own anyway.

3. Paste Step 4 part 3 item 7 again, from its `# 7)` line down to and including the `sed` line. That installs the guard and saves the template's MAC. Then run the part 4 check (every line `OK`, including *refuses the template*), part 5 (power off), and Steps 5 and 6.

> **Already cloned from the changed template?** Every clone then says `k3s-lb` (or whichever name ran), and only one of them holds that IP. The others see a duplicate address and leave `eth1` empty. You can keep these clones. Step 6 replaces everything `set-identity` changed, so run it on them as usual. Option B works if `~\.ssh\known_hosts` holds the changed template's key under `kali-template`, which step 1 above saves. If option B says *REMOTE HOST IDENTIFICATION HAS CHANGED* for every clone, use option A. Afterwards, do steps 1–3 on the template before you boot or clone it again (Appendix A/B). Otherwise it comes up with the node's IP and clashes with the real node.

Useful logs:

```bash
sudo journalctl -u k3s -f              # servers: ssh -p 22021 kali@127.0.0.1 (k3s-srv-1), port 22022 (k3s-srv-2)
sudo journalctl -u k3s-agent -f        # agents:  ssh -p 22031 kali@127.0.0.1 (k3s-agent-1), port 22032 / 22033 (-2 / -3)
sudo tail -f /var/log/nginx/k3s-stream.log /var/log/nginx/error.log   # k3s-lb: ssh -p 22010 kali@127.0.0.1
sudo journalctl -u mariadb -f          # k3s-db:  ssh -p 22011 kali@127.0.0.1
```

```powershell
kubectl get events -A --sort-by=.lastTimestamp | Select-Object -Last 30
kubectl -n argocd logs deploy/argocd-server --tail=50
```

---

## Appendix D — Reset and destroy

```bash
# agents:  ssh -p 22031 kali@127.0.0.1 (k3s-agent-1), port 22032 / 22033 (k3s-agent-2 / -3)
/usr/local/bin/k3s-agent-uninstall.sh
# servers: ssh -p 22021 kali@127.0.0.1 (k3s-srv-1), port 22022 (k3s-srv-2)
/usr/local/bin/k3s-uninstall.sh
# k3s-db:  ssh -p 22011 kali@127.0.0.1. Wipe the cluster state (only after ALL servers are uninstalled)
sudo mariadb -e "DROP DATABASE k3s; CREATE DATABASE k3s;"
```

Delete all VMs: **Run on** Windows host. ⚠️ This deletes their disks.

```powershell
foreach ($vm in "k3s-lb", "k3s-db", "k3s-srv-1", "k3s-srv-2", "k3s-agent-1", "k3s-agent-2", "k3s-agent-3") {
    VBoxManage controlvm $vm poweroff 2>$null
    VBoxManage unregistervm $vm --delete
}
# keep 'kali-template' to rebuild quickly, or remove it too:
# VBoxManage unregistervm kali-template --delete
# the Kali workstation (Appendix E) is kept as well; to remove it:
# VBoxManage controlvm kali poweroff; VBoxManage unregistervm kali --delete
```

---

## Appendix E — Kali Linux workstation VM

`kali` is a desktop VM for running the cluster from a second machine. It has the Xfce desktop, Firefox, `kubectl` and `argocd`. It reaches the cluster through `k3s-lb`, the same way the Windows host does, and is not a cluster node.

It is **not** cloned from `kali-template`, because a clone inherits the template's headless install. E.1 installs it from the **same Kali ISO** as Step 3. The GUI comes from the installer's **Software selection** screen, where you tick **Xfce**. You can do E.1 and E.2 any time after Step 6. E.3 needs the cluster.

| Setting | Value |
|---|---|
| VM name / hostname / FQDN | `kali` / `kali` / `kali.home.lab` (VirtualBox group **k3s-ha**) |
| NIC 1 (`eth0`) | VirtualBox NAT → Internet, DHCP, default route |
| NIC 2 (`eth1`) | host-only, static **`192.168.99.50/24`**, no gateway, DNS search `home.lab` (set in E.2) |
| Desktop | **Xfce** (`kali-desktop-xfce`, ticked in Software selection) with the LightDM graphical login; VirtualBox guest utilities (the installer adds them); Firefox ESR |
| User | `kali` (the password you set in its own installer, E.1) |
| vCPU / RAM / video / disk | 2 / 4 GB / 128 MB / 25 GB |

> **Already have a `kali` clone** from an earlier version of this guide? Delete it before E.1: `VBoxManage controlvm kali poweroff; VBoxManage unregistervm kali --delete`. Or keep it and run `sudo tasksel` on it. Kali's `tasksel` is the program behind the installer's Software selection screen. Tick **Xfce**, then `sudo reboot`.

### E.1 Install `kali` with the Xfce desktop

**Run on:** Windows host. This is Step 3 again with three changes: more RAM, more video memory for the desktop, and the shared clipboard turned on:

```powershell
$VM  = "kali"
$HOA = "VirtualBox Host-Only Ethernet Adapter"
$ISO = (Get-ChildItem "C:\Working\tools\iso\kali-linux-*-installer*-amd64.iso" | Sort-Object Name -Descending | Select-Object -First 1).FullName
$VDI = "$HOME\VirtualBox VMs\$VM\$VM.vdi"
"ISO: $ISO"     # the same ISO as Step 3 (C:\Working\tools\iso)

VBoxManage createvm --name $VM --ostype Debian_64 --register
VBoxManage modifyvm $VM --cpus 2 --memory 4096 --vram 128 --graphicscontroller vmsvga --audio-enabled off
VBoxManage modifyvm $VM --clipboard-mode bidirectional --drag-and-drop bidirectional
VBoxManage modifyvm $VM --nic1 nat --nic2 hostonly --host-only-adapter2 $HOA
VBoxManage modifyvm $VM --boot1 disk --boot2 dvd --boot3 none --boot4 none
VBoxManage createmedium disk --filename $VDI --size 25600 --format VDI     # 51200 if you will tick a tool collection
VBoxManage storagectl $VM --name SATA --add sata --controller IntelAhci --portcount 2
VBoxManage storageattach $VM --storagectl SATA --port 0 --device 0 --type hdd --medium $VDI
VBoxManage storageattach $VM --storagectl SATA --port 1 --device 0 --type dvddrive --medium $ISO
VBoxManage modifyvm $VM --groups "/k3s-ha"
VBoxManage startvm $VM --type gui
```

**In the VM window**, go through the installer as in Step 3. Three screens differ: **Hostname**, the **password** and **Software selection**.

| Screen | Choose |
|---|---|
| Boot menu | **Graphical install** |
| Language / location / keyboard | your choice |
| Primary network interface | the **first** interface in the list, shown as `eth0` (or `enp0s3`), which is Adapter 1 = NAT. Both entries say *Intel 82540EM*, so go by the order: the second one is the host-only NIC and has no internet |
| Hostname | **`kali`** |
| Domain name | **`home.lab`** |
| Full name / username | `kali` / **`kali`** · a **strong** password, because this VM will hold cluster-admin credentials |
| Time zone | your choice |
| Partition disks | **Guided - use entire disk** → the disk → **All files in one partition** → Finish → Yes |
| HTTP proxy / mirror | leave empty / defaults |
| **Software selection** | tick **Xfce (Kali's default desktop environment)**, as shown below |
| GRUB boot loader | **Yes** → `/dev/sda` |
| Finish | **Continue** (the VM reboots from disk) |

The **Software selection** screen lists these entries, with the items under each heading indented. Set them like this:

```text
[ ] Desktop environment [selecting this item has no effect]
[x]     Xfce (Kali's default desktop environment)                    <- TICK: this is the GUI
[ ]     GNOME
[ ]     KDE Plasma
[ ] Collection of tools [selecting this item has no effect]
[ ]     top10 -- the 10 most popular tools                           <- optional
[ ]     default -- recommended tools (available in the live system)  <- optional
[ ]     large -- default selection plus additional tools             <- optional
```

- **Xfce** is the line that adds the GUI. It installs `kali-desktop-xfce`: the Xfce desktop plus the LightDM graphical login. One desktop is enough. GNOME or KDE Plasma also work, but they need more RAM.
- Ticking a **heading** line on its own does nothing, which is what *"selecting this item has no effect"* means. If you tick *Desktop environment* but not *Xfce*, `kali` boots to a text login with no GUI.
- The tool collections are optional and the cluster doesn't need them. Kali's size table (Kali 2024.1, Xfce) gives about **3.7 GB** with none of them, **6.7 GB** with *top10* and **13 GB** with *default*. If you keep one, use `51200` in the `createmedium` line above, and decide before you run the block.
- The VirtualBox guest utilities are not on this screen. The installer detects VirtualBox and adds `virtualbox-guest-x11` itself, which gives you window resizing and the shared clipboard.

After **Continue**, `kali` reboots into the graphical **LightDM login screen**. Log in as `kali` to get the Xfce desktop.

### E.2 kali: network, SSH and the `home.lab` names

The installer only set up internet access. This part gives `kali` the same network layout as the nodes: `eth0` = NAT and `eth1` = static `192.168.99.50`. It also adds the `home.lab` names and an SSH server.

**Run on:** `kali`, in a terminal on its desktop (**Applications → Terminal Emulator**). Kali's zsh accepts this block as it is.

1. Type `sudo -v` and enter the `kali` password.
2. Copy the block on Windows.
3. Paste it into the terminal with **Ctrl+Shift+V**, then press **Enter**.

```bash
# 1) passwordless sudo: the same lab convenience as the nodes (Step 19 shuts kali down over SSH)
echo 'kali ALL=(ALL) NOPASSWD:ALL' | sudo tee /etc/sudoers.d/90-kali-nopasswd
sudo chmod 440 /etc/sudoers.d/90-kali-nopasswd

# 2) SSH server (the Windows host uses it below and in E.3) + Firefox
sudo apt update
sudo apt install -y openssh-server firefox-esr
sudo systemctl enable ssh

# 3) the same NIC names as the nodes: eth0 = Adapter 1 (NAT), eth1 = Adapter 2 (host-only)
printf 'auto lo\niface lo inet loopback\n' | sudo tee /etc/network/interfaces
sudo rm -f /var/lib/NetworkManager/no-auto-default.state
sudo mkdir -p /etc/default/grub.d
echo 'GRUB_CMDLINE_LINUX="$GRUB_CMDLINE_LINUX net.ifnames=0"' | sudo tee /etc/default/grub.d/90-eth-names.cfg
sudo update-grub

# 4) NetworkManager keyfiles: the same two that set-identity writes on the nodes
sudo sh -c 'rm -f /etc/NetworkManager/system-connections/*'
sudo tee /etc/NetworkManager/system-connections/nat.nmconnection >/dev/null <<'EOF'
[connection]
id=nat
type=ethernet
interface-name=eth0

[ipv4]
method=auto

[ipv6]
method=ignore
EOF
sudo tee /etc/NetworkManager/system-connections/hostonly.nmconnection >/dev/null <<'EOF'
[connection]
id=hostonly
type=ethernet
interface-name=eth1

[ipv4]
method=manual
address1=192.168.99.50/24
never-default=true
dns-search=home.lab;

[ipv6]
method=disabled
EOF
sudo chmod 600 /etc/NetworkManager/system-connections/nat.nmconnection /etc/NetworkManager/system-connections/hostonly.nmconnection

# 5) lab domain home.lab: the same names block as the nodes (Step 4)
sudo sed -i '/^# BEGIN home.lab/,/^# END home.lab/d' /etc/hosts
sudo tee -a /etc/hosts >/dev/null <<'EOF'
# BEGIN home.lab
192.168.99.10  k3s-lb.home.lab k3s-lb k3s.home.lab argocd.home.lab whoami.home.lab
192.168.99.11  k3s-db.home.lab k3s-db
192.168.99.21  k3s-srv-1.home.lab k3s-srv-1
192.168.99.22  k3s-srv-2.home.lab k3s-srv-2
192.168.99.31  k3s-agent-1.home.lab k3s-agent-1
192.168.99.32  k3s-agent-2.home.lab k3s-agent-2
192.168.99.33  k3s-agent-3.home.lab k3s-agent-3
192.168.99.50  kali.home.lab kali
# only used by Appendix A (k3s-srv-3) and Appendix B (k3s-lb-1/2):
192.168.99.23  k3s-srv-3.home.lab k3s-srv-3
192.168.99.8   k3s-lb-1.home.lab k3s-lb-1
192.168.99.9   k3s-lb-2.home.lab k3s-lb-2
# END home.lab
EOF
grep '^127.0.1.1' /etc/hosts      # 127.0.1.1  kali.home.lab  kali   (the installer wrote it from Hostname + Domain name)

sudo reboot
```

After the reboot, log in and run `ip -br a` in a terminal. `eth0` should have `10.0.2.15` and `eth1` should have `192.168.99.50/24`.

**Run on:** Windows host. Step 6 already added `Host kali` to `~/.ssh/config`. The first command asks for the `kali` password, and after that SSH uses your key:

```powershell
ssh-keygen -R 192.168.99.50 2>$null | Out-Null     # a new install has new host keys: forget those of an earlier kali VM
Get-Content "$HOME\.ssh\id_ed25519.pub" | ssh kali "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
ssh kali "hostname -f; ip -4 -br addr show eth1"     # kali.home.lab, then eth1 UP 192.168.99.50/24 - no password
VBoxManage snapshot kali take "01-desktop" --live
```

**Run on:** Windows host, in an **Administrator** PowerShell, so the name also works from Windows:

```powershell
Add-Content -Path "$env:SystemRoot\System32\drivers\etc\hosts" -Value "`r`n192.168.99.50  kali.home.lab"
ipconfig /flushdns
```

### E.3 kali: cluster tools and access

This needs the cluster: the kubectl part works after **Step 12**, the Argo CD part after **Step 16**.

**Run on:** `kali` (`ssh kali`, or a terminal on its desktop):

```bash
# kubectl + Argo CD CLI (official binaries)
curl -fsSLo /tmp/kubectl "https://dl.k8s.io/release/$(curl -fsSL https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
curl -fsSLo /tmp/argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
sudo install -m 0755 /tmp/kubectl /tmp/argocd /usr/local/bin/
kubectl version --client && argocd version --client

# Kali's own SSH key + a shortcut to the nodes (the Windows host authorizes the key next)
mkdir -p ~/.ssh && chmod 700 ~/.ssh
[ -f ~/.ssh/id_ed25519 ] || ssh-keygen -t ed25519 -N '' -f ~/.ssh/id_ed25519
grep -q '^Host k3s-\*' ~/.ssh/config 2>/dev/null || cat >> ~/.ssh/config <<'EOF'
Host k3s-*
    StrictHostKeyChecking accept-new
EOF
chmod 600 ~/.ssh/config
```

**Run on:** Windows host. Copy the kubeconfig, which Step 12 already pointed at `192.168.99.10`, and authorize Kali's key on every node:

```powershell
ssh kali "mkdir -p ~/.kube && chmod 700 ~/.kube"
scp "$HOME\.kube\config" kali:.kube/config
ssh kali "chmod 600 ~/.kube/config"

$kaliKey = (ssh kali "cat ~/.ssh/id_ed25519.pub").Trim()
foreach ($h in "k3s-lb", "k3s-db", "k3s-srv-1", "k3s-srv-2", "k3s-agent-1", "k3s-agent-2", "k3s-agent-3") {
    ssh $h "grep -qxF '$kaliKey' ~/.ssh/authorized_keys || echo '$kaliKey' >> ~/.ssh/authorized_keys"
}
```

**Run on:** `kali`:

```bash
kubectl config current-context                # k3s-ha
kubectl get nodes -o wide                     # the same 5 nodes as on Windows
ssh k3s-srv-1 hostname                        # k3s-srv-1 - key-based, no password
curl -skI https://argocd.home.lab | head -1   # HTTP/1.1 200 OK (after Step 15)
argocd login argocd.home.lab --username admin --grpc-web --insecure    # password from Step 16
argocd app list
```

Firefox on Kali opens **<https://argocd.home.lab>** through the same `/etc/hosts` entry.

> `~/.kube/config` on Kali holds the **cluster-admin** credentials, the same as on the Windows host. Give Kali a strong password and keep it on the host-only network.

---

## What ended up on each VM

| VM | Installed / running | Key files |
|---|---|---|
| **Windows host** | `C:\Working\tools\`: `bin\kubectl.exe`, `bin\argocd.exe`, `git\` (portable Git), `iso\` (Kali installer), `downloads\` (installers). VirtualBox in `C:\Program Files\Oracle\VirtualBox`, with the Step 2 networks: host-only adapter `192.168.99.1/24` (DHCP `.100–.254`) and NAT network `NatNetwork`. OpenSSH client from Windows | user `Path` (`C:\Working\tools\bin`, `C:\Working\tools\git\cmd`, VirtualBox), `~\Documents\WindowsPowerShell\profile.ps1` (puts those two first in PowerShell), `~\.kube\config`, `~\.ssh\config`, `C:\Windows\System32\drivers\etc\hosts`, `~\homelab-gitops\` |
| **kali-template** | Kali Linux (minimal, headless), NetworkManager, `set-identity`, `home.lab` hosts block | `/usr/local/sbin/set-identity`, `/etc/sudoers.d/90-kali-nopasswd` + `91-secure-path`, `/etc/default/grub.d/90-eth-names.cfg`, `/etc/NetworkManager/conf.d/90-k3s.conf`, `~/.ssh/authorized_keys` (powered off, source of the 7 clones). VirtualBox NAT port forward `127.0.0.1:2222 → 22` (`ssh kali-template`) |
| **every clone** (the 7 cluster VMs) | everything from the template, plus its identity from `set-identity`, and its own NAT port forward `127.0.0.1:220xx → 22` (Step 5) | `/etc/hosts`, `/etc/NetworkManager/system-connections/{nat,hostonly}.nmconnection` |
| **k3s-lb** | `nginx` + `libnginx-mod-stream` | `/etc/nginx/nginx.conf` |
| **k3s-db** | `mariadb-server` (database `k3s`, user `k3s`) | `/etc/mysql/mariadb.conf.d/50-server.cnf` + `90-k3s.cnf`, `~/k3s-secrets.env`, `/etc/cron.d/k3s-db-backup` |
| **k3s-srv-1/2** | `k3s` (server, tainted) | `/etc/rancher/k3s/config.yaml`, `/etc/rancher/k3s/k3s.yaml` |
| **k3s-agent-1/2/3** | `k3s-agent`, plus pods: Traefik, svclb, Argo CD, apps | `/etc/rancher/k3s/config.yaml` |
| **kali** | its own Kali install: Xfce desktop + LightDM (Software selection), `virtualbox-guest-x11` (added by the installer), OpenSSH server, Firefox ESR, `kubectl`, `argocd` | `/etc/NetworkManager/system-connections/{nat,hostonly}.nmconnection`, `/etc/default/grub.d/90-eth-names.cfg`, `/etc/hosts`, `~/.kube/config`, `~/.ssh/config`, `~/.ssh/id_ed25519` |

---

## Sources

- Video: Christian Lempa — *Using \*HA\* Kubernetes at home, was never so simple!* — <https://www.youtube.com/watch?v=UdjhFLV1yt8>
- Christian Lempa's k3s cheat-sheet (HA with external DB), 2022–2023 versions — <https://github.com/ChristianLempa/cheat-sheets/blob/0a92b5a9ffb42da00717060089bb0d9ed37b940f/kubernetes/k3s.md>
- K3s docs: High Availability with an External DB — <https://docs.k3s.io/datastore/ha>
- K3s docs: Cluster Load Balancer (NGINX / HAProxy + keepalived) — <https://docs.k3s.io/datastore/cluster-loadbalancer>
- K3s docs: Configuration file — <https://docs.k3s.io/installation/configuration>
- Argo CD: Getting Started — <https://argo-cd.readthedocs.io/en/stable/getting_started/>
- Argo CD: Ingress configuration (Traefik) — <https://argo-cd.readthedocs.io/en/stable/operator-manual/ingress/>
- VirtualBox manual: VBoxManage — <https://www.virtualbox.org/manual/>
- VirtualBox manual: installing on Windows (unattended install, install-folder security requirements) — <https://www.virtualbox.org/manual/topics/installation.html>
- VirtualBox downloads and SHA256SUMS — <https://download.virtualbox.org/virtualbox/>
- Kubernetes docs: Install kubectl on Windows — <https://kubernetes.io/docs/tasks/tools/install-kubectl-windows/>
- Git for Windows releases (PortableGit) — <https://github.com/git-for-windows/git/releases>
- Kali Linux current images and SHA256SUMS — <https://cdimage.kali.org/current/>
- Kali Linux: installer images — <https://www.kali.org/get-kali/#kali-installer-images>
- Kali docs: Installing Kali Linux (hard disk install) — <https://www.kali.org/docs/installation/hard-disk-install/>
- Kali docs: VirtualBox guest additions in Kali — <https://www.kali.org/docs/virtualization/install-virtualbox-guest-additions/>
- Kali docs: installation sizes per desktop and tool collection — <https://www.kali.org/docs/installation/installation-sizes/>
- Kali's `tasksel` (the installer's Software selection entries, `kali-tasks/`) — <https://gitlab.com/kalilinux/packages/tasksel>
- Kubernetes docs: Install kubectl on Linux — <https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/>
- Argo CD: CLI installation — <https://argo-cd.readthedocs.io/en/stable/cli_installation/>
