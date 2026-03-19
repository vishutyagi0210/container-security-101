# 🐧 Ubuntu 24.04 (AWS EC2) → Create User → Docker Rootless Setup
### A Complete Personal Guide — From Fresh Instance to Running Containers

---

## 🧠 The Core Concept — Why NO Sudo for dockeruser?

The entire point of rootless Docker is **security isolation**:

> If `dockeruser` has `sudo`, they can become root anytime.
> That completely defeats the purpose of rootless mode.

| Who | What They Do | Has Sudo? |
|---|---|---|
| **Admin** (`ubuntu`) | System setup, install Docker, create user, create system service | ✅ Yes |
| **dockeruser** | Runs Docker, runs containers — nothing else | ❌ Never |

---

## ⚠️ EC2 Reality — What's Different from a Regular VM

On a regular Ubuntu VM, Docker rootless uses **systemd user sessions** automatically.
On AWS EC2, **systemd user sessions are NOT available** for non-root users. This causes two problems if not handled:

- `systemctl --user` doesn't work → Docker won't auto-start
- systemd blocks cgroup creation per container → every `docker run` crashes

Both are solved in this guide.

| | Regular VM | AWS EC2 |
|---|---|---|
| Systemd user session | ✅ Available | ❌ Not available |
| `systemctl --user` | ✅ Works | ❌ Doesn't work |
| cgroup delegation | ✅ Auto-handled | ❌ Must be explicitly configured |
| Docker socket path | `/run/user/1001/docker.sock` | `/home/dockeruser/.docker/run/docker.sock` |
| Solution for daemon | `systemctl --user enable docker` | Admin creates a system-level service |
| App survives logout | ✅ Yes (with linger) | ✅ Yes (with system service) |

---

## 🗺️ The Full Roadmap

```
Fresh Ubuntu 24.04 EC2 Instance
        ↓
[ADMIN] Update System + Install Tools
        ↓
[ADMIN] Create dockeruser — NO sudo, ever
        ↓
[ADMIN] Install Docker Engine + rootless-extras + dependencies
        ↓
[ADMIN] Create daemon.json with cgroupfs driver
        ↓
[SWITCH] su - dockeruser
        ↓
[dockeruser] Run dockerd-rootless-setuptool.sh install
        ↓
[dockeruser] Set Environment Variables in ~/.bashrc
        ↓
[SWITCH BACK TO ADMIN]
        ↓
[ADMIN] Create system-level service (with Delegate=yes)
        ↓
[ADMIN] Create cgroup delegation config
        ↓
[ADMIN] Enable + Start the service
        ↓
[dockeruser] Test & Verify ✅
```

---

## 🖥️ Phase 1 — System Setup (As Admin)

### Step 1 — Update Everything

Log in as `ubuntu` (the default EC2 admin user) and update first:

```bash
sudo apt-get update && sudo apt-get upgrade -y
```

**Proof it worked:**
```
0 upgraded, 0 newly installed, 0 to remove and 0 not upgraded.
```

---

### Step 2 — Install Essential Tools

```bash
sudo apt-get install -y curl wget git nano
```

| Tool | Purpose |
|---|---|
| `curl` | Downloads scripts from the internet |
| `wget` | Another download tool |
| `git` | Version control |
| `nano` | Simple terminal text editor |

---

## 👤 Phase 2 — Create dockeruser (As Admin — NO Sudo Given!)

### Step 3 — Create the New User

```bash
sudo adduser dockeruser
```

Ubuntu will ask you a few questions:

```
New password:          ← type a strong password
Retype new password:   ← confirm it
Full Name []:          ← press Enter to skip all fields
...
Is the information correct? [Y/n]  ← Y + Enter
```

**What `adduser` does automatically:**
- Creates the account and home folder at `/home/dockeruser`
- Creates a group with the same name
- Sets up `/bin/bash` as the shell

---

### Step 4 — ❌ Do NOT Give Sudo Privileges

