# Increasing OPNsense Throughput and Latency Performance

OPNsense ships conservative defaults, so a stock install pins network processing to a single CPU core and keeps every CPU speculation mitigation enabled. On a dedicated firewall behind your own NAT, both of those choices cost real throughput for little practical benefit. The changes below spread packet processing across all cores, reclaim the cycles lost to Spectre and Meltdown mitigation, then spend the recovered headroom on a shaper that keeps latency flat under load.

*Time: 55 mins | Requires: admin access to the OPNsense GUI, shell or SSH access, three reboots, a multi-core firewall CPU*

> **Read this before you start:** Every tunable here takes effect only after a reboot. Change one group, reboot, confirm the box still passes traffic, then move on. Batching all three phases into one reboot makes a bad tunable impossible to isolate.

## 1. Enable PowerD

*Time: 2 mins | No reboot required*

1. Navigate to **System** > **Settings** > **Miscellaneous** and find **Power Savings**.
2. Check **Use PowerD**.
3. Leave all three power mode dropdowns (**on AC power**, **on battery power**, **on unknown power**) set to `Hiadaptive`.
4. Click **Save**.

   > **Why this matters:** Without PowerD the CPU sits at its default P-state and takes a noticeable moment to ramp when a burst arrives, which shows up as lost throughput on short transfers. `Hiadaptive` scales clock speed with load but biases hard toward staying high, so the ramp happens before the packets do. Do not use `Adaptive` on a firewall. It favors power savings and will undershoot on bursty traffic.

## 2. Enable RSS (Receive Side Scaling)

*Time: 15 mins | Requires a reboot and shell access | Reference: <https://docs.opnsense.org/troubleshooting/performance.html>*

RSS is disabled by default on purpose. OPNsense turns it off because a NIC driver that implements the RSS interface incorrectly will regress instead of improve. Confirm your hardware qualifies before you touch a single tunable.

### Confirm Your NIC Supports It

1. Open a shell through **System** > **Diagnostics** > **Command Prompt** or over SSH.
2. Run `dmesg | grep vectors` and look for multiple MSI-X vectors per interface. Output such as `igb0: Using MSI-X interrupts with 5 vectors` means the NIC has multiple hardware queues.
3. Run `sysctl -a | grep rss` to see whether your driver exposes RSS as a configurable tunable.
4. Confirm your driver appears on the supported list: **em**, **igb**, **axgbe**, **netvsc**, **ixgbe**, **ixl**, **cxgbe**, **lio**, **mlx5**, **sfxge**. Only **igb** and **axgbe** are tested and confirmed working by the OPNsense team.

   > **Why this matters:** Realtek (`re`) is absent from that list. A NIC with no RSS and no other packet filter interrupts CPU 0 for every single packet, and enabling system-wide RSS on top of that costs you throughput through cache line migrations and lock contention. If your hardware is not on the list, stop here and leave RSS disabled.

### Apply the Tunables

1. Navigate to **System** > **Settings** > **Tunables**.
2. Add or edit the tunable **net.isr.bindthreads** and set it to `1` to bind each thread to a CPU.
3. Add or edit the tunable **net.isr.maxthreads** and set it to `-1` to assign a workstream to every available core.
4. Add or edit the tunable **net.inet.rss.enabled** and set it to `1`.
5. Add or edit the tunable **net.inet.rss.bits** and set it to the number of bits representing your core count. Use `2` for a **4 core** CPU, `3` for **8 cores**, and `4` for **16 cores**.

   > **Why this matters:** The value is a bit count, not a core count. RSS creates $2^{\text{bits}}$ buckets, so `2` gives you 4 queues. The FreeBSD default is actually cores x 2 in binary, intended for a load-balancing feature that was never implemented, which is why OPNsense recommends dropping it to match your real core count instead.

6. Click **Save**, then **Apply**.
7. Reboot the firewall.

   > **Why this matters:** `net.inet.rss.enabled` is read only during boot. Applying it from the GUI changes nothing until the box restarts, so a "no difference" test result before rebooting is meaningless.

### Validate the Result

1. Run `netstat -Q` and confirm the dispatch policy has moved from `direct` to `hybrid`.
2. Push traffic through the WAN and confirm load is spread across cores instead of pegging a single one.
3. Run `top -P` and expect **interrupt load to rise**. That is the intended outcome, not a regression.

   > **Why this matters:** Higher interrupt numbers here mean packets are being processed at the highest priority in the CPU scheduler. It does not mean the CPU is doing more work. Misreading this line is the most common reason people revert a working RSS configuration.

