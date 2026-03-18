# 🐧 Ubuntu 24.04 VM → Create User → Docker Rootless Setup
### A Complete Personal Guide — From Fresh VM to Running Containers

---

## 🧠 The Core Concept — Why NO Sudo for dockeruser?

The entire point of rootless Docker is **security isolation**:

> If `dockeruser` has `sudo`, they can become root anytime.
> That completely defeats the purpose of rootless mode.

So the rule is simple:

| Who | What They Do | Has Sudo? |
|---|---|---|
| **Admin/root** | System setup, install Docker engine, create user, prep dependencies | ✅ Yes |
| **dockeruser** | Runs Docker, runs containers — nothing else | ❌ Never |

---

## 🗺️ The Full Roadmap

```
Fresh Ubuntu 24.04 VM
        ↓
[ADMIN] Update the System
        ↓
[ADMIN] Install Essential Tools
        ↓
[ADMIN] Create dockeruser  ← NO sudo given to this user
        ↓
[ADMIN] Install Docker Engine
        ↓
[ADMIN] Install docker-ce-rootless-extras
        ↓
[ADMIN] Install Rootless Dependencies  ← must be done before switching user
        ↓
[ADMIN] Enable Linger for dockeruser   ← requires root, do it now
        ↓
[SWITCH] su - dockeruser
        ↓
[dockeruser] Setup Docker Rootless
        ↓
[dockeruser] Set Environment Variables
        ↓
[dockeruser] Test & Verify ✅
```

---

## 🖥️ Phase 1 — System Setup (As Admin)

### Step 1 — First Login & Update Everything

When you first log in to your Ubuntu VM (usually as `root` or a default admin user), always update first:

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

## 👤 Phase 2 — Create dockeruser (As Admin — NO Sudo Given!)

### Step 3 — Create the New User

```bash
sudo adduser dockeruser
```

> 💡 Replace `dockeruser` with whatever username you want — `devuser`, `john`, etc.

**What happens:** Ubuntu will ask you a few questions:

```
Adding user `dockeruser' ...
Adding new group `dockeruser' (1001) ...
Adding new home directory `/home/dockeruser' ...

New password:          ← type a strong password
Retype new password:   ← confirm it

Full Name []:     ← optional, press Enter to skip
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

This lets `dockeruser` read system logs — helpful for debugging Docker later. This is the **only** group addition needed, and it does NOT grant any elevated privileges.

---

### Step 6 — Verify the User Has No Sudo

```bash
groups dockeruser
```

Expected output:
```
dockeruser : dockeruser systemd-journal
```

✅ You should **NOT** see `sudo` in this list. If you do — you made a mistake in Step 4.

---

## 🐳 Phase 3 — Install Docker Engine (Admin, One Time)

### Step 7 — Download and Run Docker's Official Installer

Still as your admin user, install Docker Engine:

```bash
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
```

**What the flags mean:**
- `-f` — Fail silently on HTTP errors
- `-s` — Silent mode
- `-S` — Still show errors if they happen
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

### Step 8 — Install docker-ce-rootless-extras ← Critical Step!

This package is **required and often missed**. The `dockerd-rootless-setuptool.sh` script — which actually sets up rootless mode — lives inside this package:

```bash
sudo apt-get install -y docker-ce-rootless-extras
```

**Proof it worked:**
```bash
which dockerd-rootless-setuptool.sh
```
Expected output:
```
/usr/bin/dockerd-rootless-setuptool.sh
```

If this returns nothing, the rootless setup in Phase 4 will fail with "command not found". ✅

---

### Step 9 — Install All Rootless Dependencies (Admin Must Do This!)

```bash
sudo apt-get install -y uidmap dbus-user-session slirp4netns
```

> ⚠️ **This MUST be done by admin BEFORE switching to dockeruser.**
> `dockeruser` has no sudo and cannot install packages themselves.

**What each package does:**
| Package | Purpose |
|---|---|
| `uidmap` | Maps user IDs — lets Docker create isolated user namespaces without root |
| `dbus-user-session` | Lets a non-root user run background services like the Docker daemon |
| `slirp4netns` | Provides user-space networking for rootless containers — needed for internet access inside containers |

---

### Step 10 — Enable Linger for dockeruser (Admin Must Do This!)

```bash
sudo loginctl enable-linger dockeruser
```

> ⚠️ **This also MUST be done by admin.** It requires root privileges and `dockeruser` can't run it themselves.

**What this does:** Allows `dockeruser`'s systemd services (like the Docker daemon) to keep running even when they're not actively logged in. Without this, Docker shuts down the moment `dockeruser` logs out.

**Proof it worked:**
```bash
loginctl show-user dockeruser | grep Linger
```
Expected output:
```
Linger=yes
```

---

## 🔐 Phase 4 — Setup Docker Rootless (As dockeruser — Zero Sudo!)

Now switch to the new user. **From this point forward, NO sudo is used at all.**

```bash
su - dockeruser
```

The `-` means it's a **full login switch** — loads the user's environment and starts from their home directory.

**Proof you switched:**
```bash
whoami
```
Output:
```
dockeruser
```

