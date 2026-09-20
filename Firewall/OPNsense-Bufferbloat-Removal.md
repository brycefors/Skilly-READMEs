# Removing Bufferbloat on OPNsense with FQ-CoDel (1 Gbps Plan)

Bufferbloat is not a bandwidth problem, it is a queueing problem. Your modem or ISP gear holds a huge buffer, and when the link saturates those packets sit in line instead of being dropped, so latency spikes from 10 ms to 300 ms while the download still reads "full speed". The fix is to move the bottleneck off the ISP device and onto OPNsense, where FQ-CoDel can keep the queue short. That only works if OPNsense is shaping slightly *below* your real line rate, so the ISP buffer never gets a chance to fill.

This guide is written for a **1 Gbps** plan. A gigabit line never actually delivers 1000 Mbit/s, so every number below is anchored to a realistic measured rate of **940 Mbit/s** after Ethernet and TCP overhead.

*Time: 30 mins | Requires: admin access to the OPNsense GUI, a wired client with a 1 Gbps or faster NIC, a firewall CPU that can shape gigabit*

> **Hardware reality check:** FQ-CoDel shaping runs single threaded per pipe and is not offloaded to the NIC. A modern quad core box (N100, Ryzen embedded, i3 or better) will shape a full gigabit. An older Atom C2000, J1900, or low clocked Celeron will hit a CPU wall somewhere between 300 and 600 Mbit/s and cap your throughput. Single thread clock speed matters more than core count here.

## 1. Benchmark the Connection First

*Time: 5 mins | Do this before changing anything*

1. Plug a test machine into the LAN with **Ethernet**. Wi-Fi cannot sustain gigabit and will pollute the results.
2. Confirm the test machine has a **1 Gbps or 2.5 Gbps NIC**. A 100 Mbit port will silently cap the benchmark.
3. Stop all other traffic on the network (streaming, backups, game updates, Plex scans).
4. Run the **Waveform Bufferbloat Test** at <https://www.waveform.com/tools/bufferbloat>.
5. Write down three numbers: **idle latency**, **download latency increase**, and **upload latency increase**.
6. Run a plain speed test and record your **actual** throughput in Mbit/s. Expect roughly **930 to 950 Mbit/s** down on a healthy gigabit line.
7. Record your upload separately. Symmetric fiber gives you about **940 Mbit/s** up, while gigabit cable is typically **35 to 50 Mbit/s** up.

   > **Why this matters:** You will shape against measured throughput, not the speed printed on your ISP bill. A "1 Gbps" plan that really delivers 940 Mbit/s will still bloat if you configure a 1000 Mbit pipe, because the pipe never becomes the bottleneck and the ISP buffer keeps filling.

## 2. Disable Hardware Offloading

*Time: 5 mins | Causes a brief network interruption*

1. Navigate to **Interfaces** > **Settings**.
2. Check **Disable hardware checksum offload**.
3. Check **Disable hardware TCP segmentation offload**.
4. Check **Disable hardware large receive offload**.
5. Click **Save**, then reboot the firewall.
6. Re-run a speed test and confirm you still get close to **940 Mbit/s** with no shaper configured yet.

   > **Why this matters:** Offloading hands OPNsense giant pseudo-packets (64 KB instead of 1514 bytes). The shaper counts packets, so it mis-measures the real wire rate and FQ-CoDel cannot fairly interleave flows. The tradeoff is CPU load. At gigabit, turning offload off can cost 10 to 20% of your throughput on a weak CPU, which is exactly why Step 6 checks the raw rate before shaping is added on top.

## 3. Create the Download Pipe

*Time: 5 mins*

1. Navigate to **Firewall** > **Shaper** > **Pipes** and click **+**.
2. Set **Enabled** to checked.
3. Set **Bandwidth** to `705` and set **Bandwidth Metric** to `Mbit/s`. That is **75%** of a measured 940 Mbit/s.
4. Set **Queue** to blank and leave **Mask** as `none`.
5. Set **Scheduler** to `FlowQueue-CoDel`.
6. Expand **Advanced** and set the FQ-CoDel values:
   - **FQ-CoDel quantum:** `1514`
   - **FQ-CoDel limit:** `10240`
   - **FQ-CoDel flows:** `1024`
   - **ECN:** checked
7. Set **Description** to `wan-download`.
8. Click **Save**.

   > **Why this matters:** Start at 705 Mbit/s on purpose. It is deliberately conservative, and it is the value most likely to produce an A grade on the first try, which proves the shaper is working before you start optimizing. You will claw that bandwidth back in Step 7 by raising the pipe a few percent at a time until the grade slips. Leave **quantum** at `1514` (one full Ethernet frame). The lower `300` value you see in older guides is meant for links under 100 Mbit/s and will waste CPU cycles at gigabit.

## 4. Create the Upload Pipe

*Time: 3 mins*

1. Click **+** on the **Pipes** tab again.
2. Set **Bandwidth** to **75%** of your measured upload speed:
   - **Symmetric gigabit fiber (940 up):** `705` Mbit/s
   - **Gigabit cable (35 up):** `26` Mbit/s
   - **Gigabit cable (50 up):** `37` Mbit/s
