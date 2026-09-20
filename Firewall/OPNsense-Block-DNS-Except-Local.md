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

> **Order of operations:** NAT runs **before** the filter rules, not after. A query to `8.8.8.8` gets its destination rewritten to your resolver IP first, and the packet that reaches the Rules page already has the resolver as its destination. Two consequences follow from that. The pass rule from Section 3 is what permits redirected traffic, because the translated destination matches `Allowed_DNS`. The block rule from Section 4 never sees a rogue destination on this path, so it will not log the devices you redirected.

1. Navigate to **Firewall** > **NAT** > **Destination NAT (Port Forward)** and click **+**.
2. On the **Interface** tab, set **Interface** to `LAN`, **Version** to `IPv4`, and **Protocol** to `TCP/UDP`.
3. On the **Source** tab, set **Source Address** to `LAN net`.
4. On the **Destination** tab, check **Invert Destination**, then set **Destination Address** to the `Allowed_DNS` alias.
5. Still on the **Destination** tab, set **Destination Port** to `DOMAIN (53)`.
6. On the **Translation** tab, set **Redirect Target IP** to your **resolver IP** and **Redirect Target Port** to `DOMAIN (53)`.
7. On the **Options** tab, set **NAT Reflection** to `Disable`.
8. Still on the **Options** tab, set **Firewall rule** to `Pass`.

   > **Why this matters:** This field replaced the old `Filter rule association`, and the old `Add associated filter rule` choice is now simply `Pass`. It creates a linked rule matching the **translated** traffic. If you followed Section 3 the redirect already works without it, because the rewritten destination matches `Allowed_DNS` and that pass rule lets it through. Setting this to `Pass` makes the redirect self-contained instead, so it keeps working if you later tighten the Section 3 rule or point the redirect at a host you never added to the alias. The third option, `Register rule`, does the same thing but puts a visible, uneditable entry on the Rules page.

9. On the **Organization** tab, set **Description** to `Hijack rogue DNS to local resolver`.
10. Click **Save**, then click **Apply**.

    > **Why this matters:** The inverted destination means "everything except my approved resolvers", so a query already headed to your resolver is left alone and only strays get rewritten. Without the invert the rule matches the exact opposite set, redirecting resolver-bound traffic back to the resolver as a pointless no-op while every rogue query sails past untouched.

11. Check **Invert Source** on the **Source** tab and change **Source Address** from `LAN net` to the `DNS_Exempt` alias if your resolver forwards to a **public** upstream such as `1.1.1.1` or `9.9.9.9`.

    > **Why this matters:** This is the loop. `LAN net` includes the resolver itself, so when your resolver forwards a query upstream that packet has a LAN source, a destination outside `Allowed_DNS`, and port 53. It matches this rule and gets redirected straight back to the resolver, which then queries itself. Pi-hole surfaces it as SERVFAIL or "maximum number of concurrent DNS queries reached", and every lookup on the network dies with it. Inverting the source to mean "everything except the exempt hosts" takes the resolver out of the rule. There is no cleaner option here, because the Destination NAT dialog has no equivalent of Source NAT's **Do not NAT** and a **Disabled** rule is simply not loaded rather than treated as an exclusion.

12. Skip the invert entirely if your resolver forwards to **Unbound on OPNsense**, or if your resolver *is* Unbound on OPNsense.

    > **Why this matters:** The firewall's LAN IP is already in `Allowed_DNS`, so the inverted destination excludes that traffic and no loop is possible. Unbound running on OPNsense is safer still, because its queries originate on the firewall and never arrive on the LAN interface where this rule lives.

