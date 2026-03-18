# 🐧 Ubuntu 24.04 (AWS EC2) → Create User → Docker Rootless Setup
### A Complete Personal Guide — From Fresh Instance to Running Containers

---

## 🧠 The Core Concept — Why NO Sudo for dockeruser?

The entire point of rootless Docker is **security isolation**:

> If `dockeruser` has `sudo`, they can become root anytime.
> That completely defeats the purpose of rootless mode.

So the rule is simple:

| Who | What They Do | Has Sudo? |
|---|---|---|
| **Admin/root** | System setup, install Docker, create user, create system service | ✅ Yes |
| **dockeruser** | Runs Docker, runs containers — nothing else | ❌ Never |

---

## ⚠️ EC2 vs Regular VM — Important Difference

On a regular Ubuntu VM or desktop, Docker rootless uses **systemd user sessions** to manage the daemon automatically.

**On AWS EC2, systemd user sessions are NOT available for non-root users.** So when you run `dockerd-rootless-setuptool.sh install` on EC2 you will see:

```
[INFO] systemd not detected, dockerd-rootless.sh needs to be started manually:
PATH=/usr/bin:/sbin:/usr/sbin:$PATH dockerd-rootless.sh
```

This is **not an error** — it's just telling you systemd user mode isn't available.

The fix: **Admin creates a system-level service** (using system systemd, which IS available) that runs `dockerd-rootless.sh` as `dockeruser`. This way:
- Docker starts automatically on boot
- Docker keeps running even when `dockeruser` is logged out
- If Docker crashes, it auto-restarts
- `dockeruser` still has zero sudo

| | Regular VM | AWS EC2 |
|---|---|---|
| Systemd user session | ✅ Available | ❌ Not available |
| `systemctl --user` | ✅ Works | ❌ Doesn't work |
| Solution | `systemctl --user enable docker` | Admin creates system service |
| Docker socket path | `/run/user/1001/docker.sock` | `/home/dockeruser/.docker/run/docker.sock` |
| App survives logout | ✅ Yes (with linger) | ✅ Yes (with system service) |

---

## 🗺️ The Full Roadmap

```
Fresh Ubuntu 24.04 EC2 Instance
        ↓
[ADMIN] Update the System
        ↓
[ADMIN] Install Essential Tools
        ↓
[ADMIN] Create dockeruser  ← NO sudo, ever
        ↓
[ADMIN] Install Docker Engine
        ↓
[ADMIN] Install docker-ce-rootless-extras  ← where the setup script lives
        ↓
[ADMIN] Install Rootless Dependencies (uidmap, slirp4netns, etc.)
        ↓
[SWITCH] su - dockeruser
        ↓
[dockeruser] Run dockerd-rootless-setuptool.sh install
        ↓
[dockeruser] Set Environment Variables in ~/.bashrc
        ↓
[SWITCH BACK TO ADMIN]
        ↓
[ADMIN] Create system-level service for dockerd-rootless
        ↓
[ADMIN] Enable + Start the service
        ↓
[dockeruser] Test & Verify ✅
```

---

## 🖥️ Phase 1 — System Setup (As Admin)

### Step 1 — First Login & Update Everything

When you first log in to your EC2 instance (usually as `ubuntu` user), always update first:

```bash
sudo apt-get update && sudo apt-get upgrade -y
```

**What this does:**
- `apt-get update` — Refreshes the list of available packages
- `apt-get upgrade -y` — Installs all updates automatically
- Always do this on a fresh instance before installing anything

**Proof it worked:** No errors in output. Ends with something like:
```
0 upgraded, 0 newly installed, 0 to remove and 0 not upgraded.
```

---

### Step 2 — Install Essential Tools

```bash
sudo apt-get install -y curl wget git nano
```

| Tool | Why You Need It |
|---|---|
| `curl` | Downloads files/scripts from the internet |
| `wget` | Another download tool (good backup) |
| `git` | Version control |
| `nano` | Simple terminal text editor |

---

## 👤 Phase 2 — Create dockeruser (As Admin — NO Sudo Given!)

### Step 3 — Create the New User

```bash
sudo adduser dockeruser
```

> 💡 Replace `dockeruser` with whatever username you want.

Ubuntu will ask you a few questions:

```
Adding user `dockeruser' ...
Adding new group `dockeruser' (1001) ...
Adding new home directory `/home/dockeruser' ...

New password:          ← type a strong password
Retype new password:   ← confirm it

Full Name []:          ← press Enter to skip
Room Number []:        ← press Enter
Work Phone []:         ← press Enter
Home Phone []:         ← press Enter
Other []:              ← press Enter

