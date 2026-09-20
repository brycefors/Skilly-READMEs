# Blocking All DNS Except Your Own Resolver

Handing out a DNS server over DHCP is a suggestion, not a rule. Smart TVs, casting devices, phones, and IoT gear routinely ignore the lease and talk straight to `8.8.8.8`, which is how a device slips past your filter while the dashboard shows a suspiciously quiet client. The fix is enforcement at the firewall, so port 53 only reaches resolvers you approve, and everything else is either dropped or silently rewritten back to your own resolver.

This guide is resolver agnostic. Wherever it says **your resolver**, substitute whatever you actually run: **Pi-hole**, **AdGuard Home**, **Technitium DNS**, **Blocky**, or **Unbound on OPNsense itself**. The firewall rules are identical either way, because they key off an IP address and a port number, not the software behind it.

*Time: 45 mins | Requires: admin access to the OPNsense GUI, the LAN IP of your resolver, console or physical access in case a rule locks you out*

> **Read this before you start:** You are about to filter the one protocol that everything depends on. Build the **allow** rule before the **block** rule and save both in the same pass. If you apply a block rule with no matching allow rule above it, every client on the LAN loses name resolution instantly, including the machine you are configuring from.

> **Menu naming:** Recent OPNsense releases reworked the Firewall menu. Rules live on a single **Firewall** > **Rules** page with an interface dropdown instead of per-interface pages, and the NAT entries are now **Destination NAT (Port Forward)** and **Source NAT (Outbound)**. This guide uses the current names and flags the old ones where it matters.

## 1. Decide Between Blocking and Redirecting

*Time: 5 mins | No changes yet, this determines which sections you follow*

There are two enforcement strategies and they behave very differently.

1. Choose **Block** if you want rogue DNS to fail loudly. Queries to outside resolvers are dropped, the device times out, and most clients fall back to the DHCP-provided server.
2. Choose **Redirect** (NAT) if you want rogue DNS to work but be answered by your resolver. The device thinks it is talking to `8.8.8.8` and never knows the reply came from your own server.
3. Pick **Redirect** if you own Chromecasts, Google Home devices, or Roku hardware.

   > **Why this matters:** Google casting devices hardcode `8.8.8.8` and have no fallback. A block rule breaks casting entirely, while a redirect keeps them working and forces them through your blocklists. The tradeoff is that redirect hides the problem, so you never find out which devices are misbehaving unless you check the NAT rule's match counter.

4. Complete **Section 2** regardless of which path you chose. It is shared setup.

## 2. Create the Approved Resolver Alias

*Time: 8 mins*

1. Navigate to **Firewall** > **Aliases** and click **+**.
2. Set **Name** to `Allowed_DNS`.
3. Set **Type** to `Host(s)`.
4. Add the LAN IP of every resolver clients are permitted to reach. For a typical setup that is your **resolver IP** and your **OPNsense LAN IP**.
5. Set **Description** to `Approved internal DNS resolvers`.
6. Click **Save**.

   > **Why this matters:** Aliases are referenced by name in rules, so swapping a resolver IP later means editing one object instead of hunting through six rules. Include the OPNsense LAN IP even if clients are pointed at a separate blocker, otherwise a direct query to the firewall during troubleshooting will be blocked and you will chase a phantom outage.

7. Click **+** again and set **Name** to `DNS_Exempt`.
8. Set **Type** to `Host(s)`.
9. Add the **resolver IP**, plus the IP of anything that genuinely has to reach outside DNS. Common entries are a work laptop on a corporate VPN, a NAS running its own resolver, or a guest device you do not control.
10. Set **Description** to `Hosts allowed to bypass DNS enforcement`.
11. Click **Save**, then click **Apply**.

    > **Why this matters:** This is the escape hatch, and it belongs in its own alias rather than as an invert on the block rule. Adding a device later is one edit in one object with no rule reordering and no risk of changing what the block rule matches. Give every host in here a static lease first, because a DHCP address that moves turns the exemption into a hole pointed at the wrong device.

## 3. Allow DNS to the Approved Resolvers

*Time: 5 mins | Do this before Section 4*