13. Skip to step 16 if your redirect target is the **OPNsense LAN IP** rather than a separate machine.
14. Navigate to **Firewall** > **NAT** > **Source NAT (Outbound)**, set the mode to `Hybrid Source NAT rule generation`, click **Save**, then click **+**.
15. Build the hairpin rule across the tabs:
    - **Interface** tab: **Interface** `LAN`, **Version** `IPv4`, **Protocol** `TCP/UDP`
    - **Source** tab: **Source Address** `LAN net`
    - **Destination** tab: **Destination Address** your **resolver IP**, **Destination Port** `DOMAIN (53)`
    - **Translation** tab: **Translate Source IP** `Interface address` (or blank)
    - **Organization** tab: **Description** `Hairpin NAT for redirected DNS`

    > **Why this matters:** The client and the resolver are on the same subnet. After the redirect, the resolver sees a query from `192.168.1.50` and answers it directly, but the client is waiting for a reply from `8.8.8.8` and discards the mismatched packet. Rewriting the source to the firewall's LAN address forces the reply back through OPNsense so the address can be translated correctly. The cost is that every redirected query now appears to come from the firewall, so those lookups lose their per-client attribution in your resolver's statistics.

16. Click **Save**, then click **Apply**.

### 6.1 When the Redirect Returns Nothing

*Time: 10 mins | Work these in order, the first step tells you which half is broken*

1. Open your resolver's **query log** and run the failing lookup again from a client.
2. Read **which source IP** the query arrived from. That single field identifies the fault.
   - **The client's own IP**, for example `192.168.90.50`: the Destination NAT works but the **hairpin rule is missing or not matching**. Go to step 3.
   - **The OPNsense LAN IP**: both NAT rules work and the problem is downstream. Skip to step 7.
   - **Nothing at all**: the Destination NAT rule is not matching. Skip to step 8.

   > **Why this matters:** The query reaching the resolver proves the redirect fired. What breaks the round trip is the reply path, and the source address is the only thing that determines it. A resolver on the same subnet as the client answers the client *directly* and never routes back through the firewall, so the translation is never undone. The client is waiting for a packet from `8.8.8.8`, gets one from `192.168.90.11`, and discards it as unsolicited. You get a timeout with a perfectly healthy query log on the resolver.

3. Confirm **Firewall** > **NAT** > **Source NAT (Outbound)** is set to `Hybrid Source NAT rule generation` and not `Automatic`.

   > **Why this matters:** Manual rules are ignored entirely in automatic mode. The rule is listed on the page and looks active, but it is never loaded into the ruleset.

4. Confirm the hairpin rule's **Interface** is `LAN`, not WAN.
5. Confirm its **Destination Address** is the resolver IP and **Destination Port** is `DNS`.
6. Confirm **Translate Source IP** is `Interface address`, then jump to step 11.
7. Confirm your **Redirect Target IP** is a member of the `Allowed_DNS` alias, or set **Firewall rule** on the Destination NAT rule's **Options** tab to `Pass`. Jump to step 11.

   > **Why this matters:** On `Manual` nothing is auto-generated, so the translated query survives only if the Section 3 pass rule matches it. That rule matches on destination `Allowed_DNS`, so a redirect target missing from the alias gets rewritten and then dropped by the Section 4 block rule a moment later.

8. Check the Destination NAT rule's **Interface** is `LAN`. It has to be the interface the query **arrives on**, not the one it leaves by.
9. Check the rule's evaluation counter. Zero hits means the traffic never matched the rule at all.
10. Confirm **Invert Destination** is checked and **Destination Address** is `Allowed_DNS`.

    > **Why this matters:** Without the invert the rule matches only queries already headed to your approved resolvers, which is the exact opposite of what you want, and rogue queries to `8.8.8.8` sail past untouched.

11. Navigate to **Firewall** > **Diagnostics** > **States**, filter on the client IP, and reset the matching entries.

    > **Why this matters:** Every failed attempt you made while debugging left a state entry behind. Those entries keep using the ruleset that was loaded when they were created, so a correct fix looks like it changed nothing until they expire.

12. Run a packet capture under **Interfaces** > **Diagnostics** > **Packet Capture** on the `LAN` interface with the host set to your resolver IP if it still fails.

    > **Why this matters:** The capture shows the translated query leaving and the reply coming back, including the addresses on both. It settles in seconds what rule inspection can only infer.



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

