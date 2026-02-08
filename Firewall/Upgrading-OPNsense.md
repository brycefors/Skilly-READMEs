# OPNsense Upgrade Best Practices Guide

Upgrading a firewall requires caution to ensure network continuity and security. OPNsense follows a rolling release cycle with two major releases per year (January `YY.1` and July `YY.7`) and frequent minor updates.

## 1. Universal Preparation (Do this for ALL upgrades)

Regardless of the version type, always perform these steps before clicking "Update".

### A. Backup Configuration
This is the most critical step. If the filesystem corrupts, you can restore a fresh install in minutes if you have this file.
*   **Navigate to:** `System` > `Configuration` > `Backups`.
*   **Action:** Download the `config.xml`.
*   **Tip:** If saving to a cloud drive or email, check "Encrypt this configuration file" to protect passwords and keys.

### B. Virtual Machine Snapshots
If your OPNsense is virtualized (Proxmox, ESXi, Hyper-V):
*   **Action:** Take a full VM snapshot before starting.
*   **Benefit:** This allows for an instant rollback if the update fails, far faster than reinstalling.

### C. Verify Console Access
Updates can occasionally break the WebGUI or SSH service.
*   **Action:** Ensure you have access to the physical VGA/HDMI monitor, Serial Console, or IPMI/iDRAC.
*   **Why:** You may need to assign interfaces or restore a previous configuration from the console menu if the network stack fails.

### D. Check Disk Space
*   **Action:** Ensure you have sufficient disk space (at least 2-3 GB free) for downloading packages and extracting the base system.

---

## 2. Minor Version Upgrades
**Example:** `26.1.1` to `26.1.2`
**Frequency:** Every 1-3 weeks.
**Risk Level:** Low.

Minor updates typically contain security patches, bug fixes, and minor feature additions.

### Best Practices
1.  **Read the Changelog:** In the Firmware page, click the "Click to view details" or read the release notes on the OPNsense forum. Look for specific notes about plugins you use (e.g., WireGuard, Unbound).
2.  **Use the GUI:** Minor updates are safe to perform via the WebGUI (`System` > `Firmware` > `Status`).
3.  **Check for Reboot Requirements:**
    *   The update screen will indicate if a reboot is required (usually for Kernel or Base system updates).
    *   If no reboot is required, services will simply restart. Be aware this drops active connections briefly.
4.  **Audit Services:** After the update, check the Dashboard to ensure services like Unbound DNS, Intrusion Detection, and VPNs are running (green play icon).

---

## 3. Major Version Upgrades
**Example:** `25.7` to `26.1`
**Risk Level:** Moderate to High.

Major updates often upgrade the underlying FreeBSD operating system, PHP versions, and OpenSSL architectures. These have a higher chance of breaking configurations or third-party plugins.

### Best Practices
1.  **The "Wait and See" Rule:**
    *   **Do not** upgrade on Day 0 unless you are in a testing environment.
    *   Wait for the first minor release of the major version (e.g., wait for `26.1.1` instead of `26.1.0`). This allows early adopters to find and report critical bugs.
2.  **Reboot Before Upgrading:**
    *   Reboot the firewall *before* starting the upgrade process. This clears temporary files, zombie processes, and memory leaks, ensuring a clean environment for the heavy lifting of an OS upgrade.
3.  **Prefer Console Upgrade:**
    *   While the GUI works, major upgrades can take a long time and might restart the web server mid-process, leaving you staring at a timeout error wondering if it finished.
    *   **Recommendation:** SSH into the box (or use physical console), select **Option 12 (Update from console)**. This provides real-time feedback on every package being installed.
4.  **Check Plugin Compatibility:**
    *   Major versions often deprecate older plugins (e.g., `os-dyndns` vs `os-ddclient`). Verify that your critical plugins are supported in the new version.
5.  **Remove Third-Party Repositories:**
    *   If you have installed the "Mimugmail" community repo or others, check if they are compatible with the new major version. It is often safer to disable them, upgrade, and then re-enable/re-install.
6.  **Uninstall Microcode Plugins:**
    *   **Why:** Plugins like `os-cpu-microcode-intel` or `os-cpu-microcode-amd` can sometimes cause the upgrade process to hang, particularly during kernel updates.
    *   **Action:** Uninstall these plugins before the upgrade. You can reinstall them after the system is successfully running the new version.

---

## 4. Automated Updates

For users preferring a "set and forget" approach for security patches, OPNsense can update itself automatically.

### Configuration
*   **Navigate to:** `System` > `Settings` > `Cron`.
*   **Action:** Create a new job and select **Automatic firmware update** as the command. Set the schedule to a maintenance window (e.g., 3:00 AM).

### Limitations
*   **Major Versions Excluded:** This process **will not** upgrade major versions (e.g., `25.7` to `26.1`). It is restricted to minor updates within the current release cycle to prevent major breaking changes from occurring unattended.
*   **Reboots:** If a kernel update is installed, the system will reboot automatically, causing a temporary network interruption.

---

## 5. Post-Upgrade Checklist

After the system comes back online:

1.  **Verify Connectivity:** Check WAN IP and LAN internet access.
2.  **Check Logs:** Go to `System` > `Log Files` > `General`. Look for "Error" or "Fatal" messages occurring after the boot.
3.  **Test VPNs:** Connect to your VPN from an external network to ensure keys/certs are still valid.
4.  **ZFS Boot Environments (If using ZFS):**
    *   OPNsense on ZFS supports Boot Environments. If the update fails, you can select the "Pre-Upgrade" environment from the boot loader menu during restart to revert instantly.
