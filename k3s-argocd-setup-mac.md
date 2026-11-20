# HA K3s + Argo CD on Oracle VirtualBox (macOS) — from an empty Mac to GitOps

> **macOS edition of [`k3s-argocd-setup.md`](k3s-argocd-setup.md)** (the Windows 11 guide). It has the same design, VMs, cluster IPs and order of steps.
> - Every command that runs on the **host** is written for **Terminal on the Mac (zsh)** instead of PowerShell.
> - Every command that runs **inside a Kali VM** is the same as in the Windows guide.
>
> **Based on:** Christian Lempa — *"Using \*HA\* Kubernetes at home, was never so simple!"*
> <https://www.youtube.com/watch?v=UdjhFLV1yt8> and his k3s cheat-sheet from the same period.
>
> **The video's design:** an external **MariaDB** datastore, an **NGINX** TCP load balancer, **two K3s
> server nodes** (installed with `--node-taint` and `--tls-san`) and **agent (worker) nodes** that join through the load balancer.
>
> **What this guide adds:**
> - It runs that design as **Oracle VirtualBox** VMs running **Kali Linux** on a **Mac**.
> - **Every VM has the Kali Xfce desktop (GUI)** and opens in its own window. One extra VM, `kali`, is the admin workstation.
> - **Every VM has the same specification**, the one of the `k3s-lb` VM: **2 vCPU, 2 GB RAM, 16 MB video (VMSVGA), 25 GB disk**.
> - It installs **Argo CD** on top for GitOps.
>
> **Written for:**
> - macOS on **Apple Silicon** (M1–M4). Tested on an M2 Pro with 16 GB RAM, macOS 26.
> - VirtualBox **7.2.x** (macOS/Arm64 build).
> - **Kali Linux arm64** (rolling) with the **Xfce desktop** on every VM. It is installed once into the template, which the 7 cluster VMs are cloned from, and once more for the `kali` workstation.
> - K3s *stable* channel and Argo CD *stable*.
>
> **Intel Macs** work as well. Step 1.1 detects the CPU and picks the x86 / amd64 files instead.
>
> **Apple Silicon means ARM VMs.** VirtualBox on an M-series Mac runs only ARM64 guests. So every VM installs from Kali's **arm64** ISO, and every container image in the cluster needs an `arm64` build.
> - K3s, Traefik, Argo CD, `nginx` and `traefik/whoami` all have one.
> - The Windows guide's Step 17 example (Argo CD's *guestbook*) has none. Its image is amd64-only, so Step 17 here uses **podinfo** instead.
>
> **Kali as a node OS:** Kali is a rolling distro based on Debian testing. K3s runs on it the same way it runs on Debian, but Kali is not one of the OSes K3s is tested on, and a rolling `apt full-upgrade` can change packages without warning. Pin `INSTALL_K3S_VERSION` (Step 9), and take a snapshot before you upgrade a node.

---

## Table of contents