---

### Step 11 — Run the Rootless Setup Script

```bash
dockerd-rootless-setuptool.sh install
```

> ✅ No `sudo`. No root. Just run it directly as `dockeruser`.

**What this does:**
- Sets up a private Docker daemon that belongs only to `dockeruser`
- Creates a user-level systemd service at `~/.config/systemd/user/docker.service`
- Configures user namespaces using `uidmap`

**Proof it worked — expected output:**
```
[INFO] Creating /home/dockeruser/.config/systemd/user/docker.service
[INFO] starting systemd service docker.service
+ systemctl --user start docker.service
+ systemctl --user enable docker.service
[INFO] Docker is now available as a rootless daemon.
```

**If you see an error about `newuidmap` or `newgidmap`:**
Switch back to admin and run:
```bash
sudo apt-get install -y uidmap
```
Then switch back to `dockeruser` and retry.

---

### Step 12 — Set Environment Variables

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
The number `1001` is `dockeruser`'s UID — it may differ on your system.

---

### Step 13 — Enable Docker to Auto-Start on Login

```bash
systemctl --user enable docker
```

> ✅ The `--user` flag means this operates only on `dockeruser`'s own services. No sudo needed.

**What this does:** Auto-starts the rootless Docker daemon every time `dockeruser` logs in.

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

## ✅ Phase 5 — Final Testing & Proof (Still as dockeruser)

### Step 14 — Run the Hello World Container

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

### Step 15 — Confirm It's Actually Running Rootless

```bash
docker info | grep -i rootless
```

Expected output:
```
 rootlesskit
 Security Options: rootless
```

Confirm the daemon is NOT running as root:

```bash
ps aux | grep dockerd
```

You should see `dockerd` running under **`dockeruser`**, NOT `root`. ✅

---

### Step 16 — Run a Real Container (Extra Proof)

```bash
docker run -it ubuntu:24.04 bash
```

You're now inside an Ubuntu container! Try:

```bash
whoami               # shows 'root' INSIDE the container only — NOT on the host
cat /etc/os-release  # shows Ubuntu container info
exit                 # leave the container
```

> 💡 Even though `whoami` shows `root` inside the container, on the **host machine** the process is running as `dockeruser`. That's rootless working correctly.

---

## 🔧 Bonus — Fix Low Ports (80, 443) — Admin Must Do This

Since `dockeruser` has no sudo, the admin needs to apply this fix.
Switch back to your admin user first, then run:

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
# PHASE 1–3: RUN AS ADMIN (the user with sudo)
# ══════════════════════════════════════════════════════════════

# System setup
sudo apt-get update && sudo apt-get upgrade -y
sudo apt-get install -y curl wget git nano

# Create user — NO SUDO GROUP, ever
sudo adduser dockeruser
sudo usermod -aG systemd-journal dockeruser   # optional, logs only

# Verify: must NOT show sudo
groups dockeruser   # expected: dockeruser systemd-journal

# Install Docker Engine
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
docker --version    # verify

# Install rootless extras + all dependencies
sudo apt-get install -y docker-ce-rootless-extras
sudo apt-get install -y uidmap dbus-user-session slirp4netns

# Enable linger — must be done as admin
sudo loginctl enable-linger dockeruser
loginctl show-user dockeruser | grep Linger   # should show: Linger=yes

# ══════════════════════════════════════════════════════════════
# PHASE 4–5: SWITCH TO dockeruser — ZERO SUDO FROM HERE ON
# ══════════════════════════════════════════════════════════════
su - dockeruser
whoami   # must output: dockeruser

# Rootless setup — NO sudo at all!
dockerd-rootless-setuptool.sh install

echo 'export PATH=/usr/bin:$PATH' >> ~/.bashrc
echo 'export DOCKER_HOST=unix://$XDG_RUNTIME_DIR/docker.sock' >> ~/.bashrc
source ~/.bashrc

systemctl --user enable docker
systemctl --user status docker   # verify: active (running)

# Final verification
docker run hello-world
docker info | grep -i rootless
ps aux | grep dockerd   # must show dockeruser, NOT root
```

---

## ⚠️ Limitations of Rootless Docker

| Feature | Status | Fix/Workaround |
|---|---|---|
| Ports < 1024 (80, 443) | ❌ Blocked by default | Admin applies `sysctl` fix (Bonus section) |
| `--privileged` containers | ❌ Not supported | Use regular Docker if needed |
| Overlay networking | ⚠️ Limited | `slirp4netns` already installed in Step 9 |
| Docker Compose | ✅ Works perfectly | No changes needed |
| Volume mounts | ✅ Works | No changes needed |
| Docker Hub / pulling images | ✅ Works | No changes needed |
| Auto-start on login | ✅ Works | Linger enabled by admin in Step 10 |

---

> 📝 **Personal Reference Guide**  
> Created: March 2026  
> Ubuntu Version: 24.04 LTS (Noble Numbat)  
> Docker Version: 27.x (Rootless)  
> ✅ Fully corrected — dockeruser has zero sudo privileges