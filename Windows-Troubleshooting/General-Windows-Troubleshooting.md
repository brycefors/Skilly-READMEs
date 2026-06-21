# General Windows Troubleshooting

> If remote support is required, start with **Windows Quick Assist** on the affected PC. Note that Quick Assist does not handle UAC prompts well - the helper cannot interact with elevation dialogs, which blocks many admin tasks. If UAC access is needed, use **TeamViewer** (https://www.teamviewer.com) instead, as it supports full elevation handling during a session.
> For unattended future troubleshooting, consider **Chrome Remote Desktop** configured on the target machine.

## Before You Start
The biggest mistake is changing too many things too fast. Make one change, test, then continue. That way you always know what actually fixed the problem.

## 1. Quick first checks (2-5 minutes)
1. Reboot the PC normally.
2. Check for updates in **Settings > Windows Update > Check for updates**.
3. Confirm basics: power cable, battery level, Wi-Fi or Ethernet, and monitor connection.
4. Open **Task Manager** and sort by CPU, Memory, and Disk to spot anything maxing out resources.
5. If Task Manager is not enough, run **Process Explorer** (Sysinternals) to inspect parent/child process trees, command lines, and loaded modules: https://learn.microsoft.com/en-us/sysinternals/downloads/process-explorer.

> **Why this matters:** A lot of "serious" issues are just a stuck process, pending update, or simple connection problem.

## 2. If Windows feels broken, run repair commands (10-25 minutes)
1. Open **Windows Terminal (Admin)** from the Start button menu.
2. Run `DISM /Online /Cleanup-Image /RestoreHealth` and wait for it to finish.
3. Run `sfc /scannow` after DISM completes.
4. Restart the PC.
5. If the issue is still there, run both commands one more time.

> **Why this matters:** DISM repairs the Windows image. SFC repairs protected system files. Together they fix many random crashes and weird OS behavior.

## 3. Nail down the exact symptom before guessing (5-15 minutes)
1. Ask what happened right before the issue started (new app, update, game install, driver change).
2. Reproduce the problem once while watching **Task Manager**.
3. Open Reliability Monitor: press **Win + R**, run `perfmon /rel`, and check recent red X entries.
4. Write down exact error text and time of failure.

> **Why this matters:** Exact wording and timing save hours. "It crashed" is too vague to troubleshoot well.

## 4. Check browser issues first: cache and extensions (5-15 minutes)
1. Reproduce the issue in a private/incognito window.
2. Clear cached images/files and cookies in the affected browser.
3. Disable all browser extensions, then test the problem page again.
4. Re-enable extensions one at a time until the issue returns.
5. Remove unknown, unused, or recently added extensions.
6. Check and clean up browser notification permissions: open browser **Settings > Privacy and security > Notifications** (or Site Settings), revoke any site you don't recognize, and block new requests by default.
7. Keep only trusted security/privacy tools, such as:
	- **uBlock Origin** (blocks malicious ad networks and scam redirects)
	- **Bitwarden** (uses unique passwords and reduces credential reuse)
	- **Malwarebytes Browser Guard** (adds phishing and scam page blocking)

> **Why this matters:** Browser cache corruption and bad extensions can mimic malware, break logins, and cause random page failures. Rogue notification permissions are a common scam vector - fake virus alerts and phishing popups are frequently pushed this way. Testing with extensions off isolates browser-side problems quickly.

## 5. If someone thinks they have malware, treat it seriously (20-60+ minutes)
1. Disconnect from Wi-Fi or unplug Ethernet right away.
2. Run a full scan in **Windows Security > Virus & threat protection**.
3. Quarantine or remove anything found, then reboot.
4. Download **Process Explorer** from Sysinternals if it is not already installed: https://learn.microsoft.com/en-us/sysinternals/downloads/process-explorer.
5. In **Process Explorer**, turn on VirusTotal lookups: go to **Options > VirusTotal.com > Check VirusTotal.com**, accept the prompt, then review the **VirusTotal** column (for example, `0/76` is usually clean. Higher detections need investigation).
6. Run a second-opinion scan with **Malwarebytes** (https://www.malwarebytes.com/mwb-download) or **ESET Online Scanner** (https://www.eset.com/us/home/online-scanner/).
7. Keep the PC offline until scans come back clean.

> **Why this matters:** One clean scan is not always enough. Some threats only show up on a second tool or after reboot.

## 6. Check startup junk (and suspicious entries) with Autoruns (10-30 minutes)
1. Download Autoruns from Microsoft Sysinternals: https://learn.microsoft.com/sysinternals/downloads/autoruns.
2. Run **Autoruns.exe** as Administrator.
3. Go to **Options** and enable **Hide Microsoft Entries**.
4. Check **Logon**, **Services**, and **Scheduled Tasks**.
5. Watch specifically for remote access tools that may have been installed without the user's knowledge. Common ones include **ScreenConnect**, **AnyDesk**, **UltraViewer**, **TeamViewer**, and **AeroAdmin**. Legitimate installs are fine. Unknown ones are a red flag.
6. If something looks suspicious, right-click and use **Search Online**.
7. Uncheck entries first (disable), reboot, and test.

> **Why this matters:** Disable first, delete later. Reversible changes are safer when helping non-technical users. Remote access tools left running in the background are a classic technique used in tech support scams to regain access to a machine.

## 7. Speed up a slow PC safely (10-20 minutes)
1. Open **Task Manager > Startup apps**.
2. Disable high-impact apps that are clearly optional (game launchers, chat apps, RGB tools, vendor tray apps).
3. Keep antivirus, backup, and hardware driver software enabled unless you are sure.
4. Reboot and test startup speed.

> **Why this matters:** Startup overload is one of the most common reasons home PCs feel slow.

## 8. Free space and clean clutter (15-45 minutes)
1. Run Disk Cleanup: press **Win + R**, run `cleanmgr`, choose the system drive, and clean temporary files.
2. Turn on **Storage Sense** in **Settings > System > Storage**.
3. Use **WizTree** (https://diskanalyzer.com/) to find giant folders fast.
4. Remove unused programs from **Settings > Apps > Installed apps**.
5. For batch uninstalls, use **Bulk Crap Uninstaller** (https://www.bcuninstaller.com/).
6. Update installed apps with `winget upgrade --all`.

> **Why this matters:** Low disk space causes updates, caching, and apps to fail in unpredictable ways.

## 9. Long-term maintenance plan (10-30 minutes per week)
1. Install or update **Microsoft PC Manager** from the Microsoft Store and keep it on the latest version.
2. Open **PC Manager > Settings** and enable **Start Microsoft PC Manager automatically when I sign in to Windows**.
3. Once per week, open **PC Manager > Health Check** and remove temporary files and obvious startup clutter.
4. In **Storage Management**, use deep cleanup carefully and review file categories before deleting anything.
5. In **Process Management**, close runaway apps and disable non-essential startup items only if you understand what they do.
6. In **Windows Update** and **Protection** sections, confirm updates and security status are current.
7. Keep this cadence:
	- **Weekly:** Health Check, restart PC, verify free disk space.
	- **Monthly:** Review startup apps, uninstall unused programs, run full Windows Security scan.
	- **Quarterly:** Check driver and BIOS or firmware updates from the device manufacturer, review backup restore readiness.
8. Keep at least 20% free space on the system drive to reduce update and performance issues.

> **Why this matters:** Most Windows problems are preventable drift. A small maintenance routine catches disk pressure, startup bloat, and stale updates before they become hard failures. PC Manager is useful as a single dashboard, but avoid aggressive "one-click" cleaning habits and always review what will be removed.

## 10. Last resort: repair install or clean reinstall Windows 11 (45-120+ minutes)
1. Back up important files first.
2. Download Microsoft's installer tools: https://www.microsoft.com/software-download/windows11.
3. Choose **Keep personal files and apps** for a repair install.
4. Choose **Remove everything** only for a full reset/clean reinstall.
5. Reinstall only needed apps afterward and restore data.

> **Why this matters:** Reinstalling works, but it is time-consuming. Try the earlier sections first.

## 11. Wrap-up checklist
1. Confirm the original issue is fixed.
2. Re-enable internet if malware checks are complete and clean.
3. Tell the person what changed in plain language.
4. Keep a short note of changes so you can undo them later if needed.

## Quick rules to avoid making things worse
- Back up before major changes.
- Make one change at a time, then retest.
- Prefer official tools and downloads.
- In Autoruns, disable first and only delete after you are confident.
