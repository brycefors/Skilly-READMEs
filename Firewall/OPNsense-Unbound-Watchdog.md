# OPNsense Unbound Watchdog with Monit

Unbound has a nasty habit of crashing when it’s forced to restart too frequently or when its local database becomes bloated and unstable. It essentially gets stuck in a loop of trying to update its records, eventually hitting a technical "dead end" where the service just gives up and stops responding.

To configure Monit on OPNsense to monitor and automatically restart the Unbound DNS service, you need to set up two distinct components: a **Service Test** (the "logic" that detects the failure) and a **Service Setting** (the "rule" that tells Monit what to do).

## 1. Enable Monit
1.  Navigate to **Services** > **Monit** > **Settings**.
2.  On the **General Settings** tab, check the **Enable Monit** box.
3.  **Polling Interval**: Ensure this is set to a reasonable number (e.g., `120` for two minutes). This is how often Monit wakes up to check if Unbound is still running.
4.  Click **Save**.
5.  Click **Apply**.
 
## 2. Create the Service Test
This defines the "if" condition (e.g., "if the process does not exist").

1.  Navigate to **Services** > **Monit** > **Settings**.
2.  Click the **Service Test Settings** tab and click the **+** (Add) button.
3.  Enter the following:
    *   **Name:** `unbound_process_check`
    *   **Condition**: `does not exist`
    *   **Action**: `Restart`
4.  Click **Save**.
 
## 3. Create the Service Setting
This connects the test to the actual Unbound process and tells Monit how to start it.

1.  Click the **Service Settings** tab and click the **+** (Add) button.
2.  Enter the following:
    *   **Enabled**: Checked
    *   **Name**: `unbound_dns`
    *   **Type**: `Process`
    *   **PID File**: `/var/run/unbound.pid`
    *   **Start**: `/usr/local/sbin/pluginctl -s unbound start`
    *   **Stop**: `/usr/local/sbin/pluginctl -s unbound stop`
    *   **Tests**: Select `unbound_process_check` (from the previous step).
    *   **Description**: Restart Unbound DNS if process is missing.
3.  Click **Save**.
4.  Click **Apply** in the Monit settings page.