```bash
# ❌ NEVER run this:
# sudo usermod -aG sudo dockeruser   ← DO NOT DO THIS

# ✅ dockeruser must have ZERO sudo — that's the whole point
```

If `dockeruser` had sudo, they could `sudo su` and become root instantly. The security boundary would be completely pointless.

---

### Step 5 — Optionally Add to systemd-journal Group

```bash
sudo usermod -aG systemd-journal dockeruser
```

Lets `dockeruser` read system logs for debugging. No elevated privileges granted.

---

### Step 6 — Verify No Sudo

```bash
groups dockeruser
```

Expected output:
```
dockeruser : dockeruser systemd-journal
```

✅ `sudo` must NOT appear here. If it does, you made a mistake in Step 4.

---

## 🐳 Phase 3 — Install Docker (As Admin)

### Step 7 — Install Docker Engine

```bash
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
```

**Installs:**
- `docker-ce` — Docker Community Edition
- `docker-ce-cli` — The `docker` CLI tool
- `containerd.io` — Container runtime
- `docker-compose-plugin` — Docker Compose (built-in)

**Proof:**
```bash
docker --version
```
```
Docker version 27.x.x, build xxxxxxx
```

---

### Step 8 — Install docker-ce-rootless-extras ← Critical!

This package contains `dockerd-rootless-setuptool.sh` and `dockerd-rootless.sh` — without it, rootless mode doesn't exist:

```bash
sudo apt-get install -y docker-ce-rootless-extras
```

**Proof:**
```bash
which dockerd-rootless-setuptool.sh
which dockerd-rootless.sh
```
```
/usr/bin/dockerd-rootless-setuptool.sh
/usr/bin/dockerd-rootless.sh
```

---

### Step 9 — Install All Rootless Dependencies

```bash
sudo apt-get install -y uidmap dbus-user-session slirp4netns
```

> ⚠️ Must be done by admin. `dockeruser` has no sudo and cannot install packages.

| Package | Purpose |
|---|---|
| `uidmap` | Maps user IDs for isolated user namespaces without root |
| `dbus-user-session` | Lets a non-root user run background services |
| `slirp4netns` | User-space networking — gives containers internet access |

---

### Step 10 — Create daemon.json with cgroupfs Driver ← Critical for EC2!

On EC2, Docker's default cgroup manager tries to create systemd scope units per container, which requires polkit authentication that never comes — causing every `docker run` to fail with:

```
unable to apply cgroup configuration... Interactive authentication required
```

The fix is to tell Docker to use `cgroupfs` directly instead:

```bash
sudo mkdir -p /home/dockeruser/.config/docker
sudo bash -c 'cat > /home/dockeruser/.config/docker/daemon.json << EOF
{
  "exec-opts": ["native.cgroupdriver=cgroupfs"]
}
EOF'
sudo chown -R dockeruser:dockeruser /home/dockeruser/.config/docker
```

**What this does:**
- `exec-opts: native.cgroupdriver=cgroupfs` → Docker writes directly to the cgroup filesystem instead of asking systemd to create scope units
- `chown` → Makes sure `dockeruser` owns their own config (not root)

**Proof:**
```bash
sudo cat /home/dockeruser/.config/docker/daemon.json
```
```json
{
  "exec-opts": ["native.cgroupdriver=cgroupfs"]
}
```

---

## 🔐 Phase 4 — Setup Docker Rootless (As dockeruser — Zero Sudo!)

Switch to the new user. **No sudo used from this point forward.**

```bash
su - dockeruser
whoami   # must output: dockeruser
```

---

### Step 11 — Run the Rootless Setup Script

```bash
dockerd-rootless-setuptool.sh install
```

> ✅ No `sudo`. No root. Run directly as `dockeruser`.

**On EC2, the expected output is:**
```
[INFO] systemd not detected, dockerd-rootless.sh needs to be started manually:
PATH=/usr/bin:/sbin:/usr/sbin:$PATH dockerd-rootless.sh
[INFO] CLI context "rootless" already exists
[INFO] Using CLI context "rootless"
Current context is now "rootless"
[INFO] Make sure the following environment variable(s) are set (or add them to ~/.bashrc):
export XDG_RUNTIME_DIR=/home/dockeruser/.docker/run
export PATH=/usr/bin:$PATH
export DOCKER_HOST=unix:///home/dockeruser/.docker/run/docker.sock
```