1. Skip this whole subsection if you run **Pi-hole v6**. The `dns.specialDomains.mozillaCanary` setting defaults to `true` and already forces NXDOMAIN on that domain no matter what your blocking mode is.
2. Add `use-application-dns.net` to your resolver's blocklist on any other resolver. The HaGeZi list above covers it on most builds, but add it manually if you skipped that step.
3. Set your blocker's blocking mode to `NXDOMAIN`:
   - **AdGuard Home:** **Settings** > **DNS settings** > **Blocking mode** > `NXDOMAIN`

   > **Why this matters:** Firefox queries that one canary domain at startup and disables DoH only if the reply is `NXDOMAIN`. A default `NULL` blocking mode answers `0.0.0.0` instead, which Firefox reads as a successful lookup, so the canary silently fails and DoH stays on.

4. Confirm `dns.specialDomains.iCloudPrivateRelay` and `dns.specialDomains.designatedResolver` are also on if you run Pi-hole v6. Both default to `true`.

   > **Why this matters:** These close two bypasses that port rules cannot touch. iCloud Private Relay tunnels Apple device traffic past your resolver entirely, and Discovery of Designated Resolvers (RFC 9462) lets a client find an encrypted resolver by querying `resolver.arpa`. On any resolver that lacks these toggles you have to blocklist `mask.icloud.com`, `mask-h2.icloud.com`, and the `resolver.arpa` zone yourself.

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

1. Run every test below from a **LAN client**, never from the OPNsense box itself.

   > **Why this matters:** The rules in this guide match traffic arriving *on* the LAN interface. A query originating on the firewall never crosses that interface, so it skips both the block rule and the NAT redirect and will always appear to work.

2. Run `Resolve-DnsName google.com -Server 8.8.8.8` from a Windows client, or `dig @8.8.8.8 google.com` from Linux or macOS.
3. Confirm a **timeout** if you chose Block.
4. Confirm an **answer** if you chose Redirect.

   > **Why this matters:** The client cannot see the translation, so `nslookup` still prints `Server: dns.google` and `Address: 8.8.8.8` in its header. On Windows you will also likely see `DNS request timed out` and `Default Server: UnKnown` above the answer, because nslookup does a reverse lookup on the server address first and your resolver does not answer it the way Google would. Both are cosmetic and neither means the redirect failed.

5. Prove the redirect actually happened rather than assuming it. Query a domain your blocklist blocks, for example `nslookup doubleclick.net 8.8.8.8`, and confirm the reply is `0.0.0.0` or NXDOMAIN.

   > **Why this matters:** A successful answer only tells you something replied. Real `8.8.8.8` returns the genuine address for a blocked domain, so a filtered reply is the only result that can have come from your own resolver. Checking the NAT rule's match counter or your resolver's query log confirms it a second way.

6. Run the same query against your approved resolver IP and confirm it succeeds either way.
7. Run `dig @8.8.8.8 +tcp google.com` to confirm the TCP side is covered too.
8. Navigate to **Firewall** > **Log Files** > **Live View** and filter on port `53`.
9. Note every client IP that keeps hitting the block rule. Those are your rogue devices. Read the Destination NAT rule's match counter instead if you chose Redirect, because NAT rewrites the destination before the block rule is ever consulted.
10. Open a browser and confirm a DoH test page reports that DoH is **not** in use.
11. Reboot one client and confirm name resolution still works after the DHCP lease renews.

## 9. Troubleshooting

*Time: varies*

1. Check rule order first if **all** DNS broke. Turn on **Inspect** mode in **Firewall** > **Rules** and confirm the pass rule carries a lower **Sequence** than the block rule.
2. Verify the `Allowed_DNS` alias actually resolved to addresses under **Firewall** > **Diagnostics** > **Aliases** if the pass rule is not matching.
3. Check the rule's evaluation counter in **Inspect** mode if a rule appears correct but nothing hits it. Zero evaluations means the traffic is matching something earlier, usually a floating rule.
4. Reset states under **Firewall** > **Diagnostics** > **States** if behavior seems stuck after applying rules.

   > **Why this matters:** Existing connections keep flowing on their established state entry and ignore new rules entirely. A long-lived client can appear to bypass a correct block rule for hours until its state expires.