> **Menu naming:** Rules now live on one **Firewall** > **Rules** page with an interface dropdown, rather than the old per-interface pages like Firewall > Rules > LAN. Releases that still ship both implementations side by side label the modern one **Rules [new]**. Field labels differ slightly from the legacy pages: **TCP/IP Version** is now **Version**, and **Destination port range** is now **Destination Port**.

1. Navigate to **Firewall** > **Rules**.
2. Set the **interface filter** at the top of the page to `LAN`.

   > **Why this matters:** The filter does double duty. It narrows the view to rules affecting LAN, and any rule you create while it is set gets that interface filled in automatically.

3. Click **+**.
4. On the **Interface** tab, confirm **Interface (rule)** is `LAN` and nothing else is selected.

   > **Why this matters:** The rules page derives a rule's priority group from this field. One interface makes it an interface rule at priority `400000`. Selecting a second interface, or inverting the selection, silently promotes it to a floating rule at `200000`, which is evaluated much earlier and against traffic you did not intend to match.

5. On the **Filter** tab, set **Action** to `Pass`.
6. Set **Direction** to `in`.
7. Set **Version** to `IPv4+IPv6`.
8. Set **Protocol** to `TCP/UDP`.
9. Set **Source** to `LAN net`.
10. Set **Destination** to the `Allowed_DNS` alias.
11. Set **Destination Port** to `DOMAIN (53)`.
12. On the **Organisation** tab, set **Description** to `Allow DNS to approved resolvers`.
13. Click **Save**.

    > **Why this matters:** DNS uses UDP for normal queries and TCP for responses larger than 512 bytes, which includes most DNSSEC traffic. Allowing only UDP produces the worst kind of failure, where most lookups work and a handful of domains mysteriously time out.

14. Click **+** and build a second **Pass** rule with **Interface (rule)** `LAN`, **Direction** `in`, **Version** `IPv4+IPv6`, **Protocol** `TCP/UDP`, **Source** set to the `DNS_Exempt` alias, **Destination** `any`, and **Destination Port** `DOMAIN (53)`.
15. Set its **Description** to `DNS enforcement exemptions` and give it a **Sequence** below the block rule you build next.
16. Click **Save**.

    > **Why this matters:** Your resolver sits on the LAN, so the block rule in the next section applies to its own upstream queries too. Without this rule you cut the resolver off from the internet and break every lookup on the network. The same rule covers every other host you drop into the alias, so you never touch the ruleset again to grant an exception.

## 4. Block Everything Else on Port 53

*Time: 5 mins*

1. Click **+** on the same **Firewall** > **Rules** page.
2. On the **Interface** tab, set **Interface (rule)** to `LAN`.
3. On the **Filter** tab, set **Action** to `Block`.
4. Set **Direction** to `in`.
5. Set **Version** to `IPv4+IPv6`.
6. Set **Protocol** to `TCP/UDP`.
7. Set **Source** to `LAN net` and leave **Invert Source** unchecked.

   > **Why this matters:** It is tempting to point **Source** at `DNS_Exempt` and tick **Invert Source** instead of running a separate pass rule. Avoid it. Inverting means "any address that is not in this alias," which drops the `LAN net` scoping entirely and matches traffic from anywhere that reaches this interface. You also collapse the allow decision and the block decision into one evaluation counter, so you lose the ability to see which exempt device is actually using its exemption.

8. Set **Destination** to `any`.
9. Set **Destination Port** to `DOMAIN (53)`.
10. Check **Log** so rogue clients show up in the firewall log.
11. On the **Organisation** tab, set **Description** to `Block rogue DNS`.
12. Set **Sequence** to a number **higher** than the pass rules from Section 3.
13. Click **Save**, then click **Apply**.

    > **Why this matters:** Rules are ordered by an explicit **Sort order** like `400000.0000250` rather than by where they sit on screen. The `400000` half comes from the interface, the second half is your **Sequence**. Lower sequence wins, so the pass rule must carry the smaller number. There is no drag-and-drop here. If you would rather not do the arithmetic, use the **Move rule before this rule** arrow in the action column of the rule you want this one to precede.

14. Click the **Inspect** button and confirm the pass rules appear above the block rule in the live ruleset.

    > **Why this matters:** If you upgraded from an older release that still has the legacy rules pages, your "allow LAN to any" rule probably still lives there. Legacy interface rules are evaluated *after* the modern ones, so your new block rule already lands ahead of it. Inspect mode shows the merged ruleset from both implementations, which is the only reliable way to confirm that.