✅ "systemd not detected" is **normal and expected on EC2** — not an error. The system-level service we create in Phase 5 handles this.

---

### Step 12 — Set Environment Variables

```bash
echo 'export XDG_RUNTIME_DIR=/home/dockeruser/.docker/run' >> ~/.bashrc
echo 'export PATH=/usr/bin:/sbin:/usr/sbin:$PATH' >> ~/.bashrc
echo 'export DOCKER_HOST=unix:///home/dockeruser/.docker/run/docker.sock' >> ~/.bashrc
source ~/.bashrc
```

| Variable | Purpose |
|---|---|
| `XDG_RUNTIME_DIR` | Tells Docker where to create its socket and runtime files |
| `PATH` | Makes sure all Docker binaries are findable |
| `DOCKER_HOST` | Tells the Docker CLI exactly where the daemon socket lives |

**Proof:**
```bash
echo $XDG_RUNTIME_DIR
echo $DOCKER_HOST
```
```
/home/dockeruser/.docker/run
unix:///home/dockeruser/.docker/run/docker.sock
```

---

### Step 13 — Create the Runtime Directory

```bash
mkdir -p /home/dockeruser/.docker/run
```

This is where the Docker socket will live. The system service also creates it via `ExecStartPre`, but good to have it ready manually too.

---

## 🛠️ Phase 5 — Create System Service (Back to Admin!)

Switch back to admin:

```bash
exit
```

### Why a System Service?

Without this, the moment `dockeruser` logs out — Docker dies and so does every running container.

A system-level service means:
- Docker starts at **boot** — no login required
- Docker keeps running even when `dockeruser` is **completely logged out**
- Docker **auto-restarts** if it crashes
- `dockeruser` still has **zero sudo**

---

### Step 14 — Create the System Service File

```bash
sudo nano /etc/systemd/system/docker-rootless-dockeruser.service
```

Paste this exactly:

```ini
[Unit]
Description=Docker Rootless Daemon (dockeruser)
After=network.target

[Service]
Type=simple
User=dockeruser
Environment=HOME=/home/dockeruser
Environment=XDG_RUNTIME_DIR=/home/dockeruser/.docker/run
Environment=PATH=/usr/bin:/sbin:/usr/sbin:/usr/local/bin
Environment=DOCKER_HOST=unix:///home/dockeruser/.docker/run/docker.sock
ExecStartPre=/bin/mkdir -p /home/dockeruser/.docker/run
ExecStart=/usr/bin/dockerd-rootless.sh
Restart=always
RestartSec=5
Delegate=yes

[Install]
WantedBy=multi-user.target
```

| Setting | What It Does |
|---|---|
| `User=dockeruser` | Runs the daemon as `dockeruser` — NOT as root |
| `ExecStartPre=mkdir` | Creates the socket directory before starting |
| `ExecStart=dockerd-rootless.sh` | The actual rootless Docker daemon binary |
| `Restart=always` | Auto-restarts Docker if it ever crashes |
| `RestartSec=5` | Waits 5 seconds before restarting |
| `Delegate=yes` | Allows this service to manage its own cgroup subtree — required on EC2 |
| `WantedBy=multi-user.target` | Starts automatically at boot |

Save and exit: `Ctrl+O` → `Enter` → `Ctrl+X`

---

### Step 15 — Create cgroup Delegation Config ← Required on EC2!

Even with `Delegate=yes` in the service file, you must also tell systemd which cgroup controllers to hand over to `dockeruser`'s slice. Without this, container isolation still fails:

```bash
sudo mkdir -p /etc/systemd/system/user@.service.d/
sudo nano /etc/systemd/system/user@.service.d/delegate.conf
```

Paste this:

```ini
[Service]
Delegate=cpu cpuset io memory pids
```

