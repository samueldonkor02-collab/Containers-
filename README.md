# Containers-
Windows Docker containers lab — creation, exec, user &amp; network inspection. Cybersecurity learning notes.
# Windows Containers Lab (Docker on Windows Server) — CompTIA Assisted Lab

## What this is

This repo documents a hands-on lab I completed as part of my cybersecurity studies, working with **Docker containers on a Windows Server host** through CompTIA's assisted lab platform. The lab environment was a remote Windows 10/Server VM (hostname `MS10`) with Docker already installed, and the goal was to get comfortable creating, running, and interacting with Windows containers — plus poking around inside them to understand what a container actually looks like from the inside.

I'm putting this up mainly as a record of what I did and what I learned, not as a polished project — screenshots included so you can see the actual terminal output.

## Environment

- **Host OS:** Windows 10 (build 17763.4851 — Server 1809 base)
- **Hostname:** MS10
- **Container engine:** Docker
- **Base image used:** `mcr.microsoft.com/windows/nanoserver:1809`
- **Access:** Remote lab client via LabOnDemand, run through PowerShell

## What I did, step by step

### 1. Checked the environment first
Before touching Docker, I ran a few basic recon commands to understand the host:
```
hostname
winver
ipconfig
```
This showed the host was `MS10`, running Windows build 17763.4851, with a real network adapter (`10.1.16.13`) plus a Docker-created virtual switch (`vEthernet (nat)` on `172.23.96.1`). Good reminder that Docker on Windows sets up its own NAT network under the hood, separate from the host's actual LAN adapter.

### 2. Pulled and inspected the image
```
docker images
```
Confirmed the `nanoserver:1809` image was already present (~252MB). Nanoserver is Microsoft's minimal, headless container base image — no GUI, stripped down, built specifically for containers rather than a full Windows install.

### 3. Created my first container
```
docker create --name MyFirstContainer mcr.microsoft.com/windows/nanoserver:1809
docker ps -a
```
`docker create` just builds the container without starting it — you can see it sitting in a `Created` state in `docker ps -a` before anything is running. This was a useful distinction to actually see rather than just read about: **create ≠ start ≠ run**.

### 4. Started it, then tried running a second one interactively
```
docker start MyFirstContainer
docker run -it --name TestContainer mcr.microsoft.com/windows/nanoserver:1809
```
`docker run -it` combines create + start + attach an interactive terminal in one go, which dropped me straight into a `cmd.exe` prompt *inside* the container. From there I ran:
```
ver
ipconfig
```
which confirmed I was in a genuinely separate environment — same Windows build as the host, but its own network adapter info scoped to the container.

### 5. Exited and reattached to check container lifecycle
```
exit
docker ps -a
docker start TestContainer
docker ps -a
```
Exiting the container's shell stops the container (status flips to `Exited`), but the container itself isn't deleted — it's still there, just not running, until you either restart it or explicitly remove it. This tripped me up at first since I expected `exit` to just close a window, not stop the whole container.

### 6. Used `exec` instead of `run` to get back in
```
docker exec -it TestContainer cmd
echo %username%
net user
net user Administrator
```
This was the important distinction for me: `docker run` creates a *new* container, but `docker exec` opens a shell in a container that's *already running*. Using `exec` on `TestContainer` (after starting it) let me poke around without spinning up yet another instance.

Inside the container:
- Logged in user was `ContainerUser` — not `Administrator`, which is the default least-privilege behavior for Windows containers.
- `net user` listed the local accounts inside the container (`Administrator`, `DefaultAccount`, `Guest`, `WDAGUtilityAccount`) — these are container-local accounts, separate from the host's user accounts.
- `net user Administrator` showed account details (password never expires, account active, etc.) — useful for thinking about container hardening later, since a container with a never-expiring admin password is exactly the kind of thing you'd flag in a security review.

### 7. Networking check between host and container
```
ping 10.1.16.13
```
Confirmed connectivity back to the host's real network adapter from within the lab session — 0% packet loss, sub-1ms replies. Good sanity check that container networking wasn't blocking basic connectivity.

### 8. Cleaned up
```
docker stop TestContainer
```
Stopped the running container to wrap up the session cleanly rather than just closing the window.

## What I actually learned

