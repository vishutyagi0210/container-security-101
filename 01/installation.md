# 🐧 Ubuntu 24.04 VM → Create User → Docker Rootless Setup
### A Complete Personal Guide — From Fresh VM to Running Containers

---

## 🗺️ The Full Roadmap

```
Fresh Ubuntu 24.04 VM
        ↓
  Update the System
        ↓
  Create a New User
        ↓
  Give User Sudo Rights
        ↓
  Switch to That User
        ↓
  Install Docker Engine  ← (one-time root step)
        ↓
  Install Rootless Deps
        ↓
  Setup Docker Rootless  ← (as the new user!)
        ↓
  Test & Verify ✅
```

---

## 🖥️ Phase 1 — Fresh Ubuntu 24.04 VM Setup

### Step 1 — First Login & Update Everything

When you first log in to your Ubuntu VM (usually as `root` or a default user), always update first:

```bash
sudo apt-get update && sudo apt-get upgrade -y
```

**What this does:**
- `apt-get update` — Refreshes the list of available packages
- `apt-get upgrade -y` — Installs all updates automatically
- Always do this on a fresh VM before installing anything

**Proof it worked:** No errors in output. Ends with something like:
```
0 upgraded, 0 newly installed, 0 to remove and 0 not upgraded.
```

---

### Step 2 — Install Essential Tools

```bash
sudo apt-get install -y curl wget git nano
```