Is the information correct? [Y/n]  ← type Y and Enter
```

**What `adduser` does automatically:**
- Creates the user account
- Creates their home folder at `/home/dockeruser`
- Creates a group with the same name
- Sets up their shell as `/bin/bash`

---

### Step 4 — ❌ Do NOT Give Sudo Privileges

```bash
# ❌ NEVER run this for dockeruser:
# sudo usermod -aG sudo dockeruser   ← DO NOT DO THIS

# ✅ This user must have ZERO sudo — that's the whole point of rootless Docker
```

If `dockeruser` could run `sudo`, they could just `sudo su` and become root instantly. The entire security boundary would be pointless.

---

### Step 5 — Optionally Add to systemd-journal Group

```bash
sudo usermod -aG systemd-journal dockeruser
```

This lets `dockeruser` read system logs — helpful for debugging Docker later. This does NOT grant any elevated privileges.

---

### Step 6 — Verify the User Has No Sudo

```bash
groups dockeruser
```

Expected output:
```
dockeruser : dockeruser systemd-journal
```

✅ You should **NOT** see `sudo` in this list. If you do, you made a mistake in Step 4.

---

## 🐳 Phase 3 — Install Docker Engine (Admin, One Time)

### Step 7 — Download and Run Docker's Official Installer

```bash
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
```

**This installs:**
- `docker-ce` — Docker Community Edition (the engine)
- `docker-ce-cli` — The `docker` command-line tool
- `containerd.io` — The container runtime
- `docker-compose-plugin` — Docker Compose (built-in now)

**Proof it worked:**
```bash
docker --version
```
Expected output:
```
Docker version 27.x.x, build xxxxxxx
```

---

### Step 8 — Install docker-ce-rootless-extras ← Critical, Don't Skip!

The `dockerd-rootless-setuptool.sh` and `dockerd-rootless.sh` scripts live inside this package. Without it, rootless mode simply doesn't exist:

```bash
sudo apt-get install -y docker-ce-rootless-extras
```

**Proof it worked:**
```bash
which dockerd-rootless-setuptool.sh
which dockerd-rootless.sh
```
Expected output:
```
/usr/bin/dockerd-rootless-setuptool.sh
/usr/bin/dockerd-rootless.sh
```

---

### Step 9 — Install All Rootless Dependencies (Admin Must Do This!)

```bash
sudo apt-get install -y uidmap dbus-user-session slirp4netns
```

> ⚠️ **MUST be done by admin BEFORE switching to dockeruser.**
> `dockeruser` has no sudo and cannot install packages themselves.

| Package | Purpose |
|---|---|
| `uidmap` | Maps user IDs — lets Docker create isolated user namespaces without root |
| `dbus-user-session` | Lets a non-root user run background services |
| `slirp4netns` | User-space networking for rootless containers — needed for internet access inside containers |

---

## 🔐 Phase 4 — Setup Docker Rootless (As dockeruser — Zero Sudo!)

Switch to the new user. **From this point forward, NO sudo is used at all.**

```bash
su - dockeruser
```

**Proof you switched:**
```bash
whoami
```
Output:
```
dockeruser
```

---

### Step 10 — Run the Rootless Setup Script

```bash
dockerd-rootless-setuptool.sh install
```

> ✅ No `sudo`. No root. Run it directly as `dockeruser`.

**On EC2, the expected output will be:**
```
[INFO] systemd not detected, dockerd-rootless.sh needs to be started manually:
PATH=/usr/bin:/sbin:/usr/sbin:$PATH dockerd-rootless.sh
[INFO] CLI context "rootless" already exists
[INFO] Using CLI context "rootless"
Current context is now "rootless"
[INFO] Make sure the following environment variable(s) are set (or add them to ~/.bashrc):
# WARNING: systemd not found. You have to remove XDG_RUNTIME_DIR manually on every logout.
export XDG_RUNTIME_DIR=/home/dockeruser/.docker/run
export PATH=/usr/bin:$PATH
[INFO] Some applications may require the following environment variable too:
export DOCKER_HOST=unix:///home/dockeruser/.docker/run/docker.sock
```

✅ This is **correct and expected on EC2** — not an error. The "systemd not detected" message just means we'll use the system-level service approach (done by admin in Phase 5).

---

### Step 11 — Set Environment Variables

Based on exactly what the setup script told you, add these to `~/.bashrc`:

```bash
echo 'export XDG_RUNTIME_DIR=/home/dockeruser/.docker/run' >> ~/.bashrc
echo 'export PATH=/usr/bin:/sbin:/usr/sbin:$PATH' >> ~/.bashrc
echo 'export DOCKER_HOST=unix:///home/dockeruser/.docker/run/docker.sock' >> ~/.bashrc
source ~/.bashrc
```

**What each variable does:**
| Variable | Purpose |
|---|---|
| `XDG_RUNTIME_DIR` | Tells Docker where to create its socket and runtime files |
| `PATH` | Makes sure all Docker binaries are findable |
| `DOCKER_HOST` | Tells the Docker CLI exactly where the daemon socket is |

**Proof they're set:**
```bash
echo $XDG_RUNTIME_DIR
echo $DOCKER_HOST
```
Expected output:
```
/home/dockeruser/.docker/run
unix:///home/dockeruser/.docker/run/docker.sock
```

---

### Step 12 — Create the Runtime Directory

```bash
mkdir -p /home/dockeruser/.docker/run
```

This is where the Docker socket will live. The system service will also create this automatically, but good to have it ready.

---

## 🛠️ Phase 5 — Create System Service (Back to Admin!)

Switch back to your admin user. This is the key phase that makes Docker survive logouts and reboots.

```bash
exit   # or open a new terminal as admin
```

### Step 13 — Why Not Just Use `.bashrc` to Start Docker?

You might think: just add `dockerd-rootless.sh &` to `~/.bashrc` and be done.

**The problem:**

```
dockeruser logs in → .bashrc starts dockerd → app runs
dockeruser logs OUT → session ends → dockerd dies → app CRASHES 💥
EC2 reboots → dockerd never starts → app never runs 💥
```

**The solution:** A proper **system-level** service that:
- Starts at boot, independently of any login
- Runs `dockerd-rootless.sh` as `dockeruser` (still rootless!)
- Keeps Docker alive even when `dockeruser` is completely logged out
- Auto-restarts if Docker ever crashes

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

[Install]
WantedBy=multi-user.target
```