Save and exit: `Ctrl+O` → `Enter` → `Ctrl+X`

**What this does:** Grants processes under `dockeruser`'s slice full control over `cpu`, `cpuset`, `io`, `memory`, and `pids` cgroup controllers — exactly what Docker needs to isolate each container.

---

### Step 16 — Enable and Start the Service

```bash
# Reload systemd — picks up the service file AND delegate.conf
sudo systemctl daemon-reload

# Auto-start on every boot
sudo systemctl enable docker-rootless-dockeruser

# Start right now
sudo systemctl start docker-rootless-dockeruser
```

**Proof it's running:**
```bash
sudo systemctl status docker-rootless-dockeruser
```
Expected:
```
● docker-rootless-dockeruser.service - Docker Rootless Daemon (dockeruser)
     Loaded: loaded (/etc/systemd/system/docker-rootless-dockeruser.service; enabled)
     Active: active (running) since ...
   Main PID: XXXX (dockerd-rootless.)
```

Also confirm it's running as `dockeruser`, NOT root:
```bash
ps aux | grep dockerd
```
You should see `dockerd` listed under `dockeruser`. ✅

---

## ✅ Phase 6 — Final Testing & Proof (As dockeruser)

```bash
su - dockeruser
```

### Step 17 — Run the Hello World Container

```bash
docker run hello-world
```

Expected output:
```
Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
 3. The Docker daemon created a new container from that image...
 4. The Docker daemon streamed that output to the Docker client...
```

🎉 **If you see this — everything is working correctly!**

---

### Step 18 — Confirm It's Actually Rootless

```bash
docker info | grep -i rootless
```
```
 rootlesskit
 Security Options: rootless
```

```bash
ps aux | grep dockerd
```
`dockerd` must appear under `dockeruser`, NOT `root`. ✅

---

### Step 19 — Prove the App Survives Logout

Start a container in the background:

```bash
docker run -d --name test-survivor nginx
docker ps   # confirm it's running
```

Log out completely:

```bash
exit
```

Log back in as `dockeruser`:

```bash
su - dockeruser
docker ps   # container is STILL running ✅
```

The container survived because Docker is managed by the system service — not by the user session. ✅

---

### Step 20 — Run a Real Interactive Container

```bash
docker run -it ubuntu:24.04 bash
```

Inside the container:

```bash
whoami               # shows 'root' INSIDE container only — NOT on host
cat /etc/os-release  # shows Ubuntu info
exit
```

> 💡 `whoami` shows `root` inside the container, but on the **host** the process runs as `dockeruser`. That's rootless working exactly as designed.

---

## 🔧 Bonus — Fix Low Ports (80, 443) — Admin Must Do This

`dockeruser` has no sudo so the admin applies this:

```bash
# Permanent fix (survives reboots)
echo "net.ipv4.ip_unprivileged_port_start=80" | sudo tee /etc/sysctl.d/99-rootless-docker.conf
sudo sysctl --system
```

After this, `dockeruser` can bind containers to ports 80, 443, etc.

---

## 📋 Complete Summary — Every Command in Order