**What each tool is:**
| Tool | Why You Need It |
|---|---|
| `curl` | Downloads files/scripts from the internet |
| `wget` | Another download tool (good backup) |
| `git` | Version control (you'll need it) |
| `nano` | Simple terminal text editor |

---

## 👤 Phase 2 — Create a New User

### Step 3 — Create the New User

```bash
sudo adduser dockeruser
```

> 💡 Replace `dockeruser` with whatever username you want — `yourname`, `devuser`, `john`, etc.

**What happens:** Ubuntu will ask you a few questions:

```
Adding user `dockeruser' ...
Adding new group `dockeruser' (1001) ...
Adding new home directory `/home/dockeruser' ...

New password:          ← type a strong password
Retype new password:   ← confirm it

Full Name []:     ← optional, can press Enter to skip
Room Number []:   ← press Enter
Work Phone []:    ← press Enter
Home Phone []:    ← press Enter
Other []:         ← press Enter

Is the information correct? [Y/n]  ← type Y and Enter
```

**What `adduser` does automatically:**
- Creates the user account
- Creates their home folder at `/home/dockeruser`
- Creates a group with the same name
- Sets up their shell as `/bin/bash`

---

### Step 4 — Give the User Sudo Privileges

```bash
sudo usermod -aG sudo dockeruser
```

**What this does:**
- `usermod` — Modifies a user account
- `-aG sudo` — **Appends** the user to the `sudo` group (capital G = group)
- Without this, the user can't run `sudo` commands

**Proof it worked:**
```bash
groups dockeruser
```
Expected output:
```
dockeruser : dockeruser sudo
```
You should see `sudo` in the list. ✅

---

### Step 5 — Also Add User to `systemd-journal` Group (Optional but Useful)

```bash
sudo usermod -aG systemd-journal dockeruser
```

This lets your user read system logs — helpful for debugging Docker later.

---

### Step 6 — Switch to the New User

```bash
su - dockeruser
```

The `-` means it's a **full login switch** — loads all the user's environment variables and starts from their home directory.

**Proof you switched:**
```bash
whoami
```
Output should be:
```
dockeruser
```

And:
```bash
pwd
```
Output should be:
```
/home/dockeruser
```

---

## 🐳 Phase 3 — Install Docker (As Root, One Time)

### Step 7 — Install Docker Engine

Open a new terminal or switch back to your original user/root for this step:

```bash
su - root
# or open a new terminal tab and SSH in as your original user with sudo
```

Now install Docker:

```bash
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
```

**What the flags mean:**
- `-f` — Fail silently on HTTP errors (no junk output)
- `-s` — Silent mode
- `-S` — Show errors if they happen
- `-L` — Follow redirects

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

## 🔐 Phase 4 — Setup Docker Rootless (As Your New User!)

Now switch to your new user:

```bash
su - dockeruser
```

### Step 8 — Install Rootless Dependencies

```bash
sudo apt-get install -y uidmap dbus-user-session
```

**What these are:**
| Package | Purpose |
|---|---|
| `uidmap` | Maps user IDs — lets Docker create isolated user namespaces |
| `dbus-user-session` | Lets your user run background services (like the Docker daemon) |

---

### Step 9 — Run the Rootless Setup Script

```bash
dockerd-rootless-setuptool.sh install
```

> ⚠️ **NO `sudo` here!** Run this as `dockeruser` directly.

**What this does:**
- Sets up a private Docker daemon that belongs only to `dockeruser`
- Creates a systemd user service for Docker
- Configures user namespaces

**Proof it worked — expected output:**
```
[INFO] Creating /home/dockeruser/.config/systemd/user/docker.service
[INFO] starting systemd service docker.service
+ systemctl --user start docker.service
+ systemctl --user enable docker.service
[INFO] Docker is now available as a rootless daemon.
```

**If you see an error** about `newuidmap` or `newgidmap`:
```bash
sudo apt-get install -y uidmap
# then retry
dockerd-rootless-setuptool.sh install
```

---

### Step 10 — Set Environment Variables

```bash
echo 'export PATH=/usr/bin:$PATH' >> ~/.bashrc
echo 'export DOCKER_HOST=unix://$XDG_RUNTIME_DIR/docker.sock' >> ~/.bashrc
source ~/.bashrc
```

**What each line does:**
- `PATH` → Makes sure the terminal finds the right Docker binary
- `DOCKER_HOST` → Points Docker CLI to the **user-level** socket, not the system root socket
- `source ~/.bashrc` → Applies changes immediately without logging out

**Verify the variable is set:**
```bash
echo $DOCKER_HOST
```
Expected output:
```
unix:///run/user/1001/docker.sock
```
(The number `1001` is your user's ID — it may differ)

---

### Step 11 — Enable Docker to Start on Login

```bash
systemctl --user enable docker
sudo loginctl enable-linger dockeruser
```

**What this does:**
- `systemctl --user enable docker` → Auto-starts Docker when `dockeruser` logs in
- `loginctl enable-linger` → Keeps Docker running even when `dockeruser` isn't actively logged in (critical for servers)

**Proof:**
```bash
systemctl --user status docker
```
Expected output:
```
● docker.service - Docker Application Container Engine (Rootless)
     Loaded: loaded (/home/dockeruser/.config/systemd/user/docker.service)
     Active: active (running) since ...
```

---

## ✅ Phase 5 — Final Testing & Proof

### Step 12 — Run the Hello World Container

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

🎉 **If you see this — you're fully done!**

---

### Step 13 — Confirm It's Running Rootless

```bash
docker info | grep -i rootless
```

Expected output:
```
 rootlesskit
 Security Options: rootless
```

Also confirm the daemon is NOT running as root:

```bash
ps aux | grep dockerd
```

You should see `dockerd` running under **`dockeruser`**, NOT `root`. ✅

---

### Step 14 — Run a Real Container (Extra Proof)

```bash
docker run -it ubuntu:24.04 bash
```

You're now inside an Ubuntu container! Try:

```bash
whoami       # shows 'root' inside container (but NOT on host)
cat /etc/os-release   # shows Ubuntu info
exit         # leave the container
```

---

## 🔧 Bonus — Fix Low Ports (80, 443, etc.)

By default, non-root users can't bind to ports below 1024. Fix it:

```bash
# Temporary fix (resets on reboot)
sudo sysctl net.ipv4.ip_unprivileged_port_start=80

# Permanent fix (survives reboots)
echo "net.ipv4.ip_unprivileged_port_start=80" | sudo tee /etc/sysctl.d/99-rootless-docker.conf
sudo sysctl --system
```

---

## 📋 Complete Summary — Every Command in Order

```bash
# ── PHASE 1: System Setup ──────────────────────────────────
sudo apt-get update && sudo apt-get upgrade -y
sudo apt-get install -y curl wget git nano

# ── PHASE 2: Create User ───────────────────────────────────
sudo adduser dockeruser
sudo usermod -aG sudo dockeruser
sudo usermod -aG systemd-journal dockeruser

# Verify user
groups dockeruser         # should show: dockeruser sudo
su - dockeruser           # switch to new user
whoami                    # should output: dockeruser

# ── PHASE 3: Install Docker Engine (as root/sudo) ──────────
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
docker --version          # verify install

# ── PHASE 4: Rootless Setup (as dockeruser) ────────────────
sudo apt-get install -y uidmap dbus-user-session
dockerd-rootless-setuptool.sh install    # NO sudo!

echo 'export PATH=/usr/bin:$PATH' >> ~/.bashrc
echo 'export DOCKER_HOST=unix://$XDG_RUNTIME_DIR/docker.sock' >> ~/.bashrc
source ~/.bashrc

systemctl --user enable docker
sudo loginctl enable-linger dockeruser

# ── PHASE 5: Verify ────────────────────────────────────────
docker run hello-world
docker info | grep -i rootless
ps aux | grep dockerd     # should show dockeruser, NOT root
```

---

## ⚠️ Limitations of Rootless Docker

| Feature | Status | Fix/Workaround |
|---|---|---|
| Ports < 1024 (80, 443) | ❌ Blocked | Use `sysctl` fix above |
| `--privileged` mode | ❌ Not supported | Use regular Docker |
| Overlay networking | ⚠️ Limited | Use `slirp4netns` |
| Docker Compose | ✅ Works perfectly | No changes needed |
| Volume mounts | ✅ Works | No changes needed |
| Docker Hub / pulling images | ✅ Works | No changes needed |

---

> 📝 **Personal Reference Guide**  
> Created: March 2026  
> Ubuntu Version: 24.04 LTS (Noble Numbat)  
> Docker Version: 27.x (Rootless)