3. Apply the same **FlowQueue-CoDel** scheduler and the same advanced values from Step 3.
4. If your upload is under **100 Mbit/s**, change **FQ-CoDel quantum** to `300` on this pipe only.
5. Set **Description** to `wan-upload`.
6. Click **Save**.

   > **Why this matters:** On gigabit cable the upstream is the real problem, not the 940 Mbit download. A single 35 Mbit upstream saturates instantly and stalls the ACKs for your downstream, which makes gigabit downloads feel broken. The lower quantum on a slow upload keeps ACKs and DNS from queueing behind full-size 1514 byte frames.

## 5. Create the Queues

*Time: 5 mins*

1. Navigate to the **Queues** tab and click **+**.
2. Configure the download queue:
   - **Enabled:** checked
   - **Pipe:** `wan-download`
   - **Weight:** `100`
   - **Mask:** `Destination`
   - **Description:** `q-download`
3. Click **Save**.
4. Click **+** again and configure the upload queue:
   - **Pipe:** `wan-upload`
   - **Weight:** `100`
   - **Mask:** `Source`
   - **Description:** `q-upload`
5. Click **Save**.

   > **Why this matters:** The mask is what creates a separate flow bucket per host. Download traffic is keyed on the destination IP (your LAN client) and upload traffic on the source IP. Get these backwards and every device shares one bucket, so one torrent still drowns everyone.

## 6. Create the Shaper Rules

*Time: 5 mins*

1. Navigate to the **Rules** tab and click **+**.
2. Configure the download rule:
   - **Enabled:** checked
   - **Sequence:** `1`
   - **Interface:** `WAN`
   - **Proto:** `ip`
   - **Source:** `any`
   - **Destination:** `any`
   - **Direction:** `in`
   - **Target:** `q-download`
   - **Description:** `shape download`
3. Click **Save**.
4. Click **+** again and configure the upload rule:
   - **Sequence:** `2`
   - **Interface:** `WAN`
   - **Proto:** `ip`
   - **Direction:** `out`
   - **Target:** `q-upload`
   - **Description:** `shape upload`
5. Click **Save**.
6. Click **Apply**.

   > **Why this matters:** Direction is relative to the WAN interface. `in` is traffic arriving from the internet (your download) and `out` is traffic leaving toward the internet (your upload). Both rules must live on WAN, not LAN.

## 7. Verify and Tune

*Time: 10 mins | Repeat until the grade stops improving*

1. Re-run the **Waveform Bufferbloat Test** from the wired client.
2. Confirm the grade is **A** or **A+** and that latency increase under load stays under **30 ms**. If it is not, fix that before touching the bandwidth numbers.
3. Raise the download pipe one step and click **Apply**, then retest. Work up this ladder for a 940 Mbit/s line:
   - 80%: `752` Mbit/s
   - 85%: `799` Mbit/s
   - 90%: `846` Mbit/s
   - 95%: `893` Mbit/s
4. Retest after **every single change**. Do not skip steps.
5. Raise the upload pipe on the same 80, 85, 90, 95 percent ladder using your own measured upload number.
6. When the grade drops below **A**, go back down one step and leave it there.
7. Check **Firewall** > **Diagnostics** > **Statistics** to confirm packets are actually hitting the pipes.
8. While a speed test is running, SSH in and run `top -aSH` to watch CPU load.

   > **Why this matters:** Change one variable at a time. If you jump straight from 705 to 893 Mbit/s and the grade collapses, you have no idea where the real ceiling was. Symmetric gigabit fiber usually settles around 90% (roughly 846 Mbit/s). Gigabit cable often holds 90% down but cannot go past 80% up because of upstream buffering. If `top -aSH` shows a `dummynet` or `intr` thread pinned near 100%, the CPU is your ceiling, not the ISP.

## Troubleshooting

- **Grade did not change at all:** The rules are probably on the wrong interface or the shaper never matched. Verify both rules target **WAN** and that the pipe counters in **Firewall** > **Diagnostics** > **Statistics** are incrementing.
- **Speed test caps around 400 to 600 Mbit/s no matter what the pipe says:** This is the classic gigabit CPU wall. Run `top -aSH` during a test. If a `dummynet` thread is pinned, the box cannot shape a full gigabit and you need faster hardware. Setting the pipe higher will not help.
- **Throughput dropped the moment you disabled offloading, before any shaping:** The NIC was doing the heavy lifting. Confirm you are using a real Intel NIC (`igb`, `igc`, `ix`) rather than a Realtek onboard port, which performs badly under FreeBSD at gigabit.
- **Download is an A but upload is a D on cable:** Expected. Drop the upload pipe to **70%** of measured upstream and retest. Gigabit cable upstream buffers are enormous relative to their tiny capacity.
- **Results are inconsistent run to run:** Something else is using the link, or the test client NIC is the limit. Pause backups and cloud sync clients, confirm the client negotiated 1 Gbps, then retest.
- **Wi-Fi clients still see lag:** The shaper fixed the WAN queue, not the access point. No Wi-Fi client will see 940 Mbit/s anyway, and wireless airtime contention is a separate problem.
- **Only one device gets starved:** Re-check the queue masks. `Destination` for download and `Source` for upload is the correct pairing.