15. Keep **Version** on `IPv4+IPv6` for both rules even if you think IPv6 is off.

    > **Why this matters:** If your ISP sends router advertisements with RDNSS options, clients pick up an IPv6 resolver that your IPv4 rules never see. A device can appear fully blocked on IPv4 while resolving everything over IPv6.

## 5. Block DNS over TLS and DNS over QUIC

*Time: 3 mins*

1. Click **+** on the **Firewall** > **Rules** page.
2. On the **Interface** tab, set **Interface (rule)** to `LAN`.
3. On the **Filter** tab, set **Action** to `Block` and **Direction** to `in`.
4. Set **Version** to `IPv4+IPv6` and **Protocol** to `TCP/UDP`.
5. Set **Source** to `LAN net` and **Destination** to `any`.
6. Set **Destination Port** to `853`.
7. Check **Log**.
8. On the **Organisation** tab, set **Description** to `Block DoT and DoQ`, then give it a **Sequence** near the other DNS rules.
9. Click **Save**, then click **Apply**.

   > **Why this matters:** Android calls this "Private DNS" and turns it on automatically on many carrier builds. DoT runs on TCP 853 and DoQ on UDP 853, both completely outside your port 53 rules, which is why **Protocol** has to stay on `TCP/UDP` here. Blocking 853 makes Android's automatic mode fail closed and fall back to the DHCP resolver.

## 6. Redirect Rogue DNS Instead of Dropping It

*Time: 10 mins | Skip this section if you chose Block in Section 1*

> **Menu naming:** Recent OPNsense releases renamed the NAT pages. **Port Forward** is now **Destination NAT (Port Forward)** and **Outbound** is now **Source NAT (Outbound)**. Older guides use the old names for the same pages. The rule dialog is also split into tabs (**Organization**, **Interface**, **Source**, **Destination**, **Translation**, **Options**), so the fields below are grouped rather than listed on one long form.

1. Navigate to **Firewall** > **NAT** > **Destination NAT (Port Forward)** and click **+**.
2. On the **Interface** tab, set **Interface** to `LAN`, **Version** to `IPv4`, and **Protocol** to `TCP/UDP`.
3. On the **Source** tab, set **Source Address** to `LAN net`.
4. On the **Destination** tab, check **Invert Destination**, then set **Destination Address** to the `Allowed_DNS` alias.
5. Still on the **Destination** tab, set **Destination Port** to `DNS`.
6. On the **Translation** tab, set **Redirect Target IP** to your **resolver IP** and **Redirect Target Port** to `DNS`.
7. On the **Options** tab, set **NAT Reflection** to `Disable`.
8. Still on the **Options** tab, set **Firewall rule** to `Pass`.

   > **Why this matters:** This field replaced the old `Filter rule association`, and the old `Add associated filter rule` choice is now simply `Pass`. It creates a hidden linked rule that permits the redirected traffic. Leaving it on `Manual` builds the translation with nothing to allow it, so the query gets rewritten and then dropped by your own block rule from Section 4. The third option, `Register rule`, puts a visible but uneditable rule on the Rules page if you would rather see it listed.

9. On the **Organization** tab, set **Description** to `Hijack rogue DNS to local resolver`.
10. Click **Save**, then click **Apply**.

    > **Why this matters:** The inverted destination is the whole trick. It means "everything except my approved resolvers", so traffic already headed to your resolver passes through untouched and only strays get rewritten. Without the invert you create a redirect loop where the resolver's own traffic is bounced back at itself.

11. Add a second Destination NAT rule above this one if you populated `DNS_Exempt` in Section 2. Use the same **Interface**, **Protocol**, and **Destination Port**, set **Source Address** to `DNS_Exempt`, and on the **Options** tab set **Firewall rule** to `Pass`. On the **Translation** tab leave **Redirect Target IP** empty and tick **Disable** at the top of the **Organization** tab.

    > **Why this matters:** NAT runs before the filter rules, so an exempt host gets its query rewritten to your resolver long before the pass rule from Section 3 ever sees it. A disabled rule placed above the redirect acts as a "do not translate" marker, because NAT stops at the first match and a disabled entry is not a match to translate on. If your build does not honor that, set **Redirect Target IP** to the destination itself so the translation is a no-op instead.

