# General Windows Troubleshooting

> If remote support is required, start with **Windows Quick Assist** on the affected PC.
> For unattended future troubleshooting, consider **Chrome Remote Desktop** configured on the target machine.

## 1. Start with basic system checks
1. Reboot the PC into normal mode.
2. Verify Windows is fully updated: **Settings > Windows Update > Check for updates**.
3. Check battery, power, and network connections if hardware or connectivity issues are present.
4. Open **Task Manager** and sort by CPU, Memory, and Disk to identify a process using extreme resources.

## 2. Use DISM and SFC to repair Windows system files
1. Open an elevated Command Prompt or PowerShell: right-click the Start button and choose **Windows Terminal (Admin)** or **Command Prompt (Admin)**.
2. Run the DISM repair command first: `DISM /Online /Cleanup-Image /RestoreHealth`.
3. Wait for DISM to complete; if it reports corruption repaired, continue to SFC.
4. Run the system file checker: `sfc /scannow`.
5. Review the output for repair results and note whether files were fixed or if manual review is required.
6. Reboot the system after both commands finish.
7. If issues remain, rerun DISM and SFC once more and capture the command output for escalation.

> **Why this matters:** DISM repairs Windows image components and SFC restores protected system files, which helps fix corruption that can cause crashes or odd behavior.

## 3. Capture the current problem and how to continue support
1. Ask the user to describe the exact symptoms and when they first started.
2. Reproduce the issue while watching Task Manager, Event Viewer, and Windows Security notifications.
3. Open Reliability Monitor from the Run box: **Win + R**, type `perfmon /rel`, and press Enter.
4. Record error messages, event IDs, and exact wording shown on-screen.
5. If the problem persists after a reboot, escalate by collecting logs and environment details.

> **Why this matters:** Accurate symptom capture lets you compare a current failure to known causes instead of chasing random changes.

## 4. What to do when someone thinks they have a virus
1. Do not dismiss the concern; assume the system may be compromised.
2. Disconnect the PC from the network and internet immediately.
3. Open **Windows Security > Virus & threat protection** and run a full scan.
4. If Windows Security finds threats, quarantine or remove them and reboot.
5. If the first scan is clean, run a second scan with a second opinion tool like **Malwarebytes** (https://www.malwarebytes.com/mwb-download) or **ESET Online Scanner** (https://www.eset.com/us/home/online-scanner/).
6. If malware is confirmed, back up essential user files to external media before attempting repair.
7. After remediation, restore network connectivity and run a final full scan to confirm.

> **Why this matters:** Malware can persist through a single scan, so use multiple trusted tools and isolate the machine until it is clean.

## 5. Use Autoruns to inspect startup apps and drivers
1. Download Autoruns from Microsoft Sysinternals: **https://learn.microsoft.com/sysinternals/downloads/autoruns**.
2. Run **Autoruns.exe** as Administrator.
3. Accept the license and allow Autoruns to finish loading all entries.
4. Click the **Logon** tab to see all startup programs and services enabled at user login.
5. Click the **Services** and **Scheduled Tasks** tabs to inspect additional auto-start points.

### Use Autoruns specifically for suspected malware persistence
1. In Autoruns, open **Options** and enable **Hide Microsoft Entries** so third-party items stand out.
2. Press **F5** to refresh and wait for signature checks to complete.
3. Look for entries that are unsigned, recently added, oddly named, or launched from temp/appdata paths.
4. Right-click suspicious entries and choose **Search Online** to quickly validate reputation.
5. Uncheck suspicious entries first to disable startup without deleting them.
6. Reboot and confirm whether malicious behavior stops.
7. If behavior improves, run a full malware scan again and then remove the disabled item permanently only after verification.

> **Why this matters:** Many threats survive cleanup by reinstalling from startup locations; Autoruns helps you break that persistence safely by disabling first and validating before deletion.

## 6. Optimize startup apps for performance issues
1. Open **Task Manager > Startup apps** and sort by **Startup impact**.
2. Disable non-essential high-impact apps such as game launchers, chat clients, and vendor updaters.
3. Keep security tools, backup agents, and device drivers enabled unless you have a confirmed replacement.
4. Reboot and test whether boot time, responsiveness, and app stability improve.
5. Document what was disabled so changes can be reversed if the user needs a startup app restored.
6. If an entry appears suspicious rather than just unnecessary, follow the malware-specific Autoruns flow in Section 5.

> **Why this matters:** Separating performance tuning from malware triage prevents accidental removal of critical components and keeps troubleshooting decisions consistent.

## 7. Clean disk space and remove unnecessary programs
1. Run **Disk Cleanup**: type `cleanmgr` in the Run box, select the system drive, and remove temporary files, Windows Update cleanup, and recycle bin contents.
2. Use **Storage Sense**: open **Settings > System > Storage** and turn on Storage Sense to remove temporary files and unused downloads automatically.
3. Install **WizTree** from the official site: https://diskanalyzer.com/, then run it as Administrator.
4. Let WizTree scan the drive and sort results by size to find large files and folders quickly.
5. Delete or move large unused files after verifying they are not needed by the user.
6. Review installed programs and remove unneeded applications to reduce attack surface and clutter.
7. For safer bulk uninstalls, use **Bulk Crap Uninstaller**: https://www.bcuninstaller.com/.
8. Use **winget** to update apps quickly: open a Command Prompt or PowerShell and run `winget upgrade --all`.
9. After winget finishes, reboot if any upgrades require it.
10. Remove programs only after confirming the user does not need them and after recording the change.

> **Why this matters:** Unnecessary apps and unused files both consume drive space and can introduce conflicts or performance issues.

## 8. When in doubt, reinstall Windows 11
1. Back up all important user files and current configuration settings to external media.
2. Confirm that the PC meets Windows 11 hardware requirements if moving from Windows 10.
3. Download the Windows 11 Installation Assistant or Media Creation Tool from Microsoft: https://www.microsoft.com/software-download/windows11.
4. Run the tool and choose **Keep personal files and apps** for repair installs, or **Remove everything** for a clean reinstall.
5. Follow the prompts to complete the installation and let the PC reboot several times.
6. After reinstalling, reinstall only the verified applications needed by the user.
7. Restore user files from backup and verify that the issue is resolved before closing the support ticket.

## 9. Ongoing support steps after initial troubleshooting
1. Document all changes made, including disabled entries, scans run, and files backed up.
2. Keep the user informed: what was done, what was found, and the next recommended step.
3. If the issue is unresolved, escalate with a copy of the log files, Autoruns screenshot, and exact error details.
4. If the system is patched and clean, schedule a follow-up check after the next reboot.

## 10. Notes and best practices
- Always keep backups before making major repairs.
- Treat startup changes as reversible: use **Disable** not **Delete** in Autoruns.
- Use a trusted toolchain: Windows Security, Malwarebytes, Autoruns, and official Microsoft documentation.