0. [Architecture, IP plan, and order of operations](#0-architecture)
1. [Mac: tools in `~/Desktop/Working/Tools`, PATH, SSH key](#step-1--mac-tools-path-ssh-key)
2. [Mac: host-only network and DHCP pool](#step-2--mac-host-only-network-and-dhcp-pool)
3. [Mac: create the template VM and install Kali](#step-3--mac-create-the-template-vm-and-install-kali)
4. [kali-template: prepare the golden image](#step-4--kali-template-prepare-the-golden-image)
5. [Mac: clone the 7 cluster VMs](#step-5--mac-clone-the-7-cluster-vms)
6. [Console or Mac: give each clone its identity](#step-6--console-or-mac-give-each-clone-its-identity)
7. [k3s-db: MariaDB datastore and secrets](#step-7--k3s-db-mariadb-datastore-and-secrets)
8. [k3s-lb: NGINX TCP load balancer](#step-8--k3s-lb-nginx-tcp-load-balancer)
9. [k3s-srv-1: first K3s server](#step-9--k3s-srv-1-first-k3s-server)
10. [k3s-srv-2: second K3s server](#step-10--k3s-srv-2-second-k3s-server)
11. [k3s-agent-1/2/3: join the workers](#step-11--k3s-agent-123-join-the-workers)
12. [Mac: kubectl through the load balancer](#step-12--mac-kubectl-through-the-load-balancer)
13. [Mac: test HA failover](#step-13--mac-test-ha-failover)
14. [Mac: install Argo CD](#step-14--mac-install-argo-cd)
15. [Mac: expose Argo CD through Traefik](#step-15--mac-expose-argo-cd-through-traefik)
16. [Mac: Argo CD CLI login and admin password](#step-16--mac-argo-cd-cli-login-and-admin-password)
17. [Mac: first Argo CD app (podinfo)](#step-17--mac-first-argo-cd-app-podinfo)
18. [Mac and GitHub: your own GitOps repo (app-of-apps)](#step-18--mac-and-github-your-own-gitops-repo-app-of-apps)
19. [Day-2: start/stop, backups, upgrades](#step-19--day-2-startstop-backups-upgrades)
- [Appendix A: embedded etcd variant (no external DB)](#appendix-a--variant-embedded-etcd-no-external-db)
- [Appendix B: make the load balancer HA (keepalived)](#appendix-b--make-the-load-balancer-ha-keepalived)
- [Appendix C: troubleshooting](#appendix-c--troubleshooting)
- [Appendix D: reset and destroy](#appendix-d--reset-and-destroy)
- [Appendix E: Kali Linux desktop (GUI) workstation VM](#appendix-e--kali-linux-desktop-gui-workstation-vm)
- [What ended up on each VM](#what-ended-up-on-each-vm)
- [Sources](#sources)

---

## 0. Architecture

### 0.1 What you will build

```mermaid
flowchart TB
    subgraph HOST["macOS host (Apple Silicon) — Oracle VirtualBox 7.2 — every VM: Kali Linux arm64 + Xfce desktop, 2 vCPU · 2 GB"]
        YOU["You<br/>Terminal (zsh): kubectl · argocd · ssh<br/>Browser<br/>192.168.99.100"]
        subgraph HON["Host-only network k3s-hostonly 192.168.99.0/24 — NIC 2 (eth1) of every VM"]
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
            KALI["kali · 192.168.99.50<br/>Kali Xfce desktop (GUI) workstation<br/>kali.home.lab"]
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
=============  macOS HOST (Apple Silicon)  -  Oracle VirtualBox 7.2  =============

  [ YOU ]  Terminal / zsh (kubectl, argocd, ssh)  +  Browser
           host-only network k3s-hostonly: 192.168.99.100 (bridge100)
  [ KALI ] Kali Xfce desktop (GUI) VM    kali.home.lab   192.168.99.50
           same paths as YOU: kubectl -> k3s-lb:6443, Firefox -> k3s-lb:443
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
  OS on every VM:    Kali Linux arm64 (rolling) with the Xfce desktop (GUI)
  Every VM:          2 vCPU, 2 GB RAM, 16 MB video, 25 GB disk  (the k3s-lb specification)
```

**Why two NICs per VM?**
- **NIC 1 (NAT, `eth0`)** gives each VM internet access. Every VM gets the *same* NAT address, `10.0.2.15`, so this NIC is useless for traffic between VMs.
- **NIC 2 (host-only, `eth1`)** puts every VM and your Mac on one private network with fixed IPs. All cluster traffic uses it.

K3s therefore gets `node-ip` and `flannel-iface: eth1` explicitly. If you skip that, the nodes advertise `10.0.2.15` and the cluster breaks. This is the classic VirtualBox gotcha.

### 0.2 IP plan and sizing

| VM / thing | Role | Host-only IP (`eth1`) | Desktop | vCPU | RAM | Video | Disk (dynamic) |
|---|---|---|---|---|---|---|---|
| Mac | admin workstation (kubectl, argocd, ssh, browser) | `192.168.99.100`, on a `bridge10x` interface that exists while a VM on the network runs | – | – | – | – | – |
| macOS DHCP (vmnet) | only for the template and fresh clones | pool `.101–.254` | – | – | – | – | – |
| `kali-template` | source of the 7 clones, powered off after Step 4 | DHCP | Xfce | 2 | 2 GB | 16 MB | 25 GB |
| `k3s-lb` | NGINX L4 load balancer (API + ingress) | `192.168.99.10` | Xfce | 2 | 2 GB | 16 MB | 25 GB |
| `k3s-db` | MariaDB external datastore | `192.168.99.11` | Xfce | 2 | 2 GB | 16 MB | 25 GB |
| `k3s-srv-1` | K3s server (control plane) | `192.168.99.21` | Xfce | 2 | 2 GB | 16 MB | 25 GB |
| `k3s-srv-2` | K3s server (control plane) | `192.168.99.22` | Xfce | 2 | 2 GB | 16 MB | 25 GB |
| `k3s-agent-1` | K3s agent (worker) | `192.168.99.31` | Xfce | 2 | 2 GB | 16 MB | 25 GB |
| `k3s-agent-2` | K3s agent (worker) | `192.168.99.32` | Xfce | 2 | 2 GB | 16 MB | 25 GB |
| `k3s-agent-3` | K3s agent (worker) | `192.168.99.33` | Xfce | 2 | 2 GB | 16 MB | 25 GB |
| `kali` | Kali admin workstation ([Appendix E](#appendix-e--kali-linux-desktop-gui-workstation-vm)) | `192.168.99.50` | Xfce | 2 | 2 GB | 16 MB | 25 GB |
| **Total running** (7 cluster VMs + `kali`) | | | | **16** | **16 GB** | | ~60 GB actually used |

**Why the Mac is `.100` and not `.1`.** On macOS, VirtualBox host-only networks are built on Apple's *vmnet* framework.
- The Mac always takes the **lowest address** of the network's range.
- DHCP hands out the addresses above it.

So Step 2 starts the range at `.100`:
- The Mac is `.100`.
- The DHCP pool is `.101–.254`.
- Every static IP of this plan (`.8`–`.50`) stays outside the pool.

No VM uses the Mac's address for anything, because `eth1` has no gateway.

**Sizing: every VM has the `k3s-lb` specification.**
- **The spec:** 2 vCPU, 2 GB RAM, 16 MB VMSVGA video and a 25 GB dynamic disk, on an ARM VM with EFI, a VirtioSCSI disk and Intel 82540EM NICs. These are the settings of the `k3s-lb` VM made in the VirtualBox app.
- **Where it is set:**
  - Step 3 gives the template exactly this, and every clone inherits it.
  - The node table, `K3S_NODES` in `~/Desktop/Working/Tools/k3s-lab.zsh` (Step 1.1), repeats 2 vCPU / 2048 MB for every node.
  - E.1 gives `kali` the same.
- **16 vCPUs on a 10-core Mac is fine.** The VMs are idle most of the time.
- **RAM is the limit.** 8 running VMs × 2 GB = 16 GB, which is all the RAM of a 16 GB Mac. macOS compresses and swaps memory, so the lab still runs, but slowly. Watch *Activity Monitor → Memory → Memory Pressure*. If it turns yellow or red:
  - Keep `kali` off when you don't need it.
  - Switch the desktop off on nodes whose window you don't look at. It stays installed, frees a few hundred MB per VM, and comes back any time:
    - `sudo systemctl set-default multi-user.target && sudo reboot` gives a text console from the next boot.
    - `sudo systemctl set-default graphical.target && sudo reboot` brings the desktop back.
  - Run two agents instead of three. Two are enough, unless you want the Argo CD **HA** manifest, which needs 3 schedulable nodes.
- **32 GB or more:** all 8 VMs run with their desktops at once, with room to spare.
- **Disk:** about 60 GB free, plus the ~4 GB ISO.

The variants reserve three more IPs:
- **Appendix A** adds `k3s-srv-3` at `192.168.99.23`.
- **Appendix B** replaces `k3s-lb` with `k3s-lb-1` at `.8` and `k3s-lb-2` at `.9`, and `.10` becomes their shared virtual IP.

All three are already in the `home.lab` names (Step 4) and the SSH config (Step 6).

Every VM runs **Kali Linux arm64 with the Xfce desktop**, from the same installer ISO in `~/Desktop/Working/Tools/iso`. The desktop comes from ticking **Xfce** on the installer's **Software selection** screen.
- **The 7 cluster VMs** are clones of `kali-template` (Step 3). So they all get its desktop, and the VirtualBox Guest Additions from Step 4 for window resizing and the shared clipboard.
- **`kali`** is installed separately, as the admin workstation ([Appendix E](#appendix-e--kali-linux-desktop-gui-workstation-vm)).
- **In each VM window** you see the graphical **LightDM login**. Log in as `kali` for the desktop; *Applications → Terminal Emulator* opens a terminal.

The lab domain is **`home.lab`**:
- **Cluster VMs:** each gets its FQDN from `set-identity` (for example `k3s-srv-1.home.lab`). It resolves every other lab name through the `/etc/hosts` block it inherits from the template (Step 4).
- **`kali`:** gets `kali.home.lab` from its own installer, and the same block in Appendix E.2.
- **The Mac:** its `/etc/hosts` maps `k3s.home.lab`, `argocd.home.lab` and `whoami.home.lab` to `192.168.99.10`, the load balancer (Step 15). It maps `kali.home.lab` to `192.168.99.50` (Appendix E).

**How many VMs?**

| Scenario | VMs | Which |
|---|---|---|
| Created in VirtualBox | **9** | `kali-template` + 7 cluster VMs + `kali`, all Kali Linux |
| Installed from the Kali ISO | **2** | `kali-template` (Step 3) and `kali` (Appendix E), both with **Xfce** ticked in Software selection; the other 7 are clones of the template, desktop included |
| Running at the same time | **8** | 7 cluster VMs + `kali`; the template stays powered off after Step 4 |
| Needed for the cluster itself | **7** | `k3s-lb`, `k3s-db`, `k3s-srv-1`, `k3s-srv-2`, `k3s-agent-1/2/3` |
| Bare minimum (non-HA Argo CD, no `kali` workstation) | **6** | same, with 2 agents |
| Appendix A: embedded etcd | 7 | `k3s-db` replaced by `k3s-srv-3` |
| Appendix B: HA load balancer | 8 | `k3s-lb` replaced by `k3s-lb-1` + `k3s-lb-2` |
| Appendix A + B together | 8 | 2 LBs + 3 servers + 3 agents, no DB |

### 0.3 Traffic and ports

| From → To | Port | Purpose |
|---|---|---|
| Mac, agents → `k3s-lb` | 6443/tcp | Kubernetes API (kubectl, agent join) |
| Mac → `k3s-lb` | 80, 443/tcp | Ingress: Argo CD UI, your apps |
| `k3s-lb` → `k3s-srv-1/2` | 6443/tcp | API upstreams |
| `k3s-lb` → `k3s-agent-1/2/3` | 80, 443/tcp | Traefik (exposed on every node by K3s ServiceLB) |
| `k3s-srv-1/2` → `k3s-db` | 3306/tcp | K3s datastore (kine → MariaDB) |
| all nodes ↔ all nodes | 8472/udp | Flannel VXLAN (pod network) |
| all nodes ↔ all nodes | 10250/tcp | kubelet (logs, exec, metrics) |
| `kali` → `k3s-lb` | 6443, 80, 443/tcp | Same as the Mac: kubectl, Argo CD UI/CLI, apps |
| Mac → `kali`; `kali` → all VMs | 22/tcp | SSH |
| Mac `127.0.0.1:2222` → `kali-template`; `127.0.0.1:220xx` → each cluster VM; `127.0.0.1:22050` → `kali` | 22/tcp, through a VirtualBox NAT port forward on `eth0` | SSH with copy and paste: `ssh -p <port> kali@127.0.0.1`. Works before the host-only IPs exist (Steps 4, 6 and E.2), and every later step that runs inside a VM starts with it. `xx` = last number of the VM's IP (Step 5) |

Kali enables **no firewall** by default (no `ufw` or `firewalld`), and this lab leaves it that way. The host-only network is reachable only from your Mac. The macOS firewall, if it's on, filters only connections *to* the Mac, and nothing in this guide connects to the Mac.

### 0.4 Order of operations (where you type what)

```mermaid
flowchart TD
    A["Step 1-2 · Mac<br/>Tools folder, PATH, SSH key, host-only network"]
    B["Step 3-4 · kali-template VM<br/>install Kali arm64 + Xfce desktop, prep, Guest Additions, set-identity helper"]
    C["Step 5-6 · Mac + each VM console<br/>clone 7 cluster VMs, give each its name and IP"]
    D["Step 7 · k3s-db<br/>MariaDB, k3s database and user, secrets"]
    E["Step 8 · k3s-lb<br/>NGINX TCP load balancer"]
    F["Step 9 · k3s-srv-1<br/>first K3s server"]
    G["Step 10 · k3s-srv-2<br/>second K3s server"]
    H["Step 11 · k3s-agent-1/2/3<br/>join workers through the LB"]
    I["Step 12-13 · Mac<br/>kubeconfig, HA failover test"]
    J["Step 14-16 · Mac<br/>install, expose and log in to Argo CD"]
    K["Step 17-18 · Mac + GitHub<br/>GitOps apps, app-of-apps"]
    L["Appendix E · Mac + kali window<br/>Kali Xfce desktop (GUI) VM"]
    A --> B --> C --> D --> E --> F --> G --> H --> I --> J --> K
    C -.-> L
```

| Step | Run on | What |
|---|---|---|
| 1–2 | Mac (Terminal) | VirtualBox, kubectl, argocd and the Kali **arm64** ISO in `~/Desktop/Working/Tools` (checksums verified), `k3s-lab.zsh` loaded from `~/.zshrc` (PATH + settings), SSH key; then reset and recreate the VirtualBox networks (host-only network, NAT network) |
| 3 | Mac + template window | create the ARM VM (2 vCPU, 2 GB, 16 MB video, 25 GB) with the SSH port forward `127.0.0.1:2222`, install Kali with the **Xfce desktop** |
| 4 | `kali-template` (3 lines in its desktop terminal, then `ssh -p 2222 kali@127.0.0.1`) | SSH, NetworkManager, `eth0`/`eth1` names, EFI fallback boot path, updates, swap off, `home.lab` names, `set-identity` helper, VirtualBox Guest Additions, pre-flight check, power off |
| 5 | Mac | clone 7 cluster VMs, give each its own SSH port forward, start them |
| 6 | **each VM's console window**, or the Mac through the port forwards | `sudo set-identity <name> <ip>` |
| 7 | `k3s-db` (`ssh -p 22011 kali@127.0.0.1`) | MariaDB, DB and user, generate secrets |
| 8 | `k3s-lb` (`ssh -p 22010 kali@127.0.0.1`) | NGINX stream config |
| 9 | `k3s-srv-1` (`ssh -p 22021 kali@127.0.0.1`) | K3s server #1 |
| 10 | `k3s-srv-2` (`ssh -p 22022 kali@127.0.0.1`) | K3s server #2 |
| 11 | `k3s-agent-1`, `-2`, `-3` (`ssh -p 22031 kali@127.0.0.1`, then port `22032`, `22033`) | K3s agents |
| 12–18 | Mac | kubectl, HA test, Argo CD, GitOps |
| App. E | Mac + `kali` window | create `kali`, install Kali with the **Xfce desktop (GUI)** ticked in Software selection, then network + SSH and the VirtualBox Guest Additions (any time after Step 6); kubectl and argocd (after Steps 12 / 16) |

### 0.5 Conventions

- **`zsh` blocks run in Terminal on your Mac.** zsh has been the macOS default shell since 2019: `echo $SHELL` prints `/bin/zsh`. They also work in iTerm2 or the VS Code terminal.
  - **How to paste:** paste a whole block, then press **Return**. zsh takes a pasted block in one piece and runs it when you press Return.
  - **Where to type:** a block that runs on the Mac must be pasted at your Mac's own prompt, `you@YourMac ~ %`. If a VM session is open in that window, type `exit` first, or open a second tab with **⌘T**.
- **`#` comments in pasted blocks.** Kali's zsh accepts them, but macOS's zsh does not until `setopt interactivecomments` is on. Without it, every comment line fails with `zsh: command not found: #`.
  - Step 1.1 turns it on in every new window, through `~/.zshrc`.
  - **In your very first Terminal window**, type `setopt interactivecomments` once before you paste block 1.1.
- **`bash` blocks run inside the named VM.**
  - Each step that runs inside the template or a cluster VM first shows the `zsh` command that connects to it from the Mac: `ssh -p <port> kali@127.0.0.1`, through that VM's own NAT port forward.
  - Run it in Terminal, then paste the `bash` block into the session.
  - Type `exit` to get back to the Mac before the next `zsh` block (or use a second tab).
  - **The port picks the VM.** `2222` is **only** `kali-template` (Step 3). Each clone has its own port, `22000` + the last number of its IP (Step 5):

    | Port | VM |
    |---|---|
    | `22010` | `k3s-lb` |
    | `22011` | `k3s-db` |
    | `22021` / `22022` | `k3s-srv-1` / `k3s-srv-2` |
    | `22031` / `22032` / `22033` | `k3s-agent-1` / `-2` / `-3` |
    | `22050` | the `kali` desktop VM |

    `ssh -p 2222` never reaches a cluster node.
  - The first connection to each port asks you to confirm the VM's host key: type `yes`.
  - `ssh <vm-name>` (the shortcuts from Step 6) reaches the same VM over the host-only network instead.
- **Read the prompt before you paste.**
  - `kali@k3s-lb:~$` is a VM. `you@YourMac ~ %` is your Mac.
  - VM blocks contain `sudo reboot`. macOS has a `reboot` command too, so **`sudo reboot` pasted into the Mac's own Terminal restarts your Mac**.
  - macOS has no `poweroff`, so `sudo poweroff` on the Mac only fails.
- **Shells inside the VMs.** Kali's default login shell is **zsh**.
  - Step 4 switches the `kali` user to **bash** in the template, so `bash` blocks paste as-is on every clone.
  - The `kali` desktop VM keeps zsh. Kali's zsh config allows `#` comments, so its blocks paste as-is too.
- **Copy and paste.**
  - **On the Mac:** ⌘C / ⌘V.
  - **In a VirtualBox VM window:** the **left ⌘** key is VirtualBox's *Host key*. It releases the keyboard and mouse back to the Mac.
  - **In a VM's desktop terminal** (every VM has one): paste with **Ctrl+Shift+V** (Control, not ⌘). The shared clipboard works once the Guest Additions are in: Step 4 part 3b for the template and its clones, Appendix E.3 for `kali`.
- **zsh and special characters.** zsh treats `?`, `*` and `[...]` as file patterns, and `<word>` as a redirection.
  - The blocks quote such arguments, so keep the quotes.
  - Placeholders in `zsh` blocks are written in CAPITALS, such as `VMNAME`. Replace the whole word.
- **Run on:** at the start of each step tells you where.
- **Network:** if `192.168.99.0/24` is already used on your Mac (home router, VPN), replace `192.168.99.` everywhere and set the new range in Step 2.

---

## Step 1 — Mac: tools, PATH, SSH key

Everything this guide downloads on the Mac goes into **`~/Desktop/Working/Tools`**. That folder is the Mac counterpart of the Windows guide's `C:\Working\tools`. Each download is checked against its publisher's SHA-256 checksum or code signature.

```text
~/Desktop/Working/Tools/
├── bin/          kubectl, argocd (darwin-arm64)                (first on PATH, via ~/.zshrc)
├── iso/          the Kali installer ISO (arm64) used in Step 3 and Appendix E
├── downloads/    the VirtualBox installer (.dmg)
└── k3s-lab.zsh   this guide's shell settings: PATH, zsh options, node table, helper
```

- **Location.** `Tools` sits next to your other projects in `~/Desktop/Working`, so the lab's files never mix with them. Creating it needs no administrator rights.
- **An ISO you already downloaded.** If you saved a Kali ISO straight into `Tools` (for example `Tools/kali-linux-2026.3-installer-arm64.iso`), block 1.1 moves it into `Tools/iso`. Block 1.6 then checks it against Kali's checksum, so nothing is downloaded twice.

> **iCloud Drive.** When *System Settings → Apple Account → iCloud → Drive → Desktop & Documents Folders* is on, everything on the Desktop is uploaded to iCloud, including the ~4 GB ISO.
> - Set `TOOLS` in 1.1 to a folder outside the Desktop, for example `$HOME/Working/Tools`.
> - Everything else follows automatically, because every later block reads `$TOOLS`.

**Why `~/.zshrc` as well as PATH.** macOS builds PATH from `/etc/paths`, and `/usr/local/bin` and `/opt/homebrew/bin` come early in it.
- Docker Desktop puts its own, usually older, `kubectl` into `/usr/local/bin`, as a link into `Docker.app`.
- Homebrew may add another one in `/opt/homebrew/bin`.

Block 1.1 therefore writes `Tools/k3s-lab.zsh`, and adds one line at the **end** of `~/.zshrc` that loads it in every new Terminal window (and in iTerm2 and the VS Code terminal). The file:
- puts `Tools/bin` **first** on PATH;
- turns on `interactivecomments`;
- detects Apple Silicon or Intel;
- holds the node table and the download helper that later steps use.

It needs no administrator rights, and Docker and Homebrew keep working.

Three things are deliberately **not** under `Tools`:

- **VirtualBox.** A macOS app installs into `/Applications/VirtualBox.app` and links its commands (`VBoxManage`, `VirtualBoxVM`) into `/usr/local/bin`. Its `.dmg` installer is kept in `Tools/downloads`.
- **The OpenSSH client** (`ssh`, `scp`, `ssh-keygen`, `ssh-copy-id`) is part of macOS (`/usr/bin`). Block 1.5 makes sure it is there.
- **git** comes from Apple's *Command Line Tools* (`/usr/bin/git`), installed by block 1.4.

**Run on:** Mac, in Terminal. Run blocks **1.1 to 1.7 in order**.
- **Same window:** they work in one window. 1.1 also loads its settings into that window, and every new window gets them from `~/.zshrc`.
- **Password prompt:** the VirtualBox installer in 1.2 asks for your Mac password (it uses `sudo`).
- **Safe to run again:** anything already present with the right checksum is skipped, so a rerun only updates what changed.

**1.1 The `Tools` folders, `k3s-lab.zsh`, PATH and helpers**

In a new Terminal window, type this line first. Every new window gets it from 1.1:

```zsh
setopt interactivecomments
```

Then paste:

```zsh
# Where every tool goes: the Mac counterpart of C:\Working\tools
TOOLS="$HOME/Desktop/Working/Tools"
mkdir -p "$TOOLS"/{bin,iso,downloads}

# A Kali ISO saved straight into Tools (e.g. by the browser) moves into Tools/iso, so nothing is downloaded twice
for f in "$TOOLS"/kali-linux-*.iso(N); do mv -v "$f" "$TOOLS/iso/"; done

# this guide's shell settings (rewritten on every run of 1.1, so edit K3S_NODES here, in this block)
{ print -r -- "export TOOLS=${(q)TOOLS}"; cat <<'EOF'
# --- k3s-argocd-setup-mac.md, Step 1.1 - loaded by ~/.zshrc in every new Terminal window
typeset -U path; path=("$TOOLS/bin" $path)  # Tools/bin first: before Docker's or Homebrew's kubectl
setopt interactivecomments                   # '# comments' in pasted blocks

# this Mac's CPU picks the VirtualBox platform, the Kali ISO and the kubectl/argocd builds
if [[ $(uname -m) == arm64 ]]; then          # Apple Silicon: ARM VMs (EFI, VirtioSCSI disk)
  VBOX_ARCH=arm VBOX_OSTYPE=Debian_arm64 VBOX_CTL=VirtioSCSI VBOX_DMG=macOSArm64 KALI_ARCH=arm64
else                                         # Intel Mac: x86 VMs (BIOS, SATA disk)
  VBOX_ARCH=x86 VBOX_OSTYPE=Debian_64    VBOX_CTL=SATA       VBOX_DMG=OSX        KALI_ARCH=amd64
fi
HON=k3s-hostonly                             # the VirtualBox host-only network (Step 2)

# the 7 cluster VMs: name, vCPU, RAM in MB, host-only IP, NAT port (22000 + last number of the IP)
# every VM has the k3s-lb spec: 2 vCPU, 2048 MB (16 MB video and the 25 GB disk come from the template, Step 3)
K3S_NODES=(
  "k3s-lb       2  2048  192.168.99.10  22010"
  "k3s-db       2  2048  192.168.99.11  22011"
  "k3s-srv-1    2  2048  192.168.99.21  22021"
  "k3s-srv-2    2  2048  192.168.99.22  22022"
  "k3s-agent-1  2  2048  192.168.99.31  22031"
  "k3s-agent-2  2  2048  192.168.99.32  22032"
  "k3s-agent-3  2  2048  192.168.99.33  22033"
)

# get_verified URL FILE SHA256: download unless FILE is already there with that SHA-256; delete it on a mismatch
get_verified() {
  local url=$1 out=$2 want=${3:l}
  [[ -n $want ]] || { print -u2 "no checksum found for $url"; return 1; }
  if [[ -f $out && $(shasum -a 256 "$out" | cut -d' ' -f1) == $want ]]; then print "up to date : $out"; return 0; fi
  curl -fL --retry 3 -o "$out" "$url" || { print -u2 "download failed: $url"; return 1; }
  if [[ $(shasum -a 256 "$out" | cut -d' ' -f1) != $want ]]; then rm -f "$out"; print -u2 "SHA-256 mismatch, deleted $out"; return 1; fi
  print "verified   : $out"
}
EOF
} > "$TOOLS/k3s-lab.zsh"

# load it at the END of ~/.zshrc in every new window (an older copy of the line is replaced), and now in this window
touch ~/.zshrc
sed -i '' '/k3s-argocd-setup/d' ~/.zshrc
print -r -- "[[ -r ${(q)TOOLS}/k3s-lab.zsh ]] && source ${(q)TOOLS}/k3s-lab.zsh   # k3s-argocd-setup" >> ~/.zshrc
source "$TOOLS/k3s-lab.zsh"
print -l "TOOLS=$TOOLS" "CPU $(uname -m): VirtualBox platform $VBOX_ARCH, Kali $KALI_ARCH" "PATH starts with: $path[1]"
```

The last lines must show your `Tools` folder, `CPU arm64: VirtualBox platform arm, Kali arm64` (on Apple Silicon), and `PATH starts with: …/Tools/bin`.

**1.2 VirtualBox** (Oracle's macOS installer; Apple Silicon is supported since VirtualBox 7.1)

```zsh
() {   # a function, so 'return' stops the block at the first problem
  local VBOX_BRANCH=7.2        # the branch this guide was written for
  local app=/Applications/VirtualBox.app/Contents/MacOS/VBoxManage
  if [[ -x $app ]]; then print "VirtualBox already installed ($($app --version)) - keeping it"; return 0; fi

  # newest release of $VBOX_BRANCH, checked against Oracle's SHA256SUMS and Oracle's installer signature
  local ver=$(curl -fsSL https://download.virtualbox.org/virtualbox/ | grep -oE "href=\"${VBOX_BRANCH//./\\.}\.[0-9]+/\"" | grep -oE '[0-9]+\.[0-9]+\.[0-9]+' | sort -V | tail -1)
  local line=$(curl -fsSL "https://download.virtualbox.org/virtualbox/$ver/SHA256SUMS" | grep -E "\*VirtualBox-[0-9.]+-[0-9]+-$VBOX_DMG\.dmg$" | head -1)
  local dmg="$TOOLS/downloads/${line#*\*}"
  get_verified "https://download.virtualbox.org/virtualbox/$ver/${line#*\*}" "$dmg" "${line%% *}" || return 1

  # mount the .dmg read-only, check that the .pkg inside is signed by Oracle, install it (asks for your Mac password)
  local mnt=$(mktemp -d)
  hdiutil attach -nobrowse -readonly -mountpoint "$mnt" "$dmg" >/dev/null || return 1
  local pkg=( "$mnt"/*.pkg(N) )
  if ! pkgutil --check-signature "$pkg[1]" | grep -q 'Oracle America, Inc. (VB5E2TV963)'; then
    hdiutil detach "$mnt" >/dev/null; print -u2 "$pkg[1] is not signed by Oracle - NOT installed"; return 1
  fi
  sudo installer -pkg "$pkg[1]" -target /
  hdiutil detach "$mnt" >/dev/null
}
VBoxManage --version
```

> **Intel Mac:** macOS blocks VirtualBox's kernel extension the first time. Open *System Settings → Privacy & Security*, click **Allow** next to *Oracle America, Inc.*, then restart the Mac. Apple Silicon needs no kernel extension: VirtualBox uses Apple's Hypervisor framework.

**1.3 kubectl and the Argo CD CLI → `Tools/bin`**

```zsh
() {
  local arch=$KALI_ARCH      # arm64 (Apple Silicon) or amd64 (Intel): the names the Kubernetes and Argo CD downloads use
  # kubectl: current stable release, checked against the .sha256 file published next to it
  local k8s=$(curl -fsSL https://dl.k8s.io/release/stable.txt)
  local ksha=$(curl -fsSL "https://dl.k8s.io/release/$k8s/bin/darwin/$arch/kubectl.sha256")
  get_verified "https://dl.k8s.io/release/$k8s/bin/darwin/$arch/kubectl" "$TOOLS/bin/kubectl" "$ksha" || return 1
  # argocd: latest release, checked against the release's cli_checksums.txt
  local asha=$(curl -fsSL https://github.com/argoproj/argo-cd/releases/latest/download/cli_checksums.txt | awk -v f="argocd-darwin-$arch" '$2 == f {print $1}')
  get_verified "https://github.com/argoproj/argo-cd/releases/latest/download/argocd-darwin-$arch" "$TOOLS/bin/argocd" "$asha" || return 1
  chmod 755 "$TOOLS/bin/kubectl" "$TOOLS/bin/argocd"
}
```

Files that `curl` downloads carry no quarantine flag, so macOS runs them without a Gatekeeper prompt. If you download one with a browser instead, macOS blocks it: see Appendix C.

**1.4 git** (Apple's Command Line Tools; nothing to download by hand)

```zsh
if xcode-select -p >/dev/null 2>&1; then git --version; else xcode-select --install; fi
```

If a dialog opens, click **Install**, wait until it finishes, then run the line again: it must print `git version …`. Xcode, if installed, provides git as well.

**1.5 OpenSSH client (part of macOS)**

```zsh
for c in ssh scp ssh-keygen ssh-copy-id; do printf '%-12s %s\n' $c "$(whence -p $c || print MISSING)"; done
ssh -V
```

**1.6 Kali installer ISO → `Tools/iso`** (used in Step 3 and Appendix E)

```zsh
() {
  local image=installer     # the full image (~4 GB): the Xfce desktop installs from it offline. "installer-netinst" = small, fetches packages
  local line=$(curl -fsSL https://cdimage.kali.org/current/SHA256SUMS | grep -E "  kali-linux-[0-9.]+[a-z]?-$image-$KALI_ARCH\.iso$" | head -1)
  get_verified "https://cdimage.kali.org/current/${line##* }" "$TOOLS/iso/${line##* }" "${line%% *}"
}
```

On Apple Silicon this is the `…-installer-arm64.iso`. If it is already in `Tools/iso` (1.1 moved it there), the block only checks it and prints `up to date`. That holds as long as the file is still Kali's *current* release, so a newer release is downloaded next to it.

**1.7 Verify.** Open a **new** Terminal window (⌘N) so it loads `~/.zshrc`, then run:

```zsh
for c in VBoxManage kubectl argocd git ssh scp ssh-keygen ssh-copy-id curl shasum; do
  src=$(whence -p $c)
  if   [[ -z $src ]];                                         then tag=MISSING
  elif [[ $src == $TOOLS/* ]];                                then tag="OK  (Tools)"
  elif [[ $src == /usr/local/bin/VBox* ]];                    then tag="OK  (VirtualBox)"
  elif [[ $src == /usr/bin/* || $src == /bin/* || $src == /usr/sbin/* ]]; then tag="OK  (macOS)"
  else tag="WARN another copy is found first"; fi
  printf '%-11s %-33s %s\n' $c "$tag" "$src"
done
VBoxManage --version
kubectl version --client
argocd version --client --short
git --version
ls -lh "$TOOLS"/iso/*.iso(N)
```

Every tool must show **`OK`**:

```text
VBoxManage  OK  (VirtualBox)                  /usr/local/bin/VBoxManage
kubectl     OK  (Tools)                       /Users/you/Desktop/Working/Tools/bin/kubectl
argocd      OK  (Tools)                       /Users/you/Desktop/Working/Tools/bin/argocd
git         OK  (macOS)                       /usr/bin/git
ssh         OK  (macOS)                       /usr/bin/ssh
...
```

- **`MISSING`**: run that tool's block again (1.2–1.5), then open a new window.
- **`WARN`**: another copy is found before `Tools/bin`, for example Docker Desktop's `kubectl` (`/usr/local/bin/kubectl`) or Homebrew's (`/opt/homebrew/bin/kubectl`).
  - **The fix** is the `~/.zshrc` line from 1.1. Run 1.1 again and open a new window.
  - **If it still says `WARN`**, something later in `~/.zshrc` puts its own folder in front again:
    - `tail -3 ~/.zshrc` must show the `k3s-argocd-setup` line last.
    - `which -a kubectl` lists every copy in search order.
  - **Why it matters for `kubectl`:** it is supported only within one minor version of the cluster, and Docker Desktop's copy is usually several versions older. `kubectl version --client` must print the version that 1.3 downloaded.
  - **The line only affects zsh.** bash or fish still find the other `kubectl` first.
- **Upgrading later:** run 1.1, 1.3 and 1.6 again. Tools that are already current are left alone.

Create an SSH key pair (press Return to accept the default path; a passphrase is optional):

```zsh
[[ -f ~/.ssh/id_ed25519 ]] || ssh-keygen -t ed25519
```

> With a passphrase, `ssh-add --apple-use-keychain ~/.ssh/id_ed25519` stores it in the macOS Keychain, so you type it once instead of at every `ssh`.

---

## Step 2 — Mac: host-only network and DHCP pool

**Run on:** Mac, in Terminal. No `sudo` needed. Every VM must be powered off.

The block always starts from empty, so it gives the same result whatever VirtualBox has now. It removes **every** VirtualBox host-only network, NAT network and DHCP server, then creates exactly what this guide needs:

| Network | Created as | Used by |
|---|---|---|
| Host-only network | `k3s-hostonly`, `192.168.99.0/24`, range `.100`–`.254` | NIC 2 (`eth1`) of every VM |
| The Mac's address on it | `192.168.99.100` (the range's lower IP), on a `bridge10x` interface | kubectl, ssh and the browser on the Mac |
| Its DHCP pool | `.101–.254`, served by macOS itself | Only the template and fresh clones. `.2–.99` stay free for the static IPs (`.10`–`.33`, `.8`/`.9` for Appendix B, `.50` for Kali) |
| NAT network | `NatNetwork`, `10.0.2.0/24` with DHCP (VirtualBox's default) | Nothing in this guide. NIC 1 uses plain **NAT**, which needs no network here. It is recreated so that VirtualBox's *Network Manager* is in a known state |

- **Host-only networks, not adapters.** Since VirtualBox 7 on macOS, *host-only networks* (`VBoxManage hostonlynet`) replace the old `vboxnet0` *host-only adapters*. They use Apple's vmnet framework, which is why the Mac takes the lower IP.
- **When to run it again:** whenever the network is broken, for example after removing it in *VirtualBox → Tools → Network*. VMs refer to the network by name, and the name comes back the same.

> ⚠️ Other VirtualBox VMs on this Mac that use a host-only or NAT network of their own lose those settings. Plain NAT is not affected.

```zsh
() {   # a function, so it stops at the first problem
  local NATNET=NatNetwork n
  if [[ -n $(VBoxManage list runningvms) ]]; then print -u2 "Power off every VM first: VBoxManage list runningvms"; return 1; fi

  # 1) remove every host-only network, old host-only adapter, NAT network and DHCP server
  for n in ${(f)"$(VBoxManage list hostonlynets | sed -n 's/^Name: *//p')"}; do VBoxManage hostonlynet remove --name="$n"; done
  for n in ${(f)"$(VBoxManage list hostonlyifs  | sed -n 's/^Name: *//p')"}; do VBoxManage hostonlyif remove "$n"; done
  for n in ${(f)"$(VBoxManage list natnets      | sed -n -E 's/^(Network)?Name: *//p')"}; do VBoxManage natnetwork remove --netname "$n"; done
  for n in ${(f)"$(VBoxManage list dhcpservers  | sed -n 's/^NetworkName: *//p')"}; do VBoxManage dhcpserver remove --network "$n"; done

  # 2) 192.168.99.x must not be used by anything else on this Mac (home router, VPN, another VM app)
  if ifconfig | grep -qE 'inet 192\.168\.99\.'; then
    ifconfig | grep -E 'inet 192\.168\.99\.'
    print -u2 "192.168.99.x is already used on this Mac (line above). Pick another network, see 0.5 Conventions"; return 1
  fi

  # 3) the host-only network: the Mac gets the lower IP (.100), macOS's DHCP hands out .101-.254
  VBoxManage hostonlynet add --name="$HON" --netmask=255.255.255.0 --lower-ip=192.168.99.100 --upper-ip=192.168.99.254 --enable || return 1

  # 4) VirtualBox's default NAT network (the VMs in this guide don't use it)
  VBoxManage natnetwork add --netname "$NATNET" --network "10.0.2.0/24" --enable --dhcp on

  # 5) check
  VBoxManage list hostonlynets | grep -E '^(Name|State|NetworkMask|LowerIP|UpperIP):'
  VBoxManage list natnets      | grep -E '^(Name|Network|DHCP Server):'
  VBoxManage list hostonlynets | grep -qE '^LowerIP: +192\.168\.99\.100$' &&
    VBoxManage list hostonlynets | grep -qE '^UpperIP: +192\.168\.99\.254$' &&
    VBoxManage list natnets | grep -qE "^Name: +${NATNET}\$" || { print -u2 "Step 2 check failed: compare with the expected output below"; return 1; }
  print "Step 2 OK"
}
```

Expected output at the end:

```text
Name:            k3s-hostonly
State:           Enabled
NetworkMask:     255.255.255.0
LowerIP:         192.168.99.100
UpperIP:         192.168.99.254
Name:         NatNetwork
Network:      10.0.2.0/24
DHCP Server:  Yes
Step 2 OK
```

- **No `192.168.99.100` in `ifconfig` yet:** that is expected. macOS creates the `bridge10x` interface with that address when the first VM on `k3s-hostonly` starts, and removes it when the last one stops.
- **No `dhcpserver` commands:** a host-only network on macOS has no VirtualBox DHCP server. macOS's own DHCP service serves the range, and keeps its leases in `/var/db/dhcpd_leases`.

---

## Step 3 — Mac: create the template VM and install Kali

You install Kali into `kali-template` with the **Xfce desktop (GUI)** and the `k3s-lb` specification: 2 vCPU, 2 GB RAM, 16 MB video, 25 GB disk.
- Step 5 then clones it 7 times, once per cluster node, so every node gets the same desktop and specification.
- The `kali` workstation is a second install from the same ISO, with the same desktop and specification ([Appendix E](#appendix-e--kali-linux-desktop-gui-workstation-vm)).

1. Step 1.6 already verified the **Kali Linux installer** ISO in **`~/Desktop/Working/Tools/iso`**.
   - To fetch it by hand instead, download it from <https://www.kali.org/get-kali/#kali-installer-images> into that folder. Pick **Apple Silicon (ARM64)**: the full **Installer** (`kali-linux-20xx.x-installer-arm64.iso`) or the **NetInstaller** (`…-installer-netinst-arm64.iso`).
   - An `amd64` ISO does not boot on Apple Silicon.

2. **Run on:** Mac:

```zsh
() {
  local VM=kali-template
  local ISO=( "$TOOLS"/iso/kali-linux-*-installer*-$KALI_ARCH.iso(N) ); ISO=$ISO[-1]
  local VDI="$HOME/VirtualBox VMs/$VM/$VM.vdi"
  print "ISO: $ISO"     # must print a path under Tools/iso ending in -arm64.iso (Apple Silicon)
  [[ -f $ISO ]] || { print -u2 "no Kali $KALI_ARCH installer ISO in $TOOLS/iso - run Step 1.6"; return 1; }

  # --default = VirtualBox's recommended settings for this OS type. On Apple Silicon: an ARM VM with EFI,
  # USB keyboard + tablet (ARM has no PS/2), VMSVGA graphics and a VirtioSCSI disk controller
  VBoxManage createvm --name $VM --platform-architecture=$VBOX_ARCH --ostype $VBOX_OSTYPE --default --register || return 1
  VBoxManage modifyvm $VM --cpus 2 --memory 2048 --vram 16 --graphicscontroller vmsvga --audio-enabled off   # the k3s-lb spec, inherited by every clone
  VBoxManage modifyvm $VM --clipboard-mode bidirectional --drag-and-drop bidirectional                      # shared clipboard for the desktops (Step 4 part 3b)
  VBoxManage modifyvm $VM --nic1 nat --nic2 hostonlynet --host-only-net2 $HON
  VBoxManage modifyvm $VM --natpf1 "ssh,tcp,127.0.0.1,2222,,22"      # NAT port forward: Mac 127.0.0.1:2222 -> template port 22 (Step 4)
  VBoxManage modifyvm $VM --boot1 disk --boot2 dvd --boot3 none --boot4 none
  VBoxManage createmedium disk --filename "$VDI" --size 25600 --format VDI
  VBoxManage storagectl $VM --name $VBOX_CTL --portcount 2
  VBoxManage storageattach $VM --storagectl $VBOX_CTL --port 0 --device 0 --type hdd --medium "$VDI"
  VBoxManage storageattach $VM --storagectl $VBOX_CTL --port 1 --device 0 --type dvddrive --medium "$ISO"
  VBoxManage startvm $VM --type gui
}
```

> The boot order is **disk first, then DVD**. The empty disk falls through to the ISO; after the install the VM boots from disk. If it shows the installer again, or stops at a UEFI `Shell>` prompt, see Appendix C.

> **Intel Mac:** the same block makes an x86 VM (BIOS, SATA controller) from Kali's amd64 ISO, because `k3s-lab.zsh` set `VBOX_ARCH=x86`.

> **Port forward for SSH.** The VM window has no shared clipboard until the Guest Additions are in (Step 4 part 3b). Step 4 therefore works over SSH from a Terminal window on the Mac, where copy and paste work normally. The `--natpf1` line is the command-line form of *Settings → Network → Adapter 1 (NAT) → Port Forwarding → +*:
>
> | Name | Protocol | Host IP | Host Port | Guest IP | Guest Port |
> |---|---|---|---|---|---|
> | `ssh` | TCP | `127.0.0.1` | `2222` | *(blank)* | `22` |
>
> - **When it works:** while the VM is running and its SSH server is on (Step 4 part 1).
> - **Who can use it:** Host IP `127.0.0.1` means only your own Mac.
> - **Clones:** every clone inherits this rule, so Step 5 gives each clone its own port.
>
> **`kali-template` already exists without the rule?** Add it now:
> - while it is powered off: `VBoxManage modifyvm kali-template --natpf1 "ssh,tcp,127.0.0.1,2222,,22"`
> - while it runs: `VBoxManage controlvm kali-template natpf1 "ssh,tcp,127.0.0.1,2222,,22"`
>
> Check with `VBoxManage showvminfo kali-template --machinereadable | grep '^Forwarding'`, which must print `Forwarding(0)="ssh,tcp,127.0.0.1,2222,,22"`.

3. **In the VM window**, go through the Kali installer (it is the Debian installer). Click into the window to type; the **left ⌘** key gives the keyboard and mouse back to the Mac.

| Screen | Choose |
|---|---|
| Boot menu | **Graphical install** (or **Install**: the same screens in text mode) |
| Language / location / keyboard | your choice |
| Primary network interface | the **first** interface in the list, shown as `eth0` (or `enp0s…`), which is Adapter 1 = NAT. Both entries say *Intel 82540EM*, so go by the order: the second one is the host-only NIC and has no internet |
| Hostname | `kali-template` |
| Domain name | **`home.lab`** |
| Full name / username | `kali` / **`kali`** · a password (every clone inherits it) |
| Time zone | your choice (Step 4 sets UTC) |
| Partition disks | **Guided - use entire disk** → the 25 GB disk → **All files in one partition** → Finish → Yes (on Apple Silicon this also creates the small EFI partition) |
| HTTP proxy / mirror | leave empty / defaults |
| **Software selection** | keep **Xfce (Kali's default desktop environment)** ticked: this is the **GUI**, and every clone inherits it. **Untick** the tool collections (*top10*, *default*, *large*): the cluster doesn't need them, and *default* alone takes about 13 GB of the 25 GB disk. [E.1](#e1-install-kali-with-the-xfce-desktop) shows the whole screen |
| Boot loader | **Apple Silicon (EFI):** there is no GRUB disk question. If it asks *Force GRUB installation to the EFI removable media path?*, answer **Yes**. **Intel Mac:** **Yes** → `/dev/sda` |
| Finish | **Continue** (the VM reboots from disk) |

---

## Step 4 — kali-template: prepare the golden image

Everything you do here is inherited by all 7 clones.

1. **In the VM window**, the template shows the graphical **LightDM login**. Log in as `kali`, open **Applications → Terminal Emulator** on the Xfce desktop, and turn on SSH. The window has no shared clipboard yet, so type these three lines. They are the only ones you type there; everything after them is pasted over SSH:

```bash
sudo apt update
sudo apt install -y openssh-server
sudo systemctl enable --now ssh
```

`systemctl is-active ssh` must print `active`. Your Mac can now reach the template through the Step 3 port forward, `127.0.0.1:2222`.

2. **Run on:** Mac, in Terminal. This block:
   - adds an SSH shortcut `kali-template` for that port forward;
   - forgets the host keys of any earlier template install;
   - copies your SSH key into the template;
   - connects.

   `ssh-copy-id` asks for the `kali` password you set in the installer:

```zsh
mkdir -p ~/.ssh && chmod 700 ~/.ssh && touch ~/.ssh/config && chmod 600 ~/.ssh/config
# SSH shortcut "kali-template" = the Step 3 port forward 127.0.0.1:2222 -> port 22 (added only once)
grep -qx 'Host kali-template' ~/.ssh/config || cat >> ~/.ssh/config <<'EOF'

Host kali-template
    HostName 127.0.0.1
    Port 2222
    User kali
    HostKeyAlias kali-template
    StrictHostKeyChecking accept-new
EOF
# Step 3 installed a NEW system with NEW host keys. Forget the keys of any earlier install,
# otherwise ssh stops with "REMOTE HOST IDENTIFICATION HAS CHANGED" (prints the line it removed, if any)
ssh-keygen -R kali-template 2>/dev/null
ssh-keygen -R "[127.0.0.1]:2222" 2>/dev/null
# copy your Mac's SSH key into the template (asks for the kali password once; skips a key that is already there)
ssh-copy-id -i ~/.ssh/id_ed25519.pub kali-template
# the key must work on its own now: password logins are switched off for this one test
ssh -o PasswordAuthentication=no -o KbdInteractiveAuthentication=no kali-template "echo 'SSH key login: OK'"
ssh kali-template      # logs in with the key now - no password (same VM, same port as: ssh -p 2222 kali@127.0.0.1)
```

The check line must print **`SSH key login: OK`**. If it says *Permission denied (publickey,password)*, your key did not reach the template: run the `ssh-copy-id` line again and enter the `kali` password when it asks. Don't continue with only a password login. Every clone inherits the template's `authorized_keys`, and Step 6 needs the key.

You now have the template's shell in a Terminal window, where copy and paste work:
- **Paste into the VM:** **⌘V**.
- **Copy from the VM:** select the text with the mouse, then **⌘C**.

> `HostKeyAlias kali-template` saves the template's SSH host key under the name `kali-template` instead of `[127.0.0.1]:2222`. The clones keep that same host key until `set-identity` replaces it, which lets Step 6 option B check that it is talking to a fresh clone.

> **Why the two `ssh-keygen -R` lines.**
> - **The problem:** every Kali install creates new SSH host keys, but `~/.ssh/known_hosts` still holds the keys of an earlier install. They are saved under `kali-template`, and under `[127.0.0.1]:2222` if you ever ran `ssh -p 2222 kali@127.0.0.1`.
> - **The symptom:** SSH refuses with *WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED … Host key verification failed*, even with `accept-new`, which only accepts hosts it has never seen.
> - **Why removing them is safe:** `127.0.0.1:2222` is a port forward on your own Mac to the VM you just installed.
> - **To compare fingerprints anyway:** run `ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub` in the VM window. It must print the `SHA256:` value that `ssh` shows on its first connection.

Still in that SSH session: hand **both** NICs to NetworkManager, fix the NIC names to `eth0` / `eth1`, and (Apple Silicon) put GRUB on the EFI fallback path. Type `sudo -v` and enter the `kali` password first, so no password prompt can interrupt the paste. Then paste:

```bash
sudo apt install -y network-manager
printf 'auto lo\niface lo inet loopback\n' | sudo tee /etc/network/interfaces
sudo rm -f /var/lib/NetworkManager/no-auto-default.state
sudo mkdir -p /etc/default/grub.d
echo 'GRUB_CMDLINE_LINUX="$GRUB_CMDLINE_LINUX net.ifnames=0"' | sudo tee /etc/default/grub.d/90-eth-names.cfg
sudo update-grub
# Apple Silicon (EFI): also install GRUB to the fallback path EFI/BOOT/BOOTAA64.EFI, and keep it there on GRUB updates,
# so the disk (and every clone of it) boots even when VirtualBox's EFI has no boot entry for it. No-op on Intel Macs
if [ -d /sys/firmware/efi ] && dpkg -s grub-efi-arm64 >/dev/null 2>&1; then
  echo 'grub-efi-arm64 grub2/force_efi_extra_removable boolean true' | sudo debconf-set-selections
  sudo grub-install --target=arm64-efi --efi-directory=/boot/efi --force-extra-removable
fi
sudo reboot
```

The reboot closes the session (*Connection to 127.0.0.1 closed by remote host*).
1. Wait about 30 seconds.
2. Connect again with `ssh kali-template` (or `ssh -p 2222 kali@127.0.0.1`).
3. Run `ip -br a`. `eth0` has `10.0.2.15`, and `eth1` has a DHCP address from the host-only pool, for example `192.168.99.101`.

> **`ssh kali-template` gets no answer after the reboot?** Look at the VM window. `eth0` is probably `UP` but has no address. NetworkManager saw the NAT NIC already configured by the installer, so it never created a profile for it.
> - The port forward runs over `eth0`, so type the fix in the VM window: `sudo rm -f /var/lib/NetworkManager/no-auto-default.state; sudo nmcli connection add type ethernet ifname eth0 con-name nat autoconnect yes; sudo nmcli connection up nat`.
> - The clones are not affected, because `set-identity` later writes its own `eth0` profile.

> `net.ifnames=0` gives every clone the same names: **`eth0` = Adapter 1 (NAT)** and **`eth1` = Adapter 2 (host-only)**. The rest of the guide relies on them.

> **Why the EFI fallback path.** ARM VMs always boot through EFI. VirtualBox keeps the EFI boot entry that the installer created in the VM's `.nvram` file. A clone, or a VM whose `.nvram` was reset, can lose that entry and stop at the UEFI `Shell>`. `EFI/BOOT/BOOTAA64.EFI` is the file EFI tries when it has no entry. The `debconf` line keeps that copy current when Kali updates GRUB.

> The host-only network works for SSH too: `ssh kali@192.168.99.101`, with the `eth1` address that `ip -br a` shows. The port forward is the better choice for the template, because that DHCP address can change, while `127.0.0.1:2222` stays the same.

3. **Run on:** `kali-template`, in the SSH session from part 2. If you closed it, connect again from the Mac:

```zsh
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
#    lines a paste put above it, and stray carriage returns (\r)
sudo sed -i -e 's/\r$//' -e '/./,$!d' /usr/local/sbin/set-identity
```

3b. **Apple Silicon: VirtualBox Guest Additions for the desktop.** They make each VM's desktop follow its window size, and share the clipboard with the Mac (Step 3 turned it on). Installed once here, every clone inherits them.
- Kali publishes no arm64 build of its `virtualbox-guest-x11` package, so this uses the Guest Additions CD that ships with VirtualBox (`VBoxLinuxAdditions-arm64.run`).
- **Intel Mac:** skip 3b. The installer already added `virtualbox-guest-x11` together with Xfce.

**Run on:** `kali-template`, same SSH session. Install the build tools and the headers of the newest kernel (item 2 above may have installed a new one), then reboot into it:

```bash
sudo DEBIAN_FRONTEND=noninteractive apt install -y build-essential dkms bzip2 linux-headers-arm64
sudo reboot
```

**Run on:** Mac. Insert the Guest Additions CD, wait about 30 seconds for the reboot, then connect again:

```zsh
VBoxManage storageattach kali-template --storagectl $VBOX_CTL --port 1 --device 0 --type dvddrive --medium additions
ssh kali-template
```

**Run on:** `kali-template`. This builds the kernel modules for the running kernel, which takes a few minutes:

```bash
sudo mount -o ro /dev/sr0 /mnt
sudo sh /mnt/VBoxLinuxAdditions-arm64.run
sudo umount /mnt
sudo reboot
```

**Run on:** Mac. Eject the CD, so the clones don't inherit it. Then connect again for part 4:

```zsh
VBoxManage storageattach kali-template --storagectl $VBOX_CTL --port 1 --device 0 --type dvddrive --medium emptydrive
ssh kali-template
```

The template's desktop now follows its window size (*View → Auto-resize Guest Display*). When Kali later installs a new kernel, the Guest Additions rebuild their modules for it, as long as `linux-headers-arm64` stays installed.

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
chk "Mac SSH key authorized"           'grep -qE "^(ssh|ecdsa|sk)-" ~/.ssh/authorized_keys'
chk "Xfce desktop (graphical boot)"    '[ "$(systemctl get-default)" = graphical.target ] && systemctl is-active --quiet lightdm'
chk "Guest Additions (Apple Silicon)"  '[ "$(dpkg --print-architecture)" != arm64 ] || lsmod | grep -q "^vboxguest"'
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
| Adapter 2 = host-only network, so `eth1` reaches `192.168.99.0/24` | Step 3 (`--nic2 hostonlynet --host-only-net2 k3s-hostonly`), copied by every clone | on the Mac, `VBoxManage showvminfo kali-template --machinereadable \| grep -E '^(nic2\|hostonly-network2)='` must print `nic2="hostonlynetwork"` and `hostonly-network2="k3s-hostonly"`; check Step 2, then `sudo reboot` in the VM |
| your Mac's SSH key in `~/.ssh/authorized_keys`, which every clone inherits (the Step 6 check, Step 6 option B, and every later `ssh <vm>`) | part 2: the `ssh-copy-id … kali-template` line | Run that line again on the Mac and enter the `kali` password. Then the part 2 check line must print `SSH key login: OK` |

`set-identity` doesn't need the last two check lines, but every clone inherits what they test:
- ***Xfce desktop*** `MISSING`: Xfce was not ticked in Step 3. See the Appendix C row *A VM boots to a text login*.
- ***Guest Additions*** `MISSING`: run part 3b.

5. When every line says `OK`, clean up and power off. The template is never booted again, unless you have to fix it:

```bash
sudo apt clean
sudo poweroff
```

> **Never** run `set-identity` with arguments on the template itself.
> - The check above calls it without arguments, which only prints its usage line.
> - If you run it on the template anyway, it stops with *this is kali-template itself* and changes nothing.
> - A template built before this guard existed doesn't stop: see *The template got a clone's identity* in Appendix C.
>
> `sudo poweroff` is a Linux command for this SSH session or the VM window.
> - In your **Mac's** Terminal it only fails (*command not found*), but `sudo reboot` there would restart the Mac.
> - To power the template off from the Mac, run `ssh kali-template "sudo poweroff"`.

**Already cloned, and a clone says `sudo: set-identity: command not found`?** The clones were made before part 3 finished. Nothing is on them yet, so delete them and fix the template. **Run on:** Mac:

```zsh
for row in $K3S_NODES; do
  VBoxManage controlvm ${row%% *} poweroff 2>/dev/null
  VBoxManage unregistervm ${row%% *} --delete
done
# make sure the template has exactly the Step 3 port forward (127.0.0.1:2222 -> 22)
VBoxManage modifyvm kali-template --natpf1 delete ssh 2>/dev/null
VBoxManage modifyvm kali-template --natpf1 "ssh,tcp,127.0.0.1,2222,,22"
VBoxManage startvm kali-template --type gui
```

Then:

1. Wait for the login prompt in the template window. If SSH isn't on yet (`systemctl is-active ssh` doesn't print `active`), type part 1 there.
2. On the Mac, run the `zsh` block of part 2 again (it is safe to run twice). It adds the `kali-template` shortcut if it is missing, copies your SSH key and connects. The `eth1` address doesn't matter, because the connection goes through `127.0.0.1:2222`.
3. Run part 3 again (it is safe to run twice), and part 3b if its check line says `MISSING`. Then run the check in part 4 and the power-off in part 5.
4. Repeat Step 5.

---

## Step 5 — Mac: clone the 7 cluster VMs

**Run on:** Mac. The node table, `K3S_NODES`, comes from `Tools/k3s-lab.zsh` (Step 1.1), so every Terminal window has it.
- `print -l $K3S_NODES` shows it.
- To change a VM's vCPU or RAM, edit the table in block 1.1 and run 1.1 again **before** this step.

```zsh
() {
  # kali-template must be powered off: a running VM keeps its disk locked ("Failed to lock source media")
  local state=$(VBoxManage showvminfo kali-template --machinereadable | grep '^VMState=')
  if [[ $state != 'VMState="poweroff"' ]]; then
    print -u2 "WARNING: kali-template is not powered off ($state), so NOTHING was cloned - don't start Step 6. Power it off from this window with: ssh kali-template 'sudo poweroff'  (or type 'sudo poweroff' in the kali-template VM window, never in the Mac's Terminal). Wait until its window closes, then run this block again."
    return 1
  fi
  local existing=$(VBoxManage list vms) row n cpus mem ip port
  for row in $K3S_NODES; do
    read -r n cpus mem ip port <<< "$row"
    if [[ $existing == *"\"$n\" "* ]]; then print "$n already exists - skipped"; continue; fi
    VBoxManage clonevm kali-template --name "$n" --register || { print -u2 "WARNING: cloning $n failed - see the error above"; return 1; }
    VBoxManage modifyvm "$n" --cpus "$cpus" --memory "$mem" --groups "/k3s-ha"     # clonevm ignores --groups, so the group is set here
  done
  for row in $K3S_NODES; do
    read -r n cpus mem ip port <<< "$row"
    # replace the inherited 127.0.0.1:2222 rule (kali-template's) with this clone's own port
    VBoxManage modifyvm "$n" --natpf1 delete ssh 2>/dev/null
    VBoxManage modifyvm "$n" --natpf1 "ssh,tcp,127.0.0.1,$port,,22"
    VBoxManage startvm "$n" --type gui
  done
  for row in $K3S_NODES; do read -r n cpus mem ip port <<< "$row"; printf '%-12s ssh -p %s kali@127.0.0.1\n' "$n" "$port"; done
}
```

> The block:
> - refuses to clone while `kali-template` is still running;
> - stops at the first failed clone;
> - skips clones that already exist, so you can safely run it again.
>
> Run it while the clones are powered off: a running VM can't change its port forward, and VirtualBox then reports that the machine is locked.
>
> **A VM you made yourself with one of these names** (for example an Ubuntu `k3s-lb` created in the VirtualBox app) is skipped as well. Step 6 then fails for it, because it has no `set-identity`. Delete or rename it in VirtualBox first: `VBoxManage list vms` shows every VM.

**Success looks like:** 7 new VM windows, and the block ends with 7 lines such as `k3s-lb       ssh -p 22010 kali@127.0.0.1`.

**If it printed a WARNING instead, nothing was cloned, so don't go on to Step 6.** The only VM running is then still `kali-template`, and a Step 6 line typed there would turn the template into that node. For *kali-template is not powered off*, run this in the same window, then run the block again:

```zsh
ssh kali-template "sudo poweroff"     # "Connection ... closed by remote host" is expected
until VBoxManage showvminfo kali-template --machinereadable | grep -q '^VMState="poweroff"'; do sleep 2; done
print "kali-template is powered off"
```

**Port forwards.** Each clone inherits the template's rule `127.0.0.1:2222 → 22`, and only one VM at a time can hold a host port. So the block swaps it for the clone's own port, the last column of `K3S_NODES`. The ports follow the IP plan: `22000` + the last number of the IP, so `k3s-lb` (`.10`) answers on `127.0.0.1:22010` and `k3s-agent-3` (`.33`) on `22033`. The template keeps `2222` for itself.

| VM | NAT port (`ssh -p <port> kali@127.0.0.1`) |
|---|---|
| `kali-template` | `2222` |
| `k3s-lb` / `k3s-db` | `22010` / `22011` |
| `k3s-srv-1` / `k3s-srv-2` / `k3s-srv-3` (App. A) | `22021` / `22022` / `22023` |
| `k3s-agent-1` / `-2` / `-3` | `22031` / `22032` / `22033` |
| `k3s-lb-1` / `k3s-lb-2` (App. B) | `22008` / `22009` |
| `kali` (Kali desktop, App. E) | `22050` |

The ports go through each VM's NAT NIC (`eth0`). So they work before Step 6 gives the clones their host-only IPs, and later whenever the host-only network is broken (Appendix C). Every later step that runs inside a clone starts with that clone's `ssh -p <port> kali@127.0.0.1` line.

`clonevm` makes full clones with **new MAC addresses**, copies the template's EFI settings (`.nvram`), and groups all 7 VMs under **k3s-ha** in the VirtualBox window.

> Every clone opens in its own window with the template's **Xfce desktop**, Guest Additions and specification (2 vCPU, 2 GB, 16 MB video, 25 GB).
>
> `kali` is **not** cloned here. [Appendix E](#appendix-e--kali-linux-desktop-gui-workstation-vm) installs it as the admin workstation, from the same ISO, with the same desktop and specification. It joins the **k3s-ha** group as well.

---

## Step 6 — Console or Mac: give each clone its identity

Every clone currently believes it is `kali-template`. Each one needs one `sudo set-identity <name> <ip>` line, and the VM then reboots by itself.

**First, check that Step 5 really cloned.** **Run on:** Mac:

```zsh
() {
  local running=$(VBoxManage list runningvms) row n cpus mem ip port
  for row in $K3S_NODES; do
    read -r n cpus mem ip port <<< "$row"
    if [[ $running == *"\"$n\" "* ]]; then printf '%-12s running\n' $n
    else printf '%-12s NOT RUNNING - run Step 5 again (no clone, or not started)\n' $n; fi
  done
  if [[ $running == *'"kali-template" '* ]]; then print -u2 "WARNING: kali-template is running. Power it off first (ssh kali-template 'sudo poweroff') and never type a Step 6 line in its window"; fi
}
```

All 7 lines must say `running`, with no warning. Then use **one** of the two options:

- **Option A:** type the line in each VM window.
- **Option B:** run one loop on the Mac, through the clones' port forwards from Step 5. Nothing to type in the VM windows.

**Option B — Run on:** Mac. Wait until all 7 VM windows show a login prompt:

```zsh
# each clone through its own port forward 127.0.0.1:<port> (Step 5), so each line reaches the right VM
for row in $K3S_NODES; do
  read -r n cpus mem ip port <<< "$row"
  print "== $n   127.0.0.1:$port"
  ssh -n -p $port -o HostKeyAlias=kali-template -o BatchMode=yes -o ConnectTimeout=10 kali@127.0.0.1 "sudo set-identity $n $ip"
done
```

Each VM prints `>>> identity set: <name> / <ip> - rebooting`, followed by *Connection to 127.0.0.1 closed by remote host*. That message is expected, because the VM reboots.

**A VM without a `>>> identity set` line changed nothing.** It was usually still booting. Run the loop again once every window shows a login prompt.

Running the loop twice is safe. A VM that already has its identity is refused with *REMOTE HOST IDENTIFICATION HAS CHANGED*, because its new host key no longer matches `kali-template`.
- **`HostKeyAlias=kali-template`:** until `set-identity` runs, every clone still has the template's SSH host key, which part 2 of Step 4 saved under that name. So a clone that answers proves it is still fresh.
- **`BatchMode=yes`:** never asks for a password. **`-n`:** the `ssh` reads nothing from the Terminal.
- ***REMOTE HOST IDENTIFICATION HAS CHANGED*** for one VM: that clone already has its own identity, and nothing was changed on it.
- ***Permission denied (publickey,…)***: your SSH key isn't on the template (Step 4 part 4, *Mac SSH key authorized*). Use option A.
- ***Connection reset***, *kex_exchange_identification* or a timeout: the VM is still booting, or its port forward is missing (`VBoxManage showvminfo VMNAME --machinereadable | grep '^Forwarding'`).

**Option A — In each VM window**, log in as `kali` at the graphical login, open **Applications → Terminal Emulator**, and type the matching line. You can also paste it with **Ctrl+Shift+V**, because the clones share the clipboard (Step 4 part 3b).
- **Or from the Mac:** connect with the `ssh -p` line from the same row and paste the line there. The port decides which VM answers, so always take both from the same row.
- The session then closes with *Connection to 127.0.0.1 closed by remote host*, because the VM reboots.

> **Know which VM you are in.**
> - The prompt says `kali@kali-template` in **every** clone, and in the template itself. Go by the VirtualBox **window title** instead (for example *k3s-srv-1 [Running]*).
> - A window titled *kali-template* is the template: never type these lines there. `set-identity` refuses there anyway, with *this is kali-template itself*.
>
> **Typos are caught.**
> - `set-identity` checks the name/IP pair against the `home.lab` block from Step 4. A wrong pair such as `k3s-srv-1 192.168.99.22` stops with the IP the plan expects, before anything is changed.
> - The check on the Mac below also catches a correct line typed into the wrong window.

| VM window | Or connect from the Mac | Type or paste this |
|---|---|---|
| `k3s-lb` | `ssh -p 22010 kali@127.0.0.1` | `sudo set-identity k3s-lb 192.168.99.10` |
| `k3s-db` | `ssh -p 22011 kali@127.0.0.1` | `sudo set-identity k3s-db 192.168.99.11` |
| `k3s-srv-1` | `ssh -p 22021 kali@127.0.0.1` | `sudo set-identity k3s-srv-1 192.168.99.21` |
| `k3s-srv-2` | `ssh -p 22022 kali@127.0.0.1` | `sudo set-identity k3s-srv-2 192.168.99.22` |
| `k3s-agent-1` | `ssh -p 22031 kali@127.0.0.1` | `sudo set-identity k3s-agent-1 192.168.99.31` |
| `k3s-agent-2` | `ssh -p 22032 kali@127.0.0.1` | `sudo set-identity k3s-agent-2 192.168.99.32` |
| `k3s-agent-3` | `ssh -p 22033 kali@127.0.0.1` | `sudo set-identity k3s-agent-3 192.168.99.33` |

You can then close the VM windows and choose **"Continue running in the background"**.

**Run on:** Mac. This block:
- adds SSH shortcuts (only once);
- forgets the old host keys for these IPs and port forwards, because `set-identity` gave every clone new ones;
- checks reachability.

```zsh
grep -qx 'Host k3s-lb' ~/.ssh/config || cat >> ~/.ssh/config <<'EOF'

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
# Kali desktop (GUI) workstation, installed in Appendix E
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
EOF
() {
  local row n cpus mem ip port mac out got res lines words
  # new identity = new host keys. Keys saved earlier (an earlier build, or ssh -p <port> before set-identity) would stop ssh
  # with "REMOTE HOST IDENTIFICATION HAS CHANGED": forget them for the host-only IP and for the port forward 127.0.0.1:<port>
  for row in $K3S_NODES; do read -r n cpus mem ip port <<< "$row"; ssh-keygen -R $ip >/dev/null 2>&1; ssh-keygen -R "[127.0.0.1]:$port" >/dev/null 2>&1; done

  # For each VM in K3S_NODES: whatever answers at its planned IP must report the planned FQDN, exactly that IP
  # on eth1, and the MAC of THIS VirtualBox VM's NIC 2 (that proves the line went into the right window)
  for row in $K3S_NODES; do
    read -r n cpus mem ip port <<< "$row"
    mac=$(VBoxManage showvminfo $n --machinereadable | sed -n 's/^macaddress2="\(.*\)"/\1/p' | sed 's/../&:/g; s/:$//' | tr 'A-F' 'a-f')   # 080027AABBCC -> 08:00:27:aa:bb:cc
    out=$(ssh -n -o ConnectTimeout=5 $n 'hostname -f; cat /sys/class/net/eth1/address; ip -4 -br addr show eth1' 2>/dev/null)
    lines=(${(f)out}); words=(${=lines[3]}); got="${words[3,-1]}"
    if   (( ${#lines} == 0 )); then res='NO ANSWER'
    elif [[ $lines[1] == "$n.home.lab" && $lines[2] == "$mac" && $got == "$ip/24" ]]; then res=OK
    else res="MISMATCH  got: ${(j: | :)lines}"; fi
    printf '%-12s %-14s %s\n' $n $ip "$res"
  done
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
- **`NO ANSWER`**: nothing answers at that IP yet. The usual causes:
  - **`set-identity` hasn't run on that VM**, so its `eth1` still has the template's setting.
  - **The VM is still rebooting**, or NetworkManager ignored the keyfile.
  - **macOS is blocking your terminal app from the local network.** Then every line says `NO ANSWER`: see the *No route to host* row in Appendix C.

  Ask each VM through its NAT port forward, which works without `eth1`:

  ```zsh
  for row in $K3S_NODES; do read -r n cpus mem ip port <<< "$row"; printf '%-12s %s\n' $n "$(ssh -n -p $port -o BatchMode=yes -o ConnectTimeout=5 -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null -o LogLevel=ERROR kali@127.0.0.1 'hostname; ip -4 -br addr show eth1' | paste -sd'|' -)"; done
  ```

  - **The second column must start with the VM's own name.** `kali-template`, or another VM's name, means `set-identity` never ran there. Run option B again, or type that VM's line in its window.
  - **The right name with no IP on `eth1`:** see the Appendix C row *A VM has no `192.168.99.x`*.
  - **Then run the whole check block again**, including its `ssh-keygen -R` line, because the new identity comes with a new host key.

Optional but highly recommended: take a snapshot so you can roll back any later step.

```zsh
for row in $K3S_NODES; do VBoxManage snapshot ${row%% *} take "01-identity" --live; done
```

---

## Step 7 — k3s-db: MariaDB datastore and secrets

**Run on:** `k3s-db`. Connect from the Mac, then paste the `bash` blocks below into that session:

```zsh
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

**Run on:** Mac. Hand the secrets to the servers (full file) and the agents (token only):

```zsh
T=$(mktemp -d)       # a private folder in your macOS temp directory
scp k3s-db:k3s-secrets.env "$T/k3s-secrets.env"
scp k3s-db:k3s-agent.env   "$T/k3s-agent.env"
for h in k3s-srv-1 k3s-srv-2;               do scp "$T/k3s-secrets.env" "$h:k3s-secrets.env"; done
for h in k3s-agent-1 k3s-agent-2 k3s-agent-3; do scp "$T/k3s-agent.env"   "$h:k3s-agent.env"; done
rm -rf "$T"
```

---

## Step 8 — k3s-lb: NGINX TCP load balancer

This VM is the single entry point. It forwards **:6443** to the K3s servers, and **:80/:443** to Traefik on the agents.

**Run on:** `k3s-lb`. Connect from the Mac, then paste the block below:

```zsh
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

**Run on:** `k3s-srv-1`. Connect from the Mac, then paste the block below:

```zsh
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

> **Apple Silicon:** the K3s installer detects `arm64` and installs the arm64 build by itself. All the images K3s starts (CoreDNS, Traefik, metrics-server, local-path-provisioner) are multi-arch.

---

## Step 10 — k3s-srv-2: second K3s server

**Run on:** `k3s-srv-2`. These are **exactly the same commands** as Step 9; `NODE_IP` detects `.22` by itself. Connect from the Mac, then paste the block below:

```zsh
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

**Run on:** `k3s-db`. Confirm that the cluster state now lives in MariaDB. Connect from the Mac, then run the check:

```zsh
ssh -p 22011 kali@127.0.0.1      # k3s-db   (or: ssh k3s-db)
```

```bash
sudo mariadb -e "SELECT COUNT(*) AS kine_rows FROM k3s.kine;"
```

**Run on:** `k3s-lb`. Confirm that the API answers through the load balancer (HTTP `401` is the expected answer here). Connect from the Mac, then run the check:

```zsh
ssh -p 22010 kali@127.0.0.1      # k3s-lb   (or: ssh k3s-lb)
```

```bash
curl -sk -o /dev/null -w '%{http_code}\n' https://127.0.0.1:6443/version
```

---

## Step 11 — k3s-agent-1/2/3: join the workers

**Run on:** `k3s-agent-1`, then `k3s-agent-2`, then `k3s-agent-3`: the identical block on each. Connect to one agent from the Mac, paste the block, type `exit`, then connect to the next one:

| VM | Connect from the Mac |
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

**Run on:** `k3s-srv-1`. Give the workers a role label and check the whole cluster. Connect from the Mac, then paste the block below:

```zsh
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

Still on `k3s-srv-1`, prepare a kubeconfig copy for the Mac:

```bash
sudo cp /etc/rancher/k3s/k3s.yaml ~/k3s.yaml && sudo chown kali: ~/k3s.yaml
```

---


## Step 12 — Mac: kubectl through the load balancer

**Run on:** Mac.

```zsh
mkdir -p ~/.kube && chmod 700 ~/.kube
[[ -f ~/.kube/config ]] && cp ~/.kube/config ~/.kube/config.bak

scp k3s-srv-1:k3s.yaml ~/.kube/config
ssh k3s-srv-1 "rm -f ~/k3s.yaml"

# point kubectl at the LOAD BALANCER, not at a single server  (macOS sed: -i '' edits in place)
sed -i '' 's/127\.0\.0\.1/192.168.99.10/' ~/.kube/config
chmod 600 ~/.kube/config
kubectl config rename-context default k3s-ha

kubectl get nodes -o wide
kubectl cluster-info          # Kubernetes control plane is running at https://192.168.99.10:6443
```

> Was there an older `config.bak` (e.g. from Docker Desktop or OrbStack)? You can merge it later with `KUBECONFIG=~/.kube/config:~/.kube/config.bak kubectl config view --flatten`.

---

## Step 13 — Mac: test HA failover

**Run on:** Mac.

```zsh
# 1) kill server #1 hard
VBoxManage controlvm k3s-srv-1 poweroff
kubectl get nodes                 # still answers (NGINX -> k3s-srv-2); srv-1 turns NotReady after ~40 s

# 2) bring it back, then kill server #2
VBoxManage startvm k3s-srv-1 --type gui
kubectl get nodes -w              # wait for k3s-srv-1 Ready, Ctrl+C
VBoxManage controlvm k3s-srv-2 poweroff
kubectl get nodes                 # still answers via k3s-srv-1
VBoxManage startvm k3s-srv-2 --type gui
```

> **Single points of failure in the video's design:** `k3s-db` and `k3s-lb`. If either one is down, the API is down (running pods keep running). Appendix A removes the DB from the picture (embedded etcd); Appendix B makes the LB redundant (keepalived VIP).

Snapshot point:

```zsh
for row in $K3S_NODES; do VBoxManage snapshot ${row%% *} take "02-k3s-ha" --live; done
```

---

## Step 14 — Mac: install Argo CD

**Run on:** Mac.

```zsh
kubectl create namespace argocd
kubectl apply -n argocd --server-side --force-conflicts -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

kubectl -n argocd wait --for=condition=Ready pod --all --timeout=600s
kubectl -n argocd get pods -o wide        # all pods land on k3s-agent-* (the servers are tainted)
```

- `--server-side --force-conflicts` is the currently documented way. Some Argo CD CRDs are too large for client-side apply.
- **To pin a version** (recommended for anything long-lived), replace `stable` in the URL with a release tag from <https://github.com/argoproj/argo-cd/releases>.
- **To run Argo CD in HA mode**, use `.../manifests/ha/install.yaml` instead. It needs **3 schedulable nodes**, which the three agents provide.
- Every Argo CD image (Argo CD, Redis, Dex) has an `arm64` build, so the same manifest works on Apple Silicon.

Quick smoke test before exposing it. Open <https://localhost:8080> (self-signed certificate warning), then press Ctrl+C:

```zsh
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

---

## Step 15 — Mac: expose Argo CD through Traefik

K3s ships **Traefik** as its ingress controller. The path is browser → `k3s-lb:443` → Traefik on an agent → `argocd-server`. Traefik terminates TLS, so `argocd-server` must serve plain HTTP behind it (`server.insecure`). Without that you get a redirect loop.

**Run on:** Mac.

```zsh
# 1) argocd-server: plain HTTP behind the ingress
kubectl -n argocd patch configmap argocd-cmd-params-cm --type merge -p '{"data":{"server.insecure":"true"}}'
kubectl -n argocd rollout restart deployment argocd-server
kubectl -n argocd rollout status deployment argocd-server

# 2) Ingress: https://argocd.home.lab  (Traefik's built-in self-signed cert)
kubectl apply -f - <<'EOF'
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
EOF
kubectl -n argocd get ingress
```

**Run on:** Mac. Add the DNS names to the Mac's `/etc/hosts` (`sudo` asks for your Mac password), then flush the macOS DNS cache:

```zsh
grep -q 'argocd\.home\.lab' /etc/hosts || printf '\n192.168.99.10  k3s.home.lab argocd.home.lab whoami.home.lab\n' | sudo tee -a /etc/hosts >/dev/null
sudo dscacheutil -flushcache; sudo killall -HUP mDNSResponder
```

Test it:

```zsh
curl -skI https://argocd.home.lab | head -1      # HTTP/2 200  (or HTTP/1.1 200 OK)
```

Open **<https://argocd.home.lab>** and accept the self-signed certificate.
- **Safari:** *Show Details → visit this website*.
- **Chrome:** *Advanced → Proceed*.

Use `https://`, because the route is attached only to the `websecure` entrypoint.

> For a trusted certificate later, add **cert-manager** with a Let's Encrypt DNS-01 issuer, or your own CA, and put a `tls:` block on the Ingress.

---

## Step 16 — Mac: Argo CD CLI login and admin password

**Run on:** Mac.

```zsh
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

## Step 17 — Mac: first Argo CD app (podinfo)

This is a zero-setup sanity check from a public Git repo.

> **Why not the Windows guide's guestbook?** Its image, `gcr.io/google-samples/gb-frontend:v5`, is built for **amd64 only**. On Apple Silicon (arm64) nodes its pods crash with `exec format error`.
> - **podinfo** is a small demo web app published for amd64 **and** arm64, with Kustomize manifests in its repo.
> - On an Intel Mac the guestbook works too: `repoURL: https://github.com/argoproj/argocd-example-apps.git`, `path: guestbook`.

**Run on:** Mac.

```zsh
kubectl apply -f - <<'EOF'
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: podinfo
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/stefanprodan/podinfo.git
    targetRevision: HEAD
    path: kustomize
  destination:
    server: https://kubernetes.default.svc
    namespace: podinfo
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
EOF

argocd app wait podinfo --health --timeout 300
argocd app get podinfo                    # Sync Status: Synced, Health: Healthy
kubectl -n podinfo get pods -o wide       # 2 pods on the agents (its HPA scales 2-4)
```

To see it, run this and open <http://localhost:9898> (it shows the pod name), then press Ctrl+C:

```zsh
kubectl -n podinfo port-forward svc/podinfo 9898:9898
```

Clean up when you're done: `argocd app delete podinfo --yes`.

---

## Step 18 — Mac and GitHub: your own GitOps repo (app-of-apps)

From here on, **Git is the source of truth**. You push YAML, and Argo CD makes the cluster match it.

```mermaid
sequenceDiagram
    participant You as You (Mac)
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

**Run on:** Mac. First create an **empty public** repo named `homelab-gitops` on github.com (no README), then:

> **Already have the repo cloned?** For example `~/Desktop/Working/homelab-gitops`, which holds this guide. The block then refuses to write anything (it checks for `.git`), so your files stay as they are. Continue with the bootstrap block below.

```zsh
GH_USER="Saveit29"                                  # <-- change me
REPO="$HOME/Desktop/Working/homelab-gitops"         # next to Tools; any folder works
() {
if [[ -d $REPO/.git ]]; then print -u2 "$REPO is already a git repository - nothing written. Continue with the bootstrap block"; return 1; fi
mkdir -p "$REPO/apps/whoami" "$REPO/argocd/apps" && cd "$REPO" || return 1

# --- the app itself --------------------------------------------------------
cat > apps/whoami/whoami.yaml <<'EOF'
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
          image: traefik/whoami        # pin a version tag for real use (multi-arch: amd64 + arm64)
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
EOF

# --- Argo CD Application for whoami ($GH_USER is filled in) ------------------
cat > argocd/apps/whoami.yaml <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: whoami
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/$GH_USER/homelab-gitops.git
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
EOF

# --- root app: watches argocd/apps and creates every Application in it ----
cat > argocd/root.yaml <<EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: root
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/$GH_USER/homelab-gitops.git
    targetRevision: main
    path: argocd/apps
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
EOF

# --- push --------------------------------------------------------------------
# (first time only) git config --global user.name "Your Name"; git config --global user.email "you@example.com"
git init -b main
git add .
git commit -m "GitOps: whoami app + app-of-apps root"
git remote add origin "https://github.com/$GH_USER/homelab-gitops.git"
git push -u origin main
}
```

> `git push` over HTTPS asks for your GitHub user name and a **personal access token** (not your password). Apple's git saves it in the macOS Keychain, so it asks only once.

Bootstrap: this is the **only** `kubectl apply` you ever do for apps.

```zsh
cd "$REPO"                                          # REPO from the block above (or: cd ~/Desktop/Working/homelab-gitops)
kubectl apply -f argocd/root.yaml
argocd app list                                     # root + whoami -> Synced / Healthy
kubectl -n whoami get pods -o wide                  # the pods spread over the agents

# load-balanced across pods/agents: Hostname changes between calls
for i in {1..6}; do curl -sk https://whoami.home.lab/whoami | grep '^Hostname'; done
```

**See GitOps work:**

```zsh
# a) change in Git -> cluster follows
sed -i '' 's/replicas: 3/replicas: 5/' apps/whoami/whoami.yaml
git commit -am "scale whoami to 5"
git push
argocd app get whoami --refresh                     # or click REFRESH in the UI
kubectl -n whoami get pods

# b) manual drift -> Argo CD reverts it (selfHeal)
kubectl -n whoami scale deployment whoami --replicas=1
kubectl -n whoami get deployment whoami -w          # jumps back to 5 within seconds, Ctrl+C
```

> In a repo that already runs a different replica count (this guide's own repo runs 5), change the number that `grep replicas apps/whoami/whoami.yaml` shows instead.

**Adding the next app:**
1. Put its manifests in `apps/<name>/`.
2. Add `argocd/apps/<name>.yaml`: copy `whoami.yaml` and change `name`, `path` and `namespace`.
3. Push.

Every image it uses needs an `arm64` build on Apple Silicon.

**Private repo?** Register it once (replace `YOUR-GITHUB-TOKEN`):

```zsh
argocd repo add "https://github.com/$GH_USER/homelab-gitops.git" --username "$GH_USER" --password 'YOUR-GITHUB-TOKEN'
```

---

## Step 19 — Day-2: start/stop, backups, upgrades

### Start and stop the lab in the right order

**Run on:** Mac.

```zsh
# START: datastore -> LB -> servers -> agents, each in its own window with its desktop (--type headless = no window)
for vm in k3s-db k3s-lb k3s-srv-1 k3s-srv-2 k3s-agent-1 k3s-agent-2 k3s-agent-3; do
  VBoxManage startvm $vm --type gui
  sleep 15
done
VBoxManage startvm kali --type gui           # Kali Xfce desktop (Appendix E): opens in its own window

# STOP (clean shutdown over SSH, reverse order; the power button if SSH doesn't answer)
ssh -n -o ConnectTimeout=5 kali "sudo poweroff"
for vm in k3s-agent-3 k3s-agent-2 k3s-agent-1 k3s-srv-2 k3s-srv-1 k3s-lb k3s-db; do
  ssh -n -o ConnectTimeout=5 $vm "sudo poweroff" || VBoxManage controlvm $vm acpipowerbutton
done
```

> The VMs pause while the Mac sleeps. After a long sleep their clocks can lag behind: see *Errors after the Mac slept* in Appendix C. For a long break, stop the lab first.

### Backups

**Run on:** `k3s-db`. Nightly dump of the cluster state, keeping 7 days. Connect from the Mac, then paste the block below:

```zsh
ssh -p 22011 kali@127.0.0.1      # k3s-db   (or: ssh k3s-db)
```

```bash
sudo mkdir -p /var/backups/k3s
echo '0 3 * * * root mariadb-dump --single-transaction k3s | gzip > /var/backups/k3s/k3s-$(date +\%F).sql.gz && find /var/backups/k3s -name "*.sql.gz" -mtime +7 -delete' | sudo tee /etc/cron.d/k3s-db-backup
```

Also keep:
- `~/k3s-secrets.env`, because the **token is required** to restore;
- VirtualBox snapshots (`VBoxManage snapshot VMNAME take SNAPSHOTNAME --live`);
- your **Git repo**. Argo CD's apps are already backed up there, and `argocd admin export -n argocd > argocd-backup.yaml` exports Argo CD's own settings.

### Upgrade K3s (one node at a time)

`config.yaml` holds all the flags, so an upgrade is just re-running the installer with a new version.

```bash
# on k3s-srv-1 (ssh -p 22021 kali@127.0.0.1), wait until Ready, then on k3s-srv-2 (ssh -p 22022 kali@127.0.0.1):
curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION="v1.xx.y+k3s1" sh -s - server
```

```zsh
# Mac, for each agent:
kubectl drain k3s-agent-1 --ignore-daemonsets --delete-emptydir-data
```

```bash
# on that agent: ssh -p 22031 kali@127.0.0.1 (k3s-agent-1), port 22032 (k3s-agent-2), port 22033 (k3s-agent-3)
curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION="v1.xx.y+k3s1" sh -s - agent
```

```zsh
kubectl uncordon k3s-agent-1     # repeat drain/upgrade/uncordon for agent-2 and agent-3
```

### Upgrade Argo CD

```zsh
ARGOCD_VERSION=vX.Y.Z      # <-- a release tag from https://github.com/argoproj/argo-cd/releases
kubectl apply -n argocd --server-side --force-conflicts -f "https://raw.githubusercontent.com/argoproj/argo-cd/$ARGOCD_VERSION/manifests/install.yaml"
```

The `server.insecure` patch from Step 15 survives this, because it is owned by a different field manager. Update the CLI on the Mac with Step 1.3.

---

## Appendix A — Variant: embedded etcd (no external DB)

This removes `k3s-db` as a single point of failure. It needs an **odd number of servers, at least 3** (3 servers tolerate 1 failure, 5 tolerate 2).

Changes to the plan:
- Skip `k3s-db`: remove its row from `K3S_NODES` in block 1.1.
- Add `k3s-srv-3` at `192.168.99.23`. Add the row `"k3s-srv-3    2  2048  192.168.99.23  22023"` to `K3S_NODES` in block 1.1, run 1.1 again, then Step 5.
- Run `sudo set-identity k3s-srv-3 192.168.99.23` in its window (or Step 6 option B). The name is already in the `home.lab` block and the SSH config.
- Add `server 192.168.99.23:6443 ...;` to the `k3s_api` upstream on `k3s-lb`.

**Run on:** `k3s-srv-1` (`K3S_TOKEN` from `openssl rand -hex 32`). Connect from the Mac, then paste the block below:

```zsh
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
1. In `K3S_NODES` (block 1.1), replace the `k3s-lb` row with two rows, `"k3s-lb-1     2  2048  192.168.99.8   22008"` and `"k3s-lb-2     2  2048  192.168.99.9   22009"` (the same specification as every VM). Run 1.1 again, then Step 5 to clone them.
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

To test, power off `k3s-lb-1` (`VBoxManage controlvm k3s-lb-1 poweroff`). The VIP moves to `k3s-lb-2`, and `kubectl get nodes` keeps working.

---

## Appendix C — Troubleshooting

### The cluster (inside the VMs)

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
| `sudo: set-identity: command not found` on a clone | Kali's `sudo` doesn't search `/usr/local/sbin` (Step 4 part 3 item 1 adds it), the template was cloned before part 3 finished, or a `sudo` password prompt swallowed part of the paste | On that clone, `sudo /usr/local/sbin/set-identity <name> <ip>` works if the file exists. Otherwise follow *Already cloned…* at the end of Step 4: delete the clones, fix the template, run the part 4 check, clone again |
| `set-identity <name> <ip>` only prints its usage line, or `[[: not found` | `#!/usr/bin/env bash` is not the script's first line (an empty line from the paste), so `sh` runs it | On the template, or on each clone before its `set-identity`: `sudo sed -i -e 's/\r$//' -e '/./,$!d' /usr/local/sbin/set-identity`, then run the line again |
| `set-identity` says *unknown VM name* or *wrong IP for …* | a typo in the name or the IP, or the `home.lab` block is missing | Type the line exactly as in the Step 6 table. `grep -A14 '^# BEGIN home.lab' /etc/hosts` must show the block (Step 4 item 6) |
| Two VMs answer on the same IP / *REMOTE HOST IDENTIFICATION HAS CHANGED* | `set-identity` skipped, or an IP was reused | Run `set-identity` on the clone; `ssh-keygen -R 192.168.99.x` |
| NICs have `enp0s…` names instead of `eth0` / `eth1` | `net.ifnames=0` is missing from the kernel command line | `cat /proc/cmdline`; redo the GRUB lines of Step 4 part 2, then reboot |
| Pods fail to start with an `apparmor` error | AppArmor userspace tools missing | `sudo apt install -y apparmor`, then `sudo systemctl restart k3s` (or `k3s-agent`) |
| Pods stay in `CrashLoopBackOff`, and `kubectl logs` shows `exec format error` | The image has no `arm64` build, and the Apple Silicon nodes are arm64 (for example the guestbook image `gcr.io/google-samples/gb-frontend:v5`) | Use an image published for `linux/arm64` (Docker Hub and GHCR list the platforms of each tag). Step 17 uses podinfo for this reason |

### The Mac, VirtualBox and the VM windows

| Symptom | Likely cause | Fix |
|---|---|---|
| `zsh: command not found: #` for every comment line of a pasted block | `interactivecomments` is off in this window: Step 1.1 has not run yet, or the window was opened before it | Type `setopt interactivecomments`, then paste again. Run 1.1 once, so every new window gets it from `~/.zshrc` |
| `zsh: no matches found: …` | zsh read `?`, `*` or `[...]` in an argument as a file pattern | Keep the quotes from the guide, e.g. `ssh-keygen -R "[127.0.0.1]:2222"`. For your own commands, quote URLs and patterns |
| `zsh: parse error near …` or *no such file or directory* after you filled in a value | a `<placeholder>` was left in a command: `<` and `>` are redirections in zsh | Replace the whole placeholder, brackets included |
| `VBoxManage: command not found` | VirtualBox is not installed, or `/usr/local/bin` is missing from PATH | Run Step 1.2; `ls -l /usr/local/bin/VBoxManage` must point into `/Applications/VirtualBox.app`, and `cat /etc/paths` must list `/usr/local/bin` |
| Step 1.7 shows `WARN` for `kubectl`, or `kubectl version --client` prints an older version than Step 1.3 downloaded | Docker Desktop (`/usr/local/bin`) or Homebrew (`/opt/homebrew/bin`) is found first | Run block 1.1 again, then open a new window. `tail -3 ~/.zshrc` must end with the `k3s-argocd-setup` line, and `which -a kubectl` must list `…/Tools/bin/kubectl` first |
| *"kubectl" (or "argocd") cannot be opened because the developer cannot be verified* | The file was downloaded with a browser, so macOS quarantined it | `xattr -d com.apple.quarantine ~/Desktop/Working/Tools/bin/kubectl` (same for `argocd`), or download with block 1.3, which uses `curl` |
| **Intel Mac:** *Kernel driver not installed (rc=-1908)*, or no VM starts after installing VirtualBox | macOS blocked VirtualBox's kernel extension | *System Settings → Privacy & Security* → **Allow** next to *Oracle America, Inc.*, then restart the Mac. Apple Silicon has no kernel extension |
| *Nonexistent host networking interface, name 'k3s-hostonly'*, or a VM won't start because of NIC 2 | The host-only network was removed (for example in *VirtualBox → Tools → Network*), or Step 2 never ran | Power off every VM, then run Step 2 again |
| From the Mac, `ssh`, `curl` or the browser to `192.168.99.x` fails with ***No route to host***, while `ssh -p 22011 kali@127.0.0.1` works | macOS **Local Network** privacy blocks that app. This is common for iTerm2, VS Code, Warp and third-party browsers; Apple's Terminal and Safari are usually fine | *System Settings → Privacy & Security → Local Network*: switch the app on, then quit and reopen it |
| `192.168.99.x` times out from the Mac although the VMs run | A VPN (or another VM app) routes `192.168.99.0/24`, or no VM on `k3s-hostonly` is running, so `bridge10x` doesn't exist | `route -n get 192.168.99.10` must show `interface: bridge100` (or another `bridge10x`), and `ifconfig \| grep 'inet 192.168.99.100'` must find the Mac's address. Disconnect the VPN, or move the lab to another network (0.5) |
| The template (or a clone) shows the Kali **installer** again after the install, or stops at the UEFI `Shell>` prompt | Apple Silicon VMs boot through EFI: the ISO was still found first, or the VM's EFI has no boot entry for the disk | Power the VM off, then remove the ISO: `VBoxManage storageattach VMNAME --storagectl VirtioSCSI --port 1 --device 0 --medium emptydrive`, and start it again. At `Shell>`: type `FS0:`, then `\EFI\kali\grubaa64.efi` (`ls \EFI\kali` lists the files). Once Kali runs, the Step 4 part 2 EFI lines make `EFI/BOOT/BOOTAA64.EFI` the permanent fallback |
| The keyboard does nothing in a VM window | An ARM VM created without `--default` gets a PS/2 keyboard, which ARM doesn't have | Power the VM off, then `VBoxManage modifyvm VMNAME --keyboard usb --mouse usbtablet --usb-xhci on` |
| `Clone VM failed` / `Failed to lock source media '…kali-template.vdi'` (`VBOX_E_INVALID_OBJECT_STATE`) in Step 5 | `kali-template` is still running, and a running VM keeps its disk locked | After the Step 4 part 4 check, power the template off: `ssh kali-template "sudo poweroff"` on the Mac, or `sudo poweroff` in its VM window. Wait until `VBoxManage list runningvms` no longer lists it, then run Step 5 again |
| Step 5 prints *WARNING: kali-template is not powered off (VMState="running")* | Same cause. The block stopped before cloning, so **no clone exists yet** | Same fix, using the power-off lines under the Step 5 block. Don't start Step 6 before the block prints its 7 `ssh -p` lines |
| **The Mac restarted** right after a paste, or the Mac says `sudo: poweroff: command not found` | A VM block was pasted into the **Mac's** own Terminal. macOS has `reboot` (and `halt`), but no `poweroff` | Check the prompt before every paste: `kali@…` is a VM, `you@YourMac … %` is the Mac. Power a VM off from the Mac with `ssh VMNAME "sudo poweroff"` |
| `ssh kali-template` or `ssh -p 2222 kali@127.0.0.1`: *Connection reset*, *kex_exchange_identification: Connection closed by remote host*, or a timeout | the VM is not running or still booting, its SSH server is off (Step 4 part 1), the port forward is missing, or `eth0` has no address | `VBoxManage showvminfo kali-template --machinereadable \| grep '^Forwarding'` must print `ssh,tcp,127.0.0.1,2222,,22` (add it as in the Step 3 note); in the VM window, `systemctl is-active ssh` must say `active` and `ip -br a` must show `10.0.2.15` on `eth0` (otherwise see the `eth0` note in Step 4 part 2) |
| *WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED … Host key verification failed* for `kali-template` or `[127.0.0.1]:2222` | The template was reinstalled, so it has new SSH host keys, but `~/.ssh/known_hosts` still holds the old ones | Run Step 4 part 2 again, which removes them first. By hand: `ssh-keygen -R kali-template; ssh-keygen -R "[127.0.0.1]:2222"`, then connect again. In Step 6 option B, the same warning only means that clone already ran `set-identity`. **Template not reinstalled?** Then `set-identity` probably ran on it: see *The template got a clone's identity* below |
| The `kali-template` window or `hostname` shows a node name (e.g. `k3s-lb`), and Step 5 has created no clones | A Step 6 `set-identity` line ran in the template, usually because Step 5 had stopped with its warning and the template was the only VM running | See *The template got a clone's identity* below |
| A clone's port forward doesn't answer, or `VBoxManage startvm` reports *Locked* / port forward errors in Step 5 | the clone was running while Step 5 changed its rule, or another program uses that host port | Power the clone off, then `VBoxManage modifyvm VMNAME --natpf1 delete ssh; VBoxManage modifyvm VMNAME --natpf1 "ssh,tcp,127.0.0.1,PORT,,22"`. `lsof -nP -iTCP:PORT -sTCP:LISTEN` shows which program holds a port |
| A VM has no `192.168.99.x` after `set-identity` | NetworkManager ignored the keyfile, or NIC 2 is not attached | Get in with copy and paste through its NAT port forward: `ssh -p PORT kali@127.0.0.1` (Step 5 table, e.g. `22021` for `k3s-srv-1`). Then `nmcli connection show`; the files in `/etc/NetworkManager/system-connections/` must be owned by `root` with mode `600`; `sudo nmcli connection up hostonly`. On the Mac, `VBoxManage showvminfo VMNAME --machinereadable \| grep -E '^(nic2\|hostonly-network2)='` must print `nic2="hostonlynetwork"` and `hostonly-network2="k3s-hostonly"` |
| The Mac gets slow, the VMs stutter, *Activity Monitor → Memory Pressure* is yellow or red | 8 running VMs × 2 GB is all the RAM of a 16 GB Mac | Power `kali` off when you don't use it. Boot nodes whose window you don't need without their desktop: `sudo systemctl set-default multi-user.target && sudo reboot` (back with `graphical.target`). Run two agents. Less RAM per VM also works, but leaves the same-spec plan: `VBoxManage modifyvm VMNAME --memory MB` while that VM is off |
| Errors after the Mac slept (TLS *not yet valid*, DB timeouts) | The VMs were paused, and their clocks lag behind | `sudo systemctl restart systemd-timesyncd; timedatectl` on each VM, or restart the lab (Step 19) |
| `kubectl` on `kali` → *connection refused* to `127.0.0.1:6443` | the raw `k3s.yaml` was copied instead of the Step 12 kubeconfig | Re-run the `scp ~/.kube/config kali:.kube/config` line from E.4 |
| A VM (the template, a clone or `kali`) boots to a **text login**, no desktop | **Xfce** was not ticked in Software selection (ticking only the *Desktop environment* heading does nothing), or the VM was switched to `multi-user.target` (0.2) | `sudo apt update && sudo apt install -y kali-desktop-xfce` (what the Xfce entry installs), or `sudo tasksel` and tick Xfce; then `sudo systemctl set-default graphical.target && sudo reboot`. On the template, do it before Step 5, so the clones inherit it |
| A VM window doesn't resize with its desktop, or the shared clipboard doesn't work | **Apple Silicon:** the VirtualBox Guest Additions are missing (template: Step 4 part 3b, inherited by the clones; `kali`: E.3). Kali has no arm64 `virtualbox-guest-x11` package. **Intel Mac:** `virtualbox-guest-x11` is missing | Apple Silicon: run the part 3b commands on the template before Step 5, or the E.3 commands (with that VM's name) on one VM. Then tick *View → Auto-resize Guest Display* in its window. Intel Mac: `sudo apt update && sudo apt install -y --reinstall virtualbox-guest-x11 && sudo reboot`. Both: `VBoxManage showvminfo VMNAME \| grep -i clipboard` must say *Bidirectional*; to turn it on, run `VBoxManage modifyvm VMNAME --clipboard-mode bidirectional` while the VM is off |
| The Guest Additions installer (Step 4 part 3b or E.3) ends with *modprobe vboxguest failed* or *Look at /var/log/vboxadd-setup.log* | The kernel headers for the running kernel are missing, or the kernel is newer than these Guest Additions support | `sudo apt install -y linux-headers-$(uname -r)`; if that package is gone, `sudo apt full-upgrade -y && sudo reboot` first. Then run the installer again, and read `/var/log/vboxadd-setup.log`. The desktop works without them, just with a fixed size and no shared clipboard |
| A Kali desktop is tiny or blurry on a Retina display | the VM draws at 1× | In that VM's window: *View → Virtual Screen 1 → Scale to 200%* (or 150%) |

### The template got a clone's identity

**Symptoms:**
- `ssh kali-template` stops with *REMOTE HOST IDENTIFICATION HAS CHANGED*, although you didn't reinstall the template.
- The template's `hostname` is a node name such as `k3s-lb`, with that node's IP on `eth1`.

**Cause:** a Step 6 `sudo set-identity …` line ran in `kali-template`.
- Templates built with the current Step 4 item 7 refuse this with *this is kali-template itself*.
- Older ones go ahead. They give themselves the node's name, IP, machine-id and SSH host keys, and reboot.

**Fix:** you don't need to reinstall. Change the template back, then continue from the Step 4 check.

1. **Run on:** Mac. Forget the template's old host key and log in again. `accept-new` from the Step 4 shortcut saves the new key under `kali-template`, which Step 6 option B checks against:

   ```zsh
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

> **Already cloned from the changed template?** Every clone then says `k3s-lb` (or whichever name ran), and only one of them holds that IP. The others see a duplicate address and leave `eth1` empty.
> - **You can keep these clones.** Step 6 replaces everything `set-identity` changed, so run it on them as usual.
> - **Option B** works if `~/.ssh/known_hosts` holds the changed template's key under `kali-template`, which step 1 above saves. If option B says *REMOTE HOST IDENTIFICATION HAS CHANGED* for every clone, use option A.
> - **Afterwards**, do steps 1–3 on the template before you boot or clone it again (Appendix A/B). Otherwise it comes up with the node's IP and clashes with the real node.

Useful logs:

```bash
sudo journalctl -u k3s -f              # servers: ssh -p 22021 kali@127.0.0.1 (k3s-srv-1), port 22022 (k3s-srv-2)
sudo journalctl -u k3s-agent -f        # agents:  ssh -p 22031 kali@127.0.0.1 (k3s-agent-1), port 22032 / 22033 (-2 / -3)
sudo tail -f /var/log/nginx/k3s-stream.log /var/log/nginx/error.log   # k3s-lb: ssh -p 22010 kali@127.0.0.1
sudo journalctl -u mariadb -f          # k3s-db:  ssh -p 22011 kali@127.0.0.1
```

```zsh
kubectl get events -A --sort-by=.lastTimestamp | tail -30
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

Delete all VMs: **Run on** Mac. ⚠️ This deletes their disks.

```zsh
for row in $K3S_NODES; do
  VBoxManage controlvm ${row%% *} poweroff 2>/dev/null
  VBoxManage unregistervm ${row%% *} --delete
done
# keep 'kali-template' to rebuild quickly, or remove it too:
# VBoxManage unregistervm kali-template --delete
# the Kali desktop VM (Appendix E) is kept as well; to remove it:
# VBoxManage controlvm kali poweroff; VBoxManage unregistervm kali --delete
# the host-only network (Step 2):           VBoxManage hostonlynet remove --name=k3s-hostonly
# the Mac's home.lab names (Step 15, E.2):  sudo sed -i '' '/home\.lab/d' /etc/hosts
```

---


## Appendix E — Kali Linux desktop (GUI) workstation VM

`kali` is the **admin workstation**. Like every VM in this lab, it is a graphical Kali desktop in its own VirtualBox window, with the same specification (2 vCPU, 2 GB RAM, 16 MB video, 25 GB disk).
- **What's on it:** the Xfce desktop with the LightDM login screen, Firefox, a terminal, `kubectl` and `argocd`.
- **How it reaches the cluster:** through `k3s-lb`, the same way the Mac does. It is not a cluster node.

E.1 installs it on its own from the **same Kali ISO** as Step 3 (`Tools/iso`, arm64 on Apple Silicon), rather than cloning it. That way it gets its own **strong** password: it will hold cluster-admin credentials, while the template's password is shared by every clone. The GUI comes from the installer's **Software selection** screen, where you tick **Xfce**, as in Step 3.

> **Shortcut: clone it instead.** The template has the same desktop, Guest Additions and specification, so `kali` can also be a clone of it. **Run on:** Mac (`kali-template` powered off):
>
> ```zsh
> VBoxManage clonevm kali-template --name kali --register && VBoxManage modifyvm kali --groups "/k3s-ha"
> VBoxManage modifyvm kali --natpf1 delete ssh; VBoxManage modifyvm kali --natpf1 "ssh,tcp,127.0.0.1,22050,,22"
> VBoxManage startvm kali --type gui
> ```
>
> 1. Wait for its login screen, then run `ssh -n -p 22050 -o HostKeyAlias=kali-template kali@127.0.0.1 "sudo set-identity kali 192.168.99.50"`. The name is already in the `home.lab` block.
> 2. Then `ssh kali`, and run `passwd` (a strong password) and `sudo apt install -y firefox-esr`.
> 3. Continue with the last *Run on: Mac* block of E.2, then E.4. E.1, the rest of E.2 and E.3 are then not needed.

| Part | What | When |
|---|---|---|
| E.1 | create the VM, install Kali with the **Xfce desktop** | any time after Step 6 |
| E.2 | lab network, SSH, `home.lab` names | right after E.1 |
| E.3 | **Apple Silicon:** VirtualBox Guest Additions, so the desktop resizes with the window and the clipboard is shared with the Mac | right after E.2 |
| E.4 | `kubectl`, `argocd` and cluster access | after Step 12 (kubectl) and Step 16 (Argo CD) |

| Setting | Value |
|---|---|
| VM name / hostname / FQDN | `kali` / `kali` / `kali.home.lab` (VirtualBox group **k3s-ha**) |
| Window | a normal VirtualBox window on the Mac (`--type gui`), showing the Kali desktop |
| NIC 1 (`eth0`) | VirtualBox NAT → Internet, DHCP, default route; SSH port forward `127.0.0.1:22050 → 22` (used in E.2) |
| NIC 2 (`eth1`) | host-only `k3s-hostonly`, static **`192.168.99.50/24`**, no gateway, DNS search `home.lab` (set in E.2) |
| Desktop (GUI) | **Xfce** (`kali-desktop-xfce`, ticked in Software selection) with the LightDM graphical login; Firefox ESR |
| Window resize + shared clipboard | **Apple Silicon:** VirtualBox Guest Additions (E.3). **Intel Mac:** `virtualbox-guest-x11`, which the installer adds itself |
| User | `kali` (the password you set in its own installer, E.1) |
| vCPU / RAM / video / disk | 2 / 2 GB / 16 MB VMSVGA / 25 GB: the `k3s-lb` specification, like every VM |

> **Already have a `kali` clone** from an earlier version of this guide? Delete it before E.1: `VBoxManage controlvm kali poweroff; VBoxManage unregistervm kali --delete`. Or keep it and run `sudo tasksel` on it. Kali's `tasksel` is the program behind the installer's Software selection screen. Tick **Xfce**, then `sudo reboot`.

### E.1 Install `kali` with the Xfce desktop

**Run on:** Mac. This is Step 3 again, with the same specification (2 vCPU, 2 GB, 16 MB video, 25 GB disk) and the shared clipboard. Only the name and the SSH port forward (`22050`) differ.

```zsh
() {
  local VM=kali
  local ISO=( "$TOOLS"/iso/kali-linux-*-installer*-$KALI_ARCH.iso(N) ); ISO=$ISO[-1]
  local VDI="$HOME/VirtualBox VMs/$VM/$VM.vdi"
  print "ISO: $ISO"     # the same ISO as Step 3 (Tools/iso)
  [[ -f $ISO ]] || { print -u2 "no Kali $KALI_ARCH installer ISO in $TOOLS/iso - run Step 1.6"; return 1; }

  VBoxManage createvm --name $VM --platform-architecture=$VBOX_ARCH --ostype $VBOX_OSTYPE --default --register || return 1
  VBoxManage modifyvm $VM --cpus 2 --memory 2048 --vram 16 --graphicscontroller vmsvga --audio-enabled off   # the k3s-lb spec, like every VM
  VBoxManage modifyvm $VM --clipboard-mode bidirectional --drag-and-drop bidirectional
  VBoxManage modifyvm $VM --nic1 nat --nic2 hostonlynet --host-only-net2 $HON
  VBoxManage modifyvm $VM --natpf1 "ssh,tcp,127.0.0.1,22050,,22"     # Mac 127.0.0.1:22050 -> kali port 22 (E.2)
  VBoxManage modifyvm $VM --boot1 disk --boot2 dvd --boot3 none --boot4 none
  VBoxManage createmedium disk --filename "$VDI" --size 25600 --format VDI     # 25 GB like every VM
  VBoxManage storagectl $VM --name $VBOX_CTL --portcount 2
  VBoxManage storageattach $VM --storagectl $VBOX_CTL --port 0 --device 0 --type hdd --medium "$VDI"
  VBoxManage storageattach $VM --storagectl $VBOX_CTL --port 1 --device 0 --type dvddrive --medium "$ISO"
  VBoxManage modifyvm $VM --groups "/k3s-ha"
  VBoxManage startvm $VM --type gui
}
```

A window titled *kali [Running]* opens with the Kali installer. Click into it to type; the **left ⌘** key gives the keyboard and mouse back to the Mac. Go through the installer as in Step 3. Three screens differ: **Hostname**, the **password** and **Software selection**.

| Screen | Choose |
|---|---|
| Boot menu | **Graphical install** (or **Install**: the same screens in text mode) |
| Language / location / keyboard | your choice |
| Primary network interface | the **first** interface in the list, shown as `eth0` (or `enp0s…`), which is Adapter 1 = NAT. Both entries say *Intel 82540EM*, so go by the order: the second one is the host-only NIC and has no internet |
| Hostname | **`kali`** |
| Domain name | **`home.lab`** |
| Full name / username | `kali` / **`kali`** · a **strong** password, because this VM will hold cluster-admin credentials |
| Time zone | your choice |
| Partition disks | **Guided - use entire disk** → the disk → **All files in one partition** → Finish → Yes |
| HTTP proxy / mirror | leave empty / defaults |
| **Software selection** | tick **Xfce (Kali's default desktop environment)**, as shown below: this is the **GUI** |
| Boot loader | **Apple Silicon (EFI):** if it asks *Force GRUB installation to the EFI removable media path?*, answer **Yes**. **Intel Mac:** **Yes** → `/dev/sda` |
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
- The tool collections are optional and the cluster doesn't need them. Kali's size table (Kali 2024.1, Xfce) gives about **3.7 GB** with none of them, **6.7 GB** with *top10* and **13 GB** with *default*. With the 25 GB disk of this plan, leave them unticked. *top10* would still fit, if you want it on this VM.
- **Guest utilities.** They are not on this screen.
  - **On Apple Silicon**, Kali publishes no arm64 build of its `virtualbox-guest-x11` package, so the installer adds none. E.3 installs VirtualBox's own Guest Additions instead.
  - **On an Intel Mac**, the installer detects VirtualBox and adds `virtualbox-guest-x11` itself, and you skip E.3.

After **Continue**, `kali` reboots into the graphical **LightDM login screen** in its window.
- **Log in** as `kali`: the **Xfce desktop** appears, with the Kali menu at the top left.
- **Until E.3** (Apple Silicon), the desktop keeps a fixed size and there is no shared clipboard.
- **On a Retina display**, if the desktop looks tiny, use *View → Virtual Screen 1 → Scale to 200%* in the VM window.

### E.2 kali: network, SSH and the `home.lab` names

The installer only set up internet access. This part:
- gives `kali` the same network layout as the nodes: `eth0` = NAT and `eth1` = static `192.168.99.50`;
- adds the `home.lab` names and an SSH server;
- on Apple Silicon, installs the newest kernel and its headers for E.3.

The desktop has no shared clipboard yet, so this part goes over SSH from your Mac, like the template in Step 4.

1. **In the `kali` window**, open **Applications → Terminal Emulator** on the desktop, then type these three lines:

```bash
sudo apt update
sudo apt install -y openssh-server
sudo systemctl enable --now ssh
```

2. **Run on:** Mac. Connect through the `22050` port forward. It asks for the `kali` password:

```zsh
ssh-keygen -R "[127.0.0.1]:22050" >/dev/null 2>&1      # a new install has new host keys: forget those of an earlier kali VM
ssh -p 22050 kali@127.0.0.1                              # the kali desktop VM; Kali's zsh accepts the block below as it is
```

3. **Run on:** `kali`, in that SSH session. Type `sudo -v` and enter the `kali` password, then paste:

```bash
# 1) passwordless sudo: the same lab convenience as the nodes (Step 19 shuts kali down over SSH)
echo 'kali ALL=(ALL) NOPASSWD:ALL' | sudo tee /etc/sudoers.d/90-kali-nopasswd
sudo chmod 440 /etc/sudoers.d/90-kali-nopasswd

# 2) SSH server (the Mac uses it from now on, in E.3 and E.4) + Firefox
sudo apt update
sudo apt install -y openssh-server firefox-esr
sudo systemctl enable ssh

# 3) the same NIC names as the nodes: eth0 = Adapter 1 (NAT), eth1 = Adapter 2 (host-only)
printf 'auto lo\niface lo inet loopback\n' | sudo tee /etc/network/interfaces
sudo rm -f /var/lib/NetworkManager/no-auto-default.state
sudo mkdir -p /etc/default/grub.d
echo 'GRUB_CMDLINE_LINUX="$GRUB_CMDLINE_LINUX net.ifnames=0"' | sudo tee /etc/default/grub.d/90-eth-names.cfg
sudo update-grub
#    Apple Silicon (EFI): GRUB also on the fallback path EFI/BOOT/BOOTAA64.EFI, as in Step 4 part 2
if [ -d /sys/firmware/efi ] && dpkg -s grub-efi-arm64 >/dev/null 2>&1; then
  echo 'grub-efi-arm64 grub2/force_efi_extra_removable boolean true' | sudo debconf-set-selections
  sudo grub-install --target=arm64-efi --efi-directory=/boot/efi --force-extra-removable
fi

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

# 6) Apple Silicon: the newest kernel and its headers, so E.3 can build the VirtualBox Guest Additions for it
if [ "$(dpkg --print-architecture)" = arm64 ]; then
  sudo DEBIAN_FRONTEND=noninteractive apt -y -o Dpkg::Options::=--force-confdef -o Dpkg::Options::=--force-confold full-upgrade
  sudo DEBIAN_FRONTEND=noninteractive apt install -y build-essential dkms bzip2 linux-headers-arm64
fi

sudo reboot
```

The reboot closes the session. In the `kali` window, log in to the desktop again, open a terminal and run `ip -br a`. `eth0` should have `10.0.2.15` and `eth1` should have `192.168.99.50/24`.

**Run on:** Mac. Step 6 already added `Host kali` to `~/.ssh/config`. This block:
- copies your key (`ssh-copy-id` asks for the `kali` password once);
- checks the new address;
- adds the name `kali.home.lab` to the Mac (`sudo` asks for your Mac password).

```zsh
ssh-keygen -R 192.168.99.50 >/dev/null 2>&1        # a new install has new host keys: forget those of an earlier kali VM
ssh-copy-id -i ~/.ssh/id_ed25519.pub kali
ssh kali "hostname -f; ip -4 -br addr show eth1"   # kali.home.lab, then eth1 UP 192.168.99.50/24 - no password
grep -q 'kali\.home\.lab' /etc/hosts || printf '192.168.99.50  kali.home.lab\n' | sudo tee -a /etc/hosts >/dev/null
sudo dscacheutil -flushcache; sudo killall -HUP mDNSResponder
```

### E.3 kali (Apple Silicon): VirtualBox Guest Additions for the GUI

The Guest Additions make the Kali desktop **follow the window size** and share the **clipboard** with the Mac in both directions. Kali has no arm64 build of its own `virtualbox-guest-x11`, so on Apple Silicon you install the Guest Additions that ship with VirtualBox (`VBoxLinuxAdditions-arm64.run` on its Guest Additions CD).

**Intel Mac:** skip E.3. The installer already added `virtualbox-guest-x11`.

**Run on:** Mac. Insert VirtualBox's Guest Additions CD into the running `kali` VM:

```zsh
VBoxManage storageattach kali --storagectl $VBOX_CTL --port 1 --device 0 --type dvddrive --medium additions
```

**Run on:** `kali` (`ssh kali`). This builds the kernel modules for the running kernel, which takes a few minutes:

```bash
sudo mount -o ro /dev/sr0 /mnt
sudo sh /mnt/VBoxLinuxAdditions-arm64.run
sudo umount /mnt
sudo reboot
```

After the reboot, check over SSH (`ssh kali`):

```bash
lsmod | grep '^vbox'                    # vboxguest (and vboxsf)
systemctl is-active vboxadd-service     # active
```

**In the `kali` window**, log in to the desktop.
1. Make sure *View → Auto-resize Guest Display* is ticked, then drag the window bigger: the Xfce desktop grows with it.
2. Test the clipboard. Copy text on the Mac with **⌘C** and paste it into Kali's terminal with **Ctrl+Shift+V**. The other way round: **Ctrl+Shift+C** in Kali, **⌘V** on the Mac.

**Run on:** Mac. Eject the CD and keep this state as a snapshot:

```zsh
VBoxManage storageattach kali --storagectl $VBOX_CTL --port 1 --device 0 --type dvddrive --medium emptydrive
VBoxManage snapshot kali take "01-desktop" --live
```

> **After Kali kernel updates**, the Guest Additions rebuild their modules for the new kernel. Keep `linux-headers-arm64` installed (E.2 item 6) so they can. If the window stops resizing after an upgrade, run E.3 again.
>
> **If the installer fails** (see Appendix C), the desktop still works: the GUI just keeps a fixed size and has no shared clipboard. Keep pasting over `ssh kali` from the Mac, and try again after the next VirtualBox update (Step 1.2).

### E.4 kali: cluster tools and access

This needs the cluster: the kubectl part works after **Step 12**, the Argo CD part after **Step 16**.

**Run on:** `kali`: `ssh kali` from the Mac, or **Applications → Terminal Emulator** on its desktop. After E.3 you can paste there with **Ctrl+Shift+V**.

```bash
# kubectl + Argo CD CLI (official binaries for this VM's CPU: arm64 on Apple Silicon, amd64 on an Intel Mac)
ARCH=$(dpkg --print-architecture)
curl -fsSLo /tmp/kubectl "https://dl.k8s.io/release/$(curl -fsSL https://dl.k8s.io/release/stable.txt)/bin/linux/$ARCH/kubectl"
curl -fsSLo /tmp/argocd "https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-$ARCH"
sudo install -m 0755 /tmp/kubectl /tmp/argocd /usr/local/bin/
kubectl version --client && argocd version --client

# Kali's own SSH key + a shortcut to the nodes (the Mac authorizes the key next)
mkdir -p ~/.ssh && chmod 700 ~/.ssh
[ -f ~/.ssh/id_ed25519 ] || ssh-keygen -t ed25519 -N '' -f ~/.ssh/id_ed25519
grep -q '^Host k3s-\*' ~/.ssh/config 2>/dev/null || cat >> ~/.ssh/config <<'EOF'
Host k3s-*
    StrictHostKeyChecking accept-new
EOF
chmod 600 ~/.ssh/config
```

**Run on:** Mac. Copy the kubeconfig, which Step 12 already pointed at `192.168.99.10`, and authorize Kali's key on every node:

```zsh
ssh kali "mkdir -p ~/.kube && chmod 700 ~/.kube"
scp ~/.kube/config kali:.kube/config
ssh kali "chmod 600 ~/.kube/config"

KALI_KEY=$(ssh kali "cat ~/.ssh/id_ed25519.pub")
for row in $K3S_NODES; do
  ssh -n ${row%% *} "grep -qxF '$KALI_KEY' ~/.ssh/authorized_keys || echo '$KALI_KEY' >> ~/.ssh/authorized_keys"
done
```

**Run on:** `kali`:

```bash
kubectl config current-context                # k3s-ha
kubectl get nodes -o wide                     # the same 5 nodes as on the Mac
ssh k3s-srv-1 hostname                        # k3s-srv-1 - key-based, no password
curl -skI https://argocd.home.lab | head -1   # HTTP/2 200 (after Step 15)
argocd login argocd.home.lab --username admin --grpc-web --insecure    # password from Step 16
argocd app list
```

**In the GUI:** open the Kali menu (top left) → **Web Browser** (Firefox ESR) and go to **<https://argocd.home.lab>**. It uses the same `/etc/hosts` entry and the same login as on the Mac. The Xfce *Terminal Emulator* runs the same `kubectl` and `argocd` commands.

> `~/.kube/config` on Kali holds the **cluster-admin** credentials, the same as on the Mac. Give Kali a strong password and keep it on the host-only network.

---

## What ended up on each VM

| VM | Installed / running | Key files |
|---|---|---|
| **Mac** | `~/Desktop/Working/Tools/`: `bin/kubectl`, `bin/argocd` (darwin-arm64), `iso/` (Kali arm64 installer), `downloads/` (VirtualBox `.dmg`), `k3s-lab.zsh`. VirtualBox in `/Applications/VirtualBox.app` (`VBoxManage` in `/usr/local/bin`), with the Step 2 networks: host-only network `k3s-hostonly` (Mac `192.168.99.100`, macOS DHCP `.101–.254`) and NAT network `NatNetwork`. OpenSSH and git from macOS | `~/.zshrc` (one line that loads `k3s-lab.zsh`: `Tools/bin` first on PATH, `interactivecomments`, `K3S_NODES`), `~/.kube/config`, `~/.ssh/config`, `~/.ssh/id_ed25519`, `/etc/hosts`, `~/Desktop/Working/homelab-gitops/` |
| **kali-template** | Kali Linux arm64 with the **Xfce desktop** + LightDM, VirtualBox Guest Additions (Apple Silicon, Step 4 part 3b), NetworkManager, `set-identity`, `home.lab` hosts block, GRUB also on the EFI fallback path. 2 vCPU / 2 GB / 16 MB video / 25 GB, shared clipboard | `/usr/local/sbin/set-identity`, `/etc/sudoers.d/90-kali-nopasswd` + `91-secure-path`, `/etc/default/grub.d/90-eth-names.cfg`, `/boot/efi/EFI/BOOT/BOOTAA64.EFI`, `/etc/NetworkManager/conf.d/90-k3s.conf`, `~/.ssh/authorized_keys` (powered off, source of the 7 clones). VirtualBox NAT port forward `127.0.0.1:2222 → 22` (`ssh kali-template`) |
| **every clone** (the 7 cluster VMs) | everything from the template (Xfce desktop, Guest Additions and the 2 vCPU / 2 GB / 16 MB / 25 GB specification included), plus its identity from `set-identity`, and its own NAT port forward `127.0.0.1:220xx → 22` (Step 5) | `/etc/hosts`, `/etc/NetworkManager/system-connections/{nat,hostonly}.nmconnection` |
| **k3s-lb** | `nginx` + `libnginx-mod-stream` | `/etc/nginx/nginx.conf` |
| **k3s-db** | `mariadb-server` (database `k3s`, user `k3s`) | `/etc/mysql/mariadb.conf.d/50-server.cnf` + `90-k3s.cnf`, `~/k3s-secrets.env`, `/etc/cron.d/k3s-db-backup` |
| **k3s-srv-1/2** | `k3s` (server, tainted) | `/etc/rancher/k3s/config.yaml`, `/etc/rancher/k3s/k3s.yaml` |
| **k3s-agent-1/2/3** | `k3s-agent`, plus pods: Traefik, svclb, Argo CD, apps | `/etc/rancher/k3s/config.yaml` |
| **kali** | its own Kali install with a **GUI**: Xfce desktop + LightDM (Software selection), VirtualBox Guest Additions (E.3, Apple Silicon) or `virtualbox-guest-x11` (Intel), OpenSSH server, Firefox ESR, `kubectl`, `argocd`. NAT port forward `127.0.0.1:22050 → 22` | `/etc/NetworkManager/system-connections/{nat,hostonly}.nmconnection`, `/etc/default/grub.d/90-eth-names.cfg`, `/etc/hosts`, `~/.kube/config`, `~/.ssh/config`, `~/.ssh/id_ed25519` |

---

## Sources

- The Windows edition of this guide — [`k3s-argocd-setup.md`](k3s-argocd-setup.md)
- Video: Christian Lempa — *Using \*HA\* Kubernetes at home, was never so simple!* — <https://www.youtube.com/watch?v=UdjhFLV1yt8>
- Christian Lempa's k3s cheat-sheet (HA with external DB), 2022–2023 versions — <https://github.com/ChristianLempa/cheat-sheets/blob/0a92b5a9ffb42da00717060089bb0d9ed37b940f/kubernetes/k3s.md>
- K3s docs: High Availability with an External DB — <https://docs.k3s.io/datastore/ha>
- K3s docs: Cluster Load Balancer (NGINX / HAProxy + keepalived) — <https://docs.k3s.io/datastore/cluster-loadbalancer>
- K3s docs: Configuration file — <https://docs.k3s.io/installation/configuration>
- Argo CD: Getting Started — <https://argo-cd.readthedocs.io/en/stable/getting_started/>
- Argo CD: Ingress configuration (Traefik) — <https://argo-cd.readthedocs.io/en/stable/operator-manual/ingress/>
- VirtualBox manual: VBoxManage — <https://www.virtualbox.org/manual/>
- VirtualBox manual: host-only networking (host-only networks on macOS) — <https://www.virtualbox.org/manual/topics/networkingdetails.html>
- VirtualBox 7.1 changelog (macOS / Arm64 hosts supported) — <https://www.virtualbox.org/wiki/Changelog-7.1>
- VirtualBox manual: installing on macOS — <https://www.virtualbox.org/manual/topics/installation.html>
- VirtualBox downloads and SHA256SUMS (`…-macOSArm64.dmg`, `…-OSX.dmg`) — <https://download.virtualbox.org/virtualbox/>
- Kubernetes docs: Install kubectl on macOS — <https://kubernetes.io/docs/tasks/tools/install-kubectl-macos/>
- Apple: TN3179, Understanding local network privacy — <https://developer.apple.com/documentation/technotes/tn3179-understanding-local-network-privacy>
- Kali Linux current images and SHA256SUMS — <https://cdimage.kali.org/current/>
- Kali Linux: installer images — <https://www.kali.org/get-kali/#kali-installer-images>
- Kali docs: Installing Kali Linux (hard disk install) — <https://www.kali.org/docs/installation/hard-disk-install/>
- Kali docs: VirtualBox guest additions in Kali — <https://www.kali.org/docs/virtualization/install-virtualbox-guest-additions/>
- Kali docs: installation sizes per desktop and tool collection — <https://www.kali.org/docs/installation/installation-sizes/>
- Kali's `tasksel` (the installer's Software selection entries, `kali-tasks/`) — <https://gitlab.com/kalilinux/packages/tasksel>
- Kubernetes docs: Install kubectl on Linux — <https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/>
- Argo CD: CLI installation — <https://argo-cd.readthedocs.io/en/stable/cli_installation/>
- podinfo (multi-arch demo app, `kustomize/`) — <https://github.com/stefanprodan/podinfo>
- Argo CD example apps (guestbook; its image is amd64-only) — <https://github.com/argoproj/argocd-example-apps>