```bash
# ══════════════════════════════════════════════════════════════
# PHASE 1–3: AS ADMIN (ubuntu user)
# ══════════════════════════════════════════════════════════════

# System setup
sudo apt-get update && sudo apt-get upgrade -y
sudo apt-get install -y curl wget git nano

# Create user — ZERO sudo, ever
sudo adduser dockeruser
sudo usermod -aG systemd-journal dockeruser   # optional, for log reading only

# Verify — sudo must NOT appear
groups dockeruser   # expected: dockeruser systemd-journal

# Install Docker Engine
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
docker --version   # verify

# Install rootless package + dependencies
sudo apt-get install -y docker-ce-rootless-extras
sudo apt-get install -y uidmap dbus-user-session slirp4netns

# Create daemon.json — switches cgroup driver to cgroupfs (fixes EC2 container crash)
sudo mkdir -p /home/dockeruser/.config/docker
sudo bash -c 'cat > /home/dockeruser/.config/docker/daemon.json << EOF
{
  "exec-opts": ["native.cgroupdriver=cgroupfs"]
}
EOF'
sudo chown -R dockeruser:dockeruser /home/dockeruser/.config/docker

# ══════════════════════════════════════════════════════════════
# PHASE 4: AS dockeruser — ZERO SUDO
# ══════════════════════════════════════════════════════════════
su - dockeruser
whoami   # must output: dockeruser

# Rootless setup — "systemd not detected" on EC2 is NORMAL
dockerd-rootless-setuptool.sh install

# Set environment variables (EC2-specific paths)
echo 'export XDG_RUNTIME_DIR=/home/dockeruser/.docker/run' >> ~/.bashrc
echo 'export PATH=/usr/bin:/sbin:/usr/sbin:$PATH' >> ~/.bashrc
echo 'export DOCKER_HOST=unix:///home/dockeruser/.docker/run/docker.sock' >> ~/.bashrc
source ~/.bashrc

# Create socket directory
mkdir -p /home/dockeruser/.docker/run

# ══════════════════════════════════════════════════════════════
# PHASE 5: BACK TO ADMIN — System Service + cgroup Config
# ══════════════════════════════════════════════════════════════
exit   # back to admin

# Create system service file
sudo nano /etc/systemd/system/docker-rootless-dockeruser.service
# (paste the [Unit][Service][Install] block from Phase 5 Step 14)

# Create cgroup delegation config
sudo mkdir -p /etc/systemd/system/user@.service.d/
sudo nano /etc/systemd/system/user@.service.d/delegate.conf
# (paste the [Service] Delegate= block from Phase 5 Step 15)

# Reload systemd, enable and start
sudo systemctl daemon-reload
sudo systemctl enable docker-rootless-dockeruser
sudo systemctl start docker-rootless-dockeruser
sudo systemctl status docker-rootless-dockeruser   # verify: active (running)
ps aux | grep dockerd                              # must show dockeruser, NOT root

# ══════════════════════════════════════════════════════════════
# PHASE 6: FINAL VERIFICATION — AS dockeruser
# ══════════════════════════════════════════════════════════════
su - dockeruser

docker run hello-world                # must print "Hello from Docker!"
docker info | grep -i rootless        # must show: Security Options: rootless

# Logout survival test
docker run -d --name test-survivor nginx
docker ps                             # running
exit                                  # log out completely
su - dockeruser
docker ps                             # STILL running after logout ✅
```

---

## ⚠️ Limitations of Rootless Docker on EC2

| Feature | Status | Notes |
|---|---|---|
| Ports < 1024 (80, 443) | ❌ Blocked by default | Admin applies `sysctl` fix (Bonus section) |
| `--privileged` containers | ❌ Not supported | Use regular Docker if truly needed |
| `systemctl --user` | ❌ No user systemd on EC2 | System service handles this (Phase 5) |
| Overlay networking | ⚠️ Limited | `slirp4netns` installed in Step 9 |
| Docker Compose | ✅ Works | No changes needed |
| Volume mounts | ✅ Works | No changes needed |
| Pulling images from Docker Hub | ✅ Works | No changes needed |
| Survives user logout | ✅ Works | System service (Phase 5) |
| Survives EC2 reboot | ✅ Works | `systemctl enable` (Step 16) |
| Auto-restarts on crash | ✅ Works | `Restart=always` in service file |
| cgroup isolation per container | ✅ Works | `cgroupfs` driver + `Delegate=yes` + delegate.conf |

---

> 📝 **Personal Reference Guide**
> Created: March 2026
> Platform: AWS EC2 — Ubuntu 24.04 LTS (Noble Numbat)
> Docker Version: 27.x (Rootless)
> ✅ dockeruser has zero sudo privileges
> ✅ Docker daemon runs as system service — survives logout and reboots
> ✅ cgroupfs driver — containers run without cgroup errors
> ✅ cgroup delegation configured — full container isolation works
> ✅ Battle-tested on EC2 — every issue encountered and fixed