**What each section means:**
| Setting | What It Does |
|---|---|
| `User=dockeruser` | Runs the daemon as `dockeruser` — NOT as root |
| `ExecStartPre=mkdir` | Creates the socket directory before starting |
| `ExecStart=dockerd-rootless.sh` | The actual rootless Docker daemon |
| `Restart=always` | Auto-restarts if Docker crashes |
| `RestartSec=5` | Waits 5 seconds before restarting |
| `WantedBy=multi-user.target` | Starts at boot, at the normal runlevel |

Save and exit: `Ctrl+O` → `Enter` → `Ctrl+X`

---

### Step 15 — Enable and Start the Service

```bash
# Reload systemd so it knows about the new service
sudo systemctl daemon-reload

# Enable it — auto-starts on every boot
sudo systemctl enable docker-rootless-dockeruser

# Start it right now
sudo systemctl start docker-rootless-dockeruser
```

---

### Step 16 — Verify the Service Is Running

```bash
sudo systemctl status docker-rootless-dockeruser
```

Expected output:
```
● docker-rootless-dockeruser.service - Docker Rootless Daemon (dockeruser)
     Loaded: loaded (/etc/systemd/system/docker-rootless-dockeruser.service)
     Active: active (running) since ...
   Main PID: XXXX (dockerd-rootless.)
```

Also verify the process is owned by `dockeruser`, NOT root:

```bash
ps aux | grep dockerd
```

You should see `dockerd` listed under `dockeruser`. ✅

---

## ✅ Phase 6 — Final Testing & Proof (As dockeruser)

Switch back to `dockeruser`:

```bash
su - dockeruser
```

### Step 17 — Run the Hello World Container

```bash
docker run hello-world
```

**Expected output:**
```
Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
 3. The Docker daemon created a new container from that image...
 4. The Docker daemon streamed that output to the Docker client...
```

🎉 **If you see this — rootless Docker is fully working!**

---

### Step 18 — Confirm It's Actually Rootless

```bash
docker info | grep -i rootless
```

Expected output:
```
 rootlesskit
 Security Options: rootless
```

---

### Step 19 — Prove the App Survives Logout

This is the ultimate test. Start a container in the background:

```bash
docker run -d --name test-survivor nginx
docker ps   # confirm it's running
```

Now log out completely:

```bash
exit   # log out of dockeruser session
```

Wait a few seconds, then log back in as `dockeruser`:

```bash
su - dockeruser
docker ps   # container is STILL running ✅
```

The container survived the logout because Docker is now managed by the system service, not by the user session. ✅

---

### Step 20 — Run a Real Container (Extra Proof)

```bash
docker run -it ubuntu:24.04 bash
```

Inside the container:

```bash
whoami               # shows 'root' INSIDE container only — NOT on host
cat /etc/os-release  # shows Ubuntu container info
exit                 # leave the container
```