12. Skip to step 15 if your redirect target is the **OPNsense LAN IP** rather than a separate machine.
13. Navigate to **Firewall** > **NAT** > **Source NAT (Outbound)**, set the mode to `Hybrid Source NAT rule generation`, click **Save**, then click **+**.
14. Build the hairpin rule across the tabs:
    - **Interface** tab: **Interface** `LAN`, **Version** `IPv4`, **Protocol** `TCP/UDP`
    - **Source** tab: **Source Address** `LAN net`
    - **Destination** tab: **Destination Address** your **resolver IP**, **Destination Port** `DNS`
    - **Translation** tab: **Translate Source IP** `Interface address`
    - **Organization** tab: **Description** `Hairpin NAT for redirected DNS`

    > **Why this matters:** The client and the resolver are on the same subnet. After the redirect, the resolver sees a query from `192.168.1.50` and answers it directly, but the client is waiting for a reply from `8.8.8.8` and discards the mismatched packet. Rewriting the source to the firewall's LAN address forces the reply back through OPNsense so the address can be translated correctly.

15. Click **Save**, then click **Apply**.

## 7. Handle DNS over HTTPS

*Time: 15 mins | Requires: a resolver that accepts subscription blocklists*

DoH rides on TCP 443 alongside ordinary web traffic, so no port rule can separate it. You attack it in two places instead. Stop the endpoint hostnames from resolving, then block the endpoint IPs at the firewall for anything that hardcodes them.

### 7.1 Subscribe to HaGeZi's DoH Bypass Blocklist

HaGeZi maintains a **DoH/VPN/Tor/Proxy Bypass** list built for exactly this problem. It is published at <https://github.com/hagezi/dns-blocklists> and comes in three variants.

1. Pick **Encrypted DNS servers only** if you want DoH and DoT endpoints blocked and nothing else. This is the right default.
2. Pick the **Complete edition** instead if you also want commercial VPN, Tor, and proxy domains blocked. Pick one or the other, never both.
3. Choose the format that matches your resolver:
   - **Adblock** for Pi-hole, AdGuard Home, and Technitium DNS
   - **Wildcard Asterisk** for Blocky and for OPNsense Unbound
   - **RPZ** for Unbound, BIND, or PowerDNS running response policy zones
4. Copy the download link for that variant and format from the repository's **DoH/VPN/Tor/Proxy Bypass** section.
5. Add the URL to your resolver:
   - **Pi-hole:** **Lists** > **Add a new subscribed list**
   - **AdGuard Home:** **Filters** > **DNS blocklists** > **Add blocklist**
   - **Technitium:** **Settings** > **Blocking** > **Allow / Block List URLs**
   - **OPNsense Unbound:** **Services** > **Unbound DNS** > **Blocklist** > **Custom blocklists**
6. Force a list update and confirm the entry count jumped by a few thousand domains.

   > **Why this matters:** Every DoH client has to resolve its endpoint hostname over plain DNS at least once before it can switch to encrypted queries. That bootstrap lookup goes through your resolver, so killing the hostname there is what makes the client fall back to Do53. The list only works if it loads on the resolver clients are actually forced to use, which is what Sections 3 through 6 guarantee.

### 7.2 Return NXDOMAIN for the Firefox Canary

1. Add `use-application-dns.net` to your resolver's blocklist. It is already covered by the HaGeZi list above on most builds, but add it manually if you skipped that step.
2. Set your blocker's blocking mode to `NXDOMAIN`:
   - **Pi-hole:** **Settings** > **DNS** > **Blocking mode** > `NXDOMAIN`
   - **AdGuard Home:** **Settings** > **DNS settings** > **Blocking mode** > `NXDOMAIN`

   > **Why this matters:** Firefox queries that one canary domain at startup and disables DoH only if the reply is `NXDOMAIN`. Pi-hole's default `NULL` mode answers `0.0.0.0` instead, which Firefox reads as a successful lookup, so the canary silently fails and DoH stays on.

### 7.3 Turn Off Browser DoH by Policy

