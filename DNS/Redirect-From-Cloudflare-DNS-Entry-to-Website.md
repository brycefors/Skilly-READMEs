# Redirect From Cloudflare DNS Entry to Website

To redirect `yourdomain.com` to another URL (e.g., a GitHub profile), the most modern and efficient method is using **Cloudflare Redirect Rules** (also known as Single Redirects).

Cloudflare has largely moved away from "Page Rules" in favor of these more granular and faster "Redirect Rules." This method is preferred because it executes at the edge and does not require a web server to host your site.

## Step 1: Configure DNS for Proxying
For any Cloudflare rule to work, traffic must actually pass through Cloudflare's network. This requires a "Proxied" DNS record.

1.  Log in to your **Cloudflare dashboard** and select `yourdomain.com`.
2.  Go to **DNS** > **Records**.
3.  **Create a new record:**
    *   **Type:** `A`
    *   **Name:** `@` (represents your root domain)
    *   **IPv4 address:** `192.0.2.1` (This is a standard "dummy" IP used for redirects; it doesn't need to be a real server).
    *   **Proxy status:** Proxied (Orange cloud icon).
4.  (Optional) Repeat for a CNAME or A record named `www` if you want `www.yourdomain.com` to also redirect.

## Step 2: Create the Redirect Rule
Now that traffic is flowing through the proxy, you can tell Cloudflare where to send it.

1.  Go to **Rules** > **Redirect Rules**.
2.  Click **Create rule**.
3.  **Rule name:** Give it a name like "Redirect to Destination".
4.  **If incoming requests match...:**
    *   Select **Custom filter expression**.
    *   **Field:** `Hostname`
    *   **Operator:** `equals`
    *   **Value:** `yourdomain.com`
    *   *Note: If you want to include `www`, change the operator to "is in" and add both hostnames.*
5.  **Then...:**
    *   **Type:** `Static`
    *   **URL:** `https://destination-url.com`
    *   **Status code:** `301` (Permanent redirect — best for SEO and browser caching).
    *   **Preserve query string:** Usually, you can leave this off unless you want to pass parameters to the destination.
6.  Click **Deploy**.