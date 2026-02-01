# My Linux Post-Install Routine

This isn't a "best practices" guide for enterprise production, it's just how I generally set up my personal Debian/Ubuntu servers to keep them low-maintenance.

**Heads up:** These settings are my personal preference. They prioritize convenience over strict control, which means they can have downsides. I've listed those risks below so you can decide if this fits your setup.

## 1. Disable IPv6
**Why I do it:** In my homelab, IPv6 usually just causes timeouts and DNS lag because my router/ISP setup isn't fully optimized for it. Turning it off forces everything to use good ol' IPv4.

**The Downside:** If you actually rely on IPv6 (or if your ISP uses CGNAT and requires IPv6 for direct access), this will break your connectivity.

Add the following to `/etc/sysctl.conf`:
```properties
net.ipv6.conf.all.disable_ipv6=1
net.ipv6.conf.default.disable_ipv6=1
net.ipv6.conf.lo.disable_ipv6=1
```
*Run `sysctl -p` to apply changes immediately.*

## 2. Update System & Install Essentials
**Why:** Before configuring services, the system must be up-to-date to ensure all security patches are applied. We also install `curl` (for downloading scripts) and `unattended-upgrades` (for automating future security updates).

```bash
apt update && apt upgrade -y
apt install curl unattended-upgrades -y
```

## 3. Configure Unattended Upgrades
**Why:** Automating updates ensures the server remains secure against vulnerabilities without manual intervention. The configuration below specifically handles:
*   **Kernel Cleanup:** Prevents the `/boot` partition from filling up with old kernel versions.
*   **Dependency Cleanup:** Removes unused packages to keep the system clean.
*   **Auto-Reboot:** Ensures kernel updates (which require a reboot) are actually applied.
*   **Reboot Time:** Schedules reboots for 05:00 AM to minimize disruption.

Edit `/etc/apt/apt.conf.d/50unattended-upgrades` and ensure the following settings are active:

```text
"origin=Debian,codename=${distro_codename}-updates";
Unattended-Upgrade::Remove-Unused-Kernel-Packages "true";
Unattended-Upgrade::Remove-New-Unused-Dependencies "true";
Unattended-Upgrade::Remove-Unused-Dependencies "true";
Unattended-Upgrade::Automatic-Reboot "true";
Unattended-Upgrade::Automatic-Reboot-WithUsers "true";
Unattended-Upgrade::Automatic-Reboot-Time "05:00";
```

## 4. Install Docker
**Why:** Docker allows applications to run in isolated containers, keeping the host OS clean and making dependency management easier. The convenience script detects the OS and installs the correct version automatically.

```bash
curl -fsSL https://get.docker.com -o get-docker.sh
sh get-docker.sh
systemctl enable docker
```

## 5. Install Watchtower
**Why:** Watchtower monitors running Docker containers and checks for updates to their base images. If an update is found, it gracefully restarts the container with the new image. This automates the maintenance of self-hosted services.

```bash
docker run --detach \
--name watchtower \
--restart always \
--volume /var/run/docker.sock:/var/run/docker.sock \
containrrr/watchtower
```