1. Set the `DnsOverHttpsMode` policy to `off` for managed Chrome and Edge installs.
2. Set the `DNSOverHTTPS` policy with `Enabled` set to false for managed Firefox installs.
3. Skip this section entirely on an unmanaged home network and rely on 7.1 and 7.2 instead.

### 7.4 Block the DoH IPs at the Firewall

*Optional. This is the only step that stops a hardcoded DoH IP, and the only one likely to break something.*

1. Navigate to **Firewall** > **Aliases** and click **+**.
2. Set **Name** to `HaGeZi_DoH_IPs`.
3. Set **Type** to `URL Table (IPs)`.
4. Set **Content** to the plain IP build of HaGeZi's DoH list at `https://cdn.jsdelivr.net/gh/hagezi/dns-blocklists@latest/ips/doh.txt`
5. Set **Refresh Frequency** to `1` day.
6. Click **Save**, then click **Apply**.
7. Create a **Block** rule on **Firewall** > **Rules** with **Interface (rule)** `LAN`, **Source** `LAN net`, **Destination** `HaGeZi_DoH_IPs`, and **Destination Port** `HTTPS`.
8. Check **Log**, click **Save**, then click **Apply**.
9. Review the block log for a week before you trust this rule.

   > **Why this matters:** That file is the IP-level companion to the DoH-only domain list, not to the complete edition, so it covers encrypted DNS endpoints and not VPN or Tor exits. Public DoH providers sit on shared CDN address space, which means a broad IP block can take ordinary websites down with it. Log first, trust later.

## 8. Verify the Rules

*Time: 5 mins*

1. Run `Resolve-DnsName google.com -Server 8.8.8.8` from a Windows client, or `dig @8.8.8.8 google.com` from Linux or macOS.
2. Confirm a **timeout** if you chose Block.
3. Confirm an **answer** if you chose Redirect, then verify the query appears in your resolver's query log as if it were normal traffic.
4. Run the same query against your approved resolver IP and confirm it succeeds either way.
5. Run `dig @8.8.8.8 +tcp google.com` to confirm the TCP side is covered too.
6. Navigate to **Firewall** > **Log Files** > **Live View** and filter on port `53`.
7. Note every client IP that keeps hitting the block rule. Those are your rogue devices.
8. Open a browser and confirm a DoH test page reports that DoH is **not** in use.
9. Reboot one client and confirm name resolution still works after the DHCP lease renews.

## 9. Troubleshooting

*Time: varies*

1. Check rule order first if **all** DNS broke. Turn on **Inspect** mode in **Firewall** > **Rules** and confirm the pass rule carries a lower **Sequence** than the block rule.
2. Verify the `Allowed_DNS` alias actually resolved to addresses under **Firewall** > **Diagnostics** > **Aliases** if the pass rule is not matching.
3. Check the rule's evaluation counter in **Inspect** mode if a rule appears correct but nothing hits it. Zero evaluations means the traffic is matching something earlier, usually a floating rule.
4. Reset states under **Firewall** > **Diagnostics** > **States** if behavior seems stuck after applying rules.

   > **Why this matters:** Existing connections keep flowing on their established state entry and ignore new rules entirely. A long-lived client can appear to bypass a correct block rule for hours until its state expires.

5. Confirm the `DNS_Exempt` pass rule from Section 3 exists and that the resolver's own IP is in the alias if the resolver itself stops resolving.
6. Add the offending client to `DNS_Exempt` under **Firewall** > **Aliases** if one specific device breaks and you are willing to let it through. The change takes effect on **Apply** with no rule edits.
7. Check for an IPv6 resolver with `ipconfig /all` on Windows or `resolvectl status` on Linux if a device still bypasses everything.
8. Remove the `HaGeZi_DoH_IPs` rule from Section 7.4 first if a legitimate website suddenly breaks. That rule is the most likely culprit for collateral damage.
9. Look for a VPN client on the device if a single machine ignores all of the above. Tunneled DNS leaves the LAN encrypted on the VPN port and your port 53 rules never see it.
10. Log in at the **console** and choose option **8** for a shell, then run `pfctl -d` to disable the packet filter temporarily if you lock yourself out of the web GUI entirely.

   > **Why this matters:** `pfctl -d` drops all filtering, including the rules protecting your WAN. Use it only long enough to fix the bad rule in the GUI, then re-enable with `pfctl -e` or reboot.
