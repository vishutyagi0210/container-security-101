# 🐳 Installing Docker in Rootless Mode — The Right Way

> **Who is this for?** Anyone who wants to run Docker securely on Linux without giving it root/superuser power over your whole system.  
> **Why rootless?** If something goes wrong inside a container, the attacker is stuck as a regular user — not as `root`. That's a huge security win.

---

## 🧠 What Is "Rootless Docker"?

Normally, Docker runs as **root** (the most powerful user on Linux). That means:
- Docker can read/write *any* file on your system
- A compromised container = game over for your machine

**Rootless mode** flips this. The Docker daemon and all containers run as *your* normal user account. No root. No risk of full system takeover.

Think of it like this:
> Regular Docker = giving a stranger the master key to your house.  
> Rootless Docker = giving them a key that only opens their own room.

---

## ✅ Prerequisites

Before starting, make sure you have:

| Requirement | Why |
|---|---|
| Ubuntu 20.04+ / Debian 10+ / Fedora 34+ | Rootless works best on modern distros |
| A **non-root** user account | You'll run everything as this user |
| `curl` installed | To download Docker's setup script |
| Internet access | To pull packages |

---

## 🛠️ Step-by-Step Installation

### Step 1 — Update Your System

Always start fresh. Run this as your normal user (it will ask for your password):

```bash
sudo apt-get update && sudo apt-get upgrade -y
```

**What this does:** Makes sure all existing packages are up to date before we add Docker on top.

---

### Step 2 — Install Docker Engine (One-Time Root Step)

This is the *only* step that needs root. We download and run Docker's official install script:

```bash
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
```

**What this does:**
- Downloads Docker's official installer
- Installs the Docker Engine on your system
- This is a one-time setup — you won't need root again after this

**Proof it worked:**
```bash
docker --version
```
Expected output:
```
Docker version 26.x.x, build xxxxxxx
```

---

### Step 3 — Install Rootless Dependencies

```bash
sudo apt-get install -y uidmap dbus-user-session
```

**What this does:**
- `uidmap` — Allows mapping user IDs so your user can "pretend" to be multiple users inside containers (needed for isolation)
- `dbus-user-session` — Lets your user manage background services (like the Docker daemon) without root

---

### Step 4 — Run the Rootless Setup Script (As Normal User — No sudo!)

```bash
dockerd-rootless-setuptool.sh install
```

> ⚠️ **Important:** Do NOT use `sudo` here. Run this as your regular user.

**What this does:** Configures Docker to run entirely under your user account. It sets up a user-level Docker daemon that starts independently from the system-level one.

**Proof it worked — you should see something like:**
```
[INFO] Creating /home/youruser/.config/systemd/user/docker.service
[INFO] starting systemd service docker.service
+ systemctl --user start docker.service
+ systemctl --user enable docker.service
[INFO] Docker is now available as a rootless daemon.
```

---

### Step 5 — Set Environment Variables

You need to tell your terminal *where* to find the rootless Docker socket. Add these lines to your shell config file.

**For Bash users:**
```bash
echo 'export PATH=/usr/bin:$PATH' >> ~/.bashrc
echo 'export DOCKER_HOST=unix://$XDG_RUNTIME_DIR/docker.sock' >> ~/.bashrc
source ~/.bashrc
```

**For Zsh users:**
```bash
echo 'export PATH=/usr/bin:$PATH' >> ~/.zshrc
echo 'export DOCKER_HOST=unix://$XDG_RUNTIME_DIR/docker.sock' >> ~/.zshrc
source ~/.zshrc
```

**What this does:**
- `PATH` — Makes sure your terminal finds the right Docker binary
- `DOCKER_HOST` — Points Docker CLI to your *user-level* daemon socket instead of the system root socket

---

### Step 6 — Enable Docker to Auto-Start on Login

```bash
systemctl --user enable docker
sudo loginctl enable-linger $(whoami)
```

**What this does:**
- `systemctl --user enable docker` — Starts Docker automatically when *you* log in
- `loginctl enable-linger` — Keeps your user services (like Docker) running even when you're not actively logged in (important for servers)

---

### Step 7 — ✅ Final Proof: Run a Test Container

This is the ultimate test. Run:

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
 3. The Docker daemon created a new container from that image which runs the
    executable that produces this output.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.
```

🎉 **If you see that — you're done. Docker rootless is fully working.**

---

### Step 8 — Confirm It's Actually Rootless (Extra Proof)

Want to be 100% sure it's not running as root? Run:

```bash
docker info | grep -i rootless
```

Expected output:
```
 rootlesskit
 Security Options: rootless
```

Or check the running process:
```bash
ps aux | grep dockerd
```

You should see `dockerd` running under **your username**, not `root`.

---

## ⚠️ Known Limitations (Good to Know)

| Feature | Status | Workaround |
|---|---|---|
| Ports below 1024 (like 80, 443) | ❌ Blocked by default | See fix below |
| `--privileged` containers | ❌ Not supported | Use regular Docker if needed |
| Overlay networking | ⚠️ Limited | Use `slirp4netns` |
| Docker Compose | ✅ Fully works | No changes needed |
| Volume mounts | ✅ Works | No changes needed |
| Docker Hub pulls | ✅ Works | No changes needed |

---

## 🔧 Bonus: Fix Low Ports (80, 443)

If you need to bind to ports like 80 or 443, run:

```bash
# Temporary (resets on reboot)
sudo sysctl net.ipv4.ip_unprivileged_port_start=80

# Permanent (survives reboots)
echo "net.ipv4.ip_unprivileged_port_start=80" | sudo tee /etc/sysctl.d/99-rootless-docker.conf
sudo sysctl --system
```

---

## 🧹 How to Uninstall (If Needed)

```bash
dockerd-rootless-setuptool.sh uninstall
sudo apt-get remove docker-ce docker-ce-cli containerd.io
```

---

## 📋 Quick Reference Cheat Sheet

```bash
# Install Docker Engine
curl -fsSL https://get.docker.com -o get-docker.sh && sudo sh get-docker.sh

# Install dependencies
sudo apt-get install -y uidmap dbus-user-session

# Setup rootless (NO sudo)
dockerd-rootless-setuptool.sh install

# Set env vars (bash)
echo 'export DOCKER_HOST=unix://$XDG_RUNTIME_DIR/docker.sock' >> ~/.bashrc && source ~/.bashrc

# Enable on login
systemctl --user enable docker && sudo loginctl enable-linger $(whoami)

# Verify
docker run hello-world
docker info | grep -i rootless
```

---

> 📝 **Written for personal reference.**  
> Last updated: March 2026  
> Tested on: Ubuntu 22.04 LTS, Ubuntu 24.04 LTS, Debian 12