- **Containers vs VMs**: this was the big one. A Windows container isn't a lightweight VM — it shares the host's kernel but isolates the filesystem, network stack, and process view. That's why `ver` inside the container matched the host build exactly, but `ipconfig` and the user list were completely different.
- **create / start / run / exec are all different**: I knew this in theory before the lab, but actually watching `docker ps -a` change state between each command made it click properly.
- **Default container users are locked down**: running as `ContainerUser` rather than `Administrator` by default is a small but deliberate security decision — least privilege baked into the base image.
- **Stopped ≠ deleted**: containers persist in a stopped state until removed, which matters for both troubleshooting and cleanup hygiene (leaving dead containers around is its own minor hygiene/security smell in real environments).
- **Docker's own virtual networking**: the `vEthernet (nat)` adapter showing up in `ipconfig` on the host was a reminder that Docker quietly reconfigures host networking when installed — something worth knowing if you're ever auditing a machine and see adapters you didn't expect.

## Skills Demonstrated

- **Containerization fundamentals** — creating, starting, running, exec'ing into, and stopping Docker containers on Windows
- **Docker CLI proficiency** — `docker images`, `docker create`, `docker start`, `docker run -it`, `docker exec -it`, `docker ps -a`, `docker stop`
- **Windows command-line / PowerShell** — navigating and querying a Windows host via `cmd` and PowerShell (`dir`, `hostname`, `winver`, `ipconfig`, `net user`, `ping`)
- **Container vs. host isolation analysis** — comparing network configuration, user accounts, and filesystem state between the host and a running container to understand the isolation boundary
- **User and account enumeration** — using `net user` and `net user Administrator` to inspect local accounts and account policy (password expiry, account status) inside a container
- **Least-privilege awareness** — identifying that containers run as a restricted `ContainerUser` by default rather than `Administrator`, and recognizing why that matters from a security standpoint
- **Network troubleshooting** — using `ipconfig` and `ping` to verify connectivity and understand Docker's NAT networking layer (`vEthernet (nat)`) alongside the host's physical adapter
- **Container lifecycle management** — understanding and demonstrating the difference between a container that's created, running, and stopped, and the state transitions between them
- **Remote lab environment navigation** — working in a browser-based virtual lab (LabOnDemand / CompTIA Learning Platform) to complete hands-on technical exercises
- **Technical documentation** — writing up a hands-on lab clearly enough for someone else (or future me) to follow and understand what was done and why

## Results

- Successfully pulled and confirmed the `nanoserver:1809` base image was available locally (~252MB) before creating any containers.
- Created two containers (`MyFirstContainer`, `TestContainer`) and verified each moved correctly through the states `Created → Up/Running → Exited` as shown in repeated `docker ps -a` checks.
- Got an interactive shell inside a running container via both `docker run -it` and `docker exec -it`, and confirmed both methods land you in the container's own `cmd.exe`, not the host's.
- Verified the container reports the same Windows build (`10.0.17763.4851`) as the host via `ver`, confirming shared-kernel behavior rather than a separate OS install.
- Confirmed the container runs as `ContainerUser` by default (via `echo %username%`) rather than `Administrator`, and enumerated the container's local accounts with `net user`.
- Verified `Administrator`'s account inside the container: active, password set with no expiry — a real, checkable finding rather than an assumption, and one I'd flag if this were a security assessment.
- Verified network separation between host and container: `ipconfig` on the host showed a physical adapter (`10.1.16.13`) plus a Docker-managed `vEthernet (nat)` adapter (`172.23.96.1`), distinct from the container's own internal network view.
- Confirmed connectivity from the container session back to the host's real adapter with a clean `ping 10.1.16.13` — 4/4 packets, 0% loss, <1ms round trip.
- Cleanly stopped the running container (`docker stop TestContainer`) at the end of the session rather than leaving it running or force-closing the window — good habit for lab hygiene.

**Bottom line:** by the end of the lab I could reliably create, start, run, exec into, inspect, and stop Windows containers from the CLI, and I had concrete, verified evidence (not just theory) of how a container's user context and network stack differ from the host it's running on.

## Screenshots

Terminal output from each stage of the lab is included in this repo for reference (`docker ps -a` states, `docker exec` session, `net user` output, networking checks, etc.).

---
*Lab completed as part of ongoing cybersecurity coursework. Notes written up afterward for my own reference and to track progress.*

## About Me

I'm coming from a healthcare background and am now transitioning into IT and cybersecurity. This repo is part of that journey — documenting labs as I go, both to reinforce what I'm learning and to build a track record of hands-on work as I move into the field.

I'm still early on, so if you spot something I've got wrong, missed, or could explain better, I'd genuinely appreciate the feedback — feel free to reach out.

**LinkedIn:** https://www.linkedin.com/in/samuel-donkor-9258aa223?utm_source=share_via&utm_content=profile&utm_medium=member_ios)

**Email: samuelcyber02@gmail.com