5. Confirm the `DNS_Exempt` pass rule from Section 3 exists and that the resolver's own IP is in the alias if the resolver itself stops resolving.
6. Suspect a redirect loop if your resolver logs SERVFAIL or reports that it hit its concurrent query limit. Apply step 11 of Section 6 to exclude the resolver from the Destination NAT rule.

   > **Why this matters:** A resolver forwarding to a public upstream matches your own redirect rule, gets pointed back at itself, and stops answering anything. The tell is that the failure is total and the resolver looks busy rather than idle.

7. Add the offending client to `DNS_Exempt` under **Firewall** > **Aliases** if one specific device breaks and you are willing to let it through. The change takes effect on **Apply** with no rule edits.
8. Check for an IPv6 resolver with `ipconfig /all` on Windows or `resolvectl status` on Linux if a device still bypasses everything.
9. Remove the `HaGeZi_DoH_IPs` rule from Section 7.4 first if a legitimate website suddenly breaks. That rule is the most likely culprit for collateral damage.
10. Look for a VPN client on the device if a single machine ignores all of the above. Tunneled DNS leaves the LAN encrypted on the VPN port and your port 53 rules never see it.
11. Log in at the **console** and choose option **8** for a shell, then run `pfctl -d` to disable the packet filter temporarily if you lock yourself out of the web GUI entirely.

   > **Why this matters:** `pfctl -d` drops all filtering, including the rules protecting your WAN. Use it only long enough to fix the bad rule in the GUI, then re-enable with `pfctl -e` or reboot.

## 10. Known Gaps After Everything Above

*Time: 10 mins | These are the failure modes the rules above do not cover*

1. Raise or disable Pi-hole's rate limit if you built the hairpin NAT rule in Section 6. Set `dns.rateLimit.count` well above `1000`, or set both `count` and `interval` to `0` to turn it off.

   > **Why this matters:** Pi-hole rate limits **per client** at 1000 queries per 60 seconds by default and answers everything beyond that with REFUSED. Hairpin NAT collapses every redirected query onto a single source address, the firewall's LAN IP, so the whole network shares one client's budget. A busy evening trips it and DNS dies for 60 seconds at a time. Look for `Rate-limiting 192.168.90.1 for at least NN seconds` in `/var/log/pihole/FTL.log`.

2. Repeat Sections 3 through 6 for every other interface you run. Every rule in this guide is scoped to `LAN` alone.

   > **Why this matters:** An IoT VLAN, a guest network, or a separate WiFi interface has none of this applied. Those are exactly the segments holding the devices most likely to hardcode `8.8.8.8`, so leaving them out defeats the point. Change **Interface (rule)** and the `LAN net` source to match each interface, and remember that a rule spanning several interfaces is promoted to a floating rule.

3. Accept that the redirect is IPv4 only, or build a second Destination NAT rule with **Version** set to `IPv6`.

   > **Why this matters:** The NAT rule in Section 6 uses **Version** `IPv4`, so an IPv6 query to `2001:4860:4860::8888` is never translated. It still hits the Section 4 block rule and fails closed, which is safe but inconsistent. Clients that prefer IPv6 get a timeout rather than a silent redirect, and casting hardware is the usual victim.

4. Verify **Quick** is checked on your pass rules if a correctly sequenced rule still loses to a later one.

   > **Why this matters:** With Quick set, the first matching rule wins and the sequence numbers behave the way Section 4 describes. Without it the last match wins instead, which inverts the ordering logic entirely and makes the block rule beat a pass rule sitting above it.

5. Re-run Section 8 after every OPNsense upgrade.

   > **Why this matters:** The rules and NAT pages are still being reworked between releases. Field names and defaults have changed more than once, and a migrated rule can survive the upgrade with a changed meaning rather than an obvious error.