> **If you run Suricata in IPS mode:** RSS gains you nothing. Netmap pulls packets off the line for inspection, and the current implementation re-injects inspected packets into the host stack through a single thread. That thread becomes the bottleneck regardless of how many RSS queues feed it.

> **If your WAN is PPPoE:** RSS will not help the WAN side. PPPoE terminates in a single netgraph thread, so the whole tunnel is pinned to one core no matter how many queues the NIC offers. Skip **net.inet.rss.enabled** and **net.inet.rss.bits**, then set **net.isr.dispatch** to `deferred` and set **net.isr.maxthreads** to your actual **CPU thread count** instead of `-1`. Reference: <https://kb.protectli.com/kb/pppoe-and-opnsense/>

## 3. Disable Spectre and Meltdown Mitigation

*Time: 10 mins | Requires a reboot | Reference: <https://docs.opnsense.org/troubleshooting/hardening.html#spectre-and-meltdown>*

> **Decide before you change anything:** These mitigations defend against untrusted code running locally on the firewall. A dedicated appliance that only routes packets and runs no third party workloads is a low risk target. If the box is shared, runs containers or plugins from outside sources, or hosts anything multi-tenant, skip this phase entirely.

1. Navigate to **System** > **Settings** > **Tunables**.
2. Add or edit **vm.pmap.pti** and set it to `0` to disable Meltdown mitigation (PTI).

   > **Why this matters:** PTI (Page Table Isolation) splits kernel and user page tables, so every syscall pays a TLB flush. A firewall makes a huge number of syscalls per second, which is why this single tunable often returns the largest measurable gain on older Intel silicon.

3. Add or edit **hw.ibrs_disable** and set it to `1` to disable the IBRS-based Spectre mitigation.
4. Click **Save**, then **Apply**.
5. Reboot the firewall.
6. Run a speed test and record the new throughput number so you can compare it against your pre-change baseline.

   > **Why this matters:** The gain scales inversely with CPU age. Pre-2019 Intel CPUs handle these mitigations in microcode and can recover 20 to 30% throughput. Modern CPUs with hardware fixes baked into silicon will barely move, so measure instead of assuming.

## 4. Enable the Shaper to Remove Bufferbloat

*Time: 25 mins | Reference: <https://docs.opnsense.org/manual/how-tos/shaper_bufferbloat.html>*

Raw throughput is only half the story. Once the link saturates, the ISP buffer fills and latency climbs from 10 ms to 300 ms even though the download still reads full speed. The shaper moves the bottleneck onto OPNsense where FQ-CoDel can keep the queue short.

1. Configure the download and upload pipes with the **FlowQueue-CoDel** scheduler. Full walkthrough lives in [OPNsense-Bufferbloat-Removal.md](OPNsense-Bufferbloat-Removal.md).
2. Set **Bandwidth** to `750` or `800` Mbit/s on both the up and down pipes.
3. Set **FQ-CoDel Quantum** to `2100` for a 750 Mbit pipe, or `2400` for an 800 Mbit pipe.

   > **Why this matters:** Quantum is the per-flow batch size the scheduler hands out before rotating to the next flow. The working rule is roughly **300 bytes per 100 Mbps**, so a 750 Mbit pipe wants about 2100 bytes. Leaving it at the 1514 byte default forces far more scheduler rotations per second at these rates, which burns CPU for no latency benefit.

4. Check **(FQ-)CoDel ECN** and run a bufferbloat test.
5. Watch CPU load during that test. If a core is pegged, uncheck **(FQ-)CoDel ECN** in the pipe settings and test again.

   > **Why this matters:** ECN marks packets instead of dropping them, which is the better outcome when the CPU can keep up. When it cannot, the extra per-packet work becomes the new bottleneck and latency gets worse than it was with no shaping at all. On a CPU-limited box, turning ECN off is a net win.

6. Re-run the **Waveform Bufferbloat Test** at <https://www.waveform.com/tools/bufferbloat> and confirm the latency increase under load stays in single or low double digit milliseconds.

## Rollback

*Time: 5 mins | Requires a reboot*

1. Navigate to **System** > **Settings** > **Tunables**.
2. Delete the tunable you suspect, or set it back to the OPNsense default.
3. Reboot and retest.

   > **Why this matters:** Tunables apply at boot, before the GUI is reachable. If a bad value makes the box unbootable or drops the network, you will need console or serial access to recover, which is the reason each phase above gets its own reboot and its own verification step.
