## 📝 OPNsense-PiHole Setup Guide (Unbound + Dnsmasq)

By letting OPNsense handle DHCP and using its Unbound DNS as the only upstream server, Pi-hole gets the "secret sauce" needed to map local hostnames to IPs. This setup keeps Pi-hole focused strictly on ad-blocking muscle, while OPNsense manages the heavy lifting of network addressing and leases. Since the duties are split, you can swap out your DNS blocker whenever you want without breaking your entire infrastructure. It’s a clean way to get detailed dashboard reporting while keeping your network flexible and easy to manage.

---

### 1. ⚙️ OPNsense Configuration

#### 1.1 System: General Settings

1.  Navigate to **System** -> **Settings** -> **General**.
2.  **Domain**: Ensure you have a local domain set (e.g., `home.arpa` or `lan`). You will need this for the domain override later.
3.  **DNS servers**: Leave this blank.
4.  Ensure **Allow DNS server list to be overridden by DHCP/PPP on WAN** is **unchecked**.
 
#### 1.2 Services: Dnsmasq DNS & DHCP

We configure Dnsmasq to handle both DHCP and local DNS resolution. This ensures DHCP leases are automatically registered as DNS records.

> **Important**: Before enabling Dnsmasq for DHCP, you must disable the default ISC DHCP server. Navigate to **Services** -> **DHCPv4** -> **[LAN]** and uncheck "Enable DHCP server on the LAN interface", then click Save.

1.  Navigate to **Services** -> **Dnsmasq DNS & DHCP** -> **General**.
2.  **Enable**: Checked.
3.  **Listen Port**: `53053`.
4.  Click **Apply**.
5.  Navigate to **Services** -> **Dnsmasq DNS & DHCP** -> **DHCP ranges**.
6.  If there is no DHCP range listed, click **+**.
    *  **Interface**: LAN
    *  **Start address**: Enter your starting IP (e.g., `192.168.1.100`)
    *  **End address**: Enter your ending IP (e.g., `192.168.1.200`)
7.  Click **Save** and **Apply**.
8.  Navigate to **Services** -> **Dnsmasq DNS & DHCP** -> **DHCP options**.
9.  Click **+**.
    *  **Interface**: LAN
    *  **Type**: Set
    *  **Option**: dns-server [6]
    *  **Value**: IP address of your Pi-Hole server
10. Click **Save** and **Apply**.
 
#### 1.3 Services: Unbound DNS (Recursive Resolver)
 
We configure Unbound to handle standard DNS but forward local lookups to Dnsmasq.

1.  Navigate to **Services** -> **Unbound DNS** -> **General**.
2.  **Enable Unbound**: Checked.
3.  **Listen Port**: `53` (Default).
4.  Click **Save** and **Apply**.
7.  Navigate to **Services** -> **Unbound DNS** -> **Query Forwarding**.
8.  Click **+**.
    *   **Domain**: Enter your system domain from Step 1.1 (e.g., `home.arpa`).
    *   **IP**: `127.0.0.1` 
    *   **Port**: `53053`
    *   **Description**: Forward local queries to Dnsmasq.
9.  Click **+** again to add the **Reverse Lookup** zone.
    *   **Domain**: Enter the reverse subnet (e.g., `1.168.192.in-addr.arpa` for `192.168.1.0/24`).
    *   **IP**: `127.0.0.1`
    *   **Port**: `53053`
    *   **Description**: Forward reverse lookups to Dnsmasq.
10. Click **Save** and **Apply**.
 
---
 
### 2. 🖥️ Pi-hole Configuration
 
#### 2.1 Settings: DNS
 
1.  Navigate to **Settings** -> **DNS** on the Pi-hole web interface.
2.  Under **Custom DNS servers**, add the **LAN IP address of your OPNsense firewall**.
3.  **Uncheck all other upstream DNS servers**.
4.  Under **Advanced DNS settings**:
    *   **Uncheck** **Never forward non-FQDNs**.
    *   **Uncheck** **Never forward reverse lookups for private IP ranges**.
    *   **Use Conditional Forwarding**: Optional. You can enable this and point it to your OPNsense IP and local domain name to ensure Pi-hole logs show hostnames, but Unbound will handle the resolution regardless. If you decide to set it up, you'll use the following format `true,192.168.1.0/24,192.168.1.1`
5.  Click **Save & Apply**.
 
---
 
### 3. ✅ Final Step
 
*   **Restart Services**: It is good practice to restart both Unbound and Dnsmasq services via the OPNsense dashboard to ensure the port bindings and overrides take effect.
*   **Renew Leases**: Reconnect your client devices to pick up the new DNS settings.