> 💡 Even though `whoami` shows `root` inside the container, on the **host machine** the process is running as `dockeruser`. That's rootless working correctly.

---

## 🔧 Bonus — Fix Low Ports (80, 443) — Admin Must Do This

Since `dockeruser` has no sudo, the admin applies this fix:

```bash
# Temporary fix (resets on reboot)
sudo sysctl net.ipv4.ip_unprivileged_port_start=80

# Permanent fix (survives reboots)
echo "net.ipv4.ip_unprivileged_port_start=80" | sudo tee /etc/sysctl.d/99-rootless-docker.conf
sudo sysctl --system
```

After this, `dockeruser` can bind to ports 80, 443, etc. in their containers.

---

## 📋 Complete Summary — Every Command in Order

```bash
# ══════════════════════════════════════════════════════════════
# PHASE 1–3: AS ADMIN
# ══════════════════════════════════════════════════════════════

# System setup
sudo apt-get update && sudo apt-get upgrade -y
sudo apt-get install -y curl wget git nano

# Create user — ZERO sudo, ever
sudo adduser dockeruser
sudo usermod -aG systemd-journal dockeruser   # optional

# Verify — must NOT show sudo
groups dockeruser   # expected: dockeruser systemd-journal

# Install Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
docker --version

# Install rootless packages
sudo apt-get install -y docker-ce-rootless-extras
sudo apt-get install -y uidmap dbus-user-session slirp4netns

# ══════════════════════════════════════════════════════════════
# PHASE 4: AS dockeruser — ZERO SUDO
# ══════════════════════════════════════════════════════════════
su - dockeruser
whoami   # must output: dockeruser

# Run rootless setup
dockerd-rootless-setuptool.sh install
# On EC2 you will see "systemd not detected" — this is NORMAL

# Set environment variables (EC2 paths)
echo 'export XDG_RUNTIME_DIR=/home/dockeruser/.docker/run' >> ~/.bashrc
echo 'export PATH=/usr/bin:/sbin:/usr/sbin:$PATH' >> ~/.bashrc
echo 'export DOCKER_HOST=unix:///home/dockeruser/.docker/run/docker.sock' >> ~/.bashrc
source ~/.bashrc

# Create socket directory
mkdir -p /home/dockeruser/.docker/run

# ══════════════════════════════════════════════════════════════
# PHASE 5: BACK TO ADMIN — Create System Service
# ══════════════════════════════════════════════════════════════
exit   # back to admin

sudo nano /etc/systemd/system/docker-rootless-dockeruser.service
# Paste the service file content (see Phase 5, Step 14)

sudo systemctl daemon-reload
sudo systemctl enable docker-rootless-dockeruser
sudo systemctl start docker-rootless-dockeruser
sudo systemctl status docker-rootless-dockeruser   # verify: active (running)

# ══════════════════════════════════════════════════════════════
# PHASE 6: FINAL VERIFICATION AS dockeruser
# ══════════════════════════════════════════════════════════════
su - dockeruser
docker run hello-world
docker info | grep -i rootless
ps aux | grep dockerd   # must show dockeruser, NOT root

# Logout test
docker run -d --name test-survivor nginx
docker ps           # running
exit                # log out
su - dockeruser
docker ps           # still running after logout ✅
```

---

## ⚠️ Limitations of Rootless Docker

| Feature | Status | Fix/Workaround |
|---|---|---|
| Ports < 1024 (80, 443) | ❌ Blocked by default | Admin applies `sysctl` fix (Bonus section) |
| `--privileged` containers | ❌ Not supported | Use regular Docker if needed |
| `systemctl --user` on EC2 | ❌ No user systemd | Use system service (Phase 5) |
| Overlay networking | ⚠️ Limited | `slirp4netns` installed in Step 9 |
| Docker Compose | ✅ Works | No changes needed |
| Volume mounts | ✅ Works | No changes needed |
| Docker Hub / pulling images | ✅ Works | No changes needed |
| Survives logout | ✅ Works | System service handles it (Phase 5) |
| Survives reboot | ✅ Works | `systemctl enable` in Step 15 |
| Auto-restarts on crash | ✅ Works | `Restart=always` in service file |

---

> 📝 **Personal Reference Guide**  
> Created: March 2026  
> Platform: AWS EC2 — Ubuntu 24.04 LTS (Noble Numbat)  
> Docker Version: 27.x (Rootless)  
> ✅ dockeruser has zero sudo privileges  
> ✅ Docker survives logout and reboots via system service  
> ✅ EC2 / no-systemd-user-session fully handled