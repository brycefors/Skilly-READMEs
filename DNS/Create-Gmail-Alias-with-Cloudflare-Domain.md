# Create Gmail Alias with Cloudflare Domain

This guide explains how to set up a custom email address (e.g., `me@mydomain.com`) using a domain managed by Cloudflare and a personal Gmail account. This allows you to receive and send emails using your custom domain.

> **Adapted from here:** [From Reddit - How to Set Up a Custom Email Using Gmail + Cloudflare for FREE](https://www.reddit.com/r/SaaS/comments/1g0m0um/how_to_set_up_a_custom_email_using_gmail/)

## Prerequisites
*   A domain name using Cloudflare DNS.
*   A Gmail account.
*   2-Step Verification enabled on your Google Account (required for App Passwords).

## 1. Cloudflare Setup (Receiving)
First, configure Cloudflare to forward incoming emails to your Gmail inbox.

1.  Log in to the **Cloudflare Dashboard** and select your domain.
2.  Navigate to **Email** > **Email Routing** in the sidebar.
3.  Click **Get Started** (or Enable).
4.  **Add Destination Address:**
    *   Enter your Gmail address as the destination.
    *   Cloudflare will send a verification email to your Gmail. Click the link in that email to verify.
5.  **Create Custom Address:**
    *   Go back to **Email Routing** > **Routes**.
    *   Click **Create address**.
    *   **Custom address:** Enter your desired prefix (e.g., `hello` for `hello@yourdomain.com`).
    *   **Destination:** Select your verified Gmail address.
    *   Click **Save**.
6.  **DNS Records:** Cloudflare may prompt you to add required DNS records (MX and TXT). Click **Add records automatically** if prompted.

## 2. Google Account Setup (App Password)
To send email via Gmail's SMTP servers using an external alias, you need an App Password.

1.  Go to your [Google Account Security page](https://myaccount.google.com/security).
2.  Under **How you sign in to Google**, ensure **2-Step Verification** is ON.
3.  Search for or navigate to **App passwords** (usually at the bottom of the 2-Step Verification page).
4.  Create a new app password:
    *   **App name:** "Cloudflare Email" (or similar).
    *   Click **Create**.
5.  **Copy the 16-character password** generated. You will need this in the next step.

> **Security Note:** An App Password grants access to your Google Account (specifically for Mail) and bypasses 2-Step Verification. Treat it with the same care as your main password. If you ever stop using this setup, remember to delete the App Password in your Google Account settings.

## 3. Gmail Setup (Sending)
Configure Gmail to send mail as your custom domain.

1.  Open Gmail and go to **Settings** (gear icon) > **See all settings**.
2.  Click the **Accounts and Import** tab.
3.  Under **Send mail as**, click **Add another email address**.
4.  **Configure the Alias:**
    *   **Name:** Enter the name you want recipients to see.
    *   **Email address:** Enter your custom domain email (e.g., `hello@yourdomain.com`).
    *   **Treat as an alias:** Checked (usually recommended).
    *   Click **Next Step**.
5.  **SMTP Settings:**
    *   **SMTP Server:** `smtp.gmail.com`
    *   **Port:** `587`
    *   **Username:** Your *original* Gmail address (e.g., `user@gmail.com`).
    *   **Password:** The **App Password** you generated in Section 2.
    *   **Connection:** Secured connection using **TLS**.
    *   Click **Add Account**.
6.  **Verify:**
    *   Gmail will send a confirmation code to your custom email address.
    *   Since Cloudflare is forwarding this to your inbox, check your Gmail for the code.
    *   Enter the code in the popup window and click **Verify**.

## 4. (Optional) Advanced: Configure SMTP Relay (Fix DKIM)
### Important Note on Deliverability (DKIM)
This setup uses Gmail's SMTP servers to send mail. While convenient and free, it has limitations regarding **DKIM (DomainKeys Identified Mail)**:

*   **The Issue:** Free Gmail accounts sign outgoing emails with a generic `d=gmail.com` signature, not your custom domain's signature.
*   **The Consequence:** This causes a DKIM alignment failure. While emails will usually be delivered if your **SPF** record is correct (ensure you add `include:_spf.google.com` to your domain's SPF record), they may be marked as spam if they pass through forwarding servers where SPF breaks.
*   **Recommendation:** For critical business email, consider a paid service (like Google Workspace). Alternatively, you can use an **SMTP Relay** (like Brevo, SMTP2GO, or SendGrid). By configuring Gmail to send through the relay's SMTP server instead of Google's, you can set up custom DKIM records for your domain, ensuring proper alignment and better deliverability.

**This step is optional.** If you are satisfied with the basic setup, you can stop here.

However, setting up an SMTP relay is the superior technical path because it allows you to sign outgoing mail with a DKIM key specific to your custom domain, rather than relying on Gmail's generic signature. This ensures "Alignment," which is critical for passing strict DMARC policies.

Here is the technical blueprint for setting this up using a common provider like Brevo (formerly Sendinblue) or SMTP2GO.

### 1. Choose and Configure Your SMTP Provider
The SMTP relay acts as your "sending" engine.

*   **Create an Account:** Sign up for a service like **Brevo** (free for 300 emails/day) or **SMTP2GO** (free for 1,000 emails/month).
*   **Add Your Domain:** In the provider's dashboard, go to **Senders & Domains** and add your custom domain (e.g., `yourdomain.com`).
*   **Generate DKIM/SPF Records:** The provider will generate specific DNS records (usually 2-3 CNAME or TXT records) that contain your domain's unique public DKIM key.

### 2. Update Cloudflare DNS Records
To "authorize" the relay to send mail on your behalf, you must add the generated records to Cloudflare.

*   **Add DKIM:** Copy the records from your provider. In Cloudflare, go to **DNS** > **Records** and add them as TXT or CNAME records. Ensure the proxy status (orange cloud) is **Off (DNS Only)**.
*   **Merge SPF:** You likely already have an SPF record for Cloudflare Email Routing. You must merge the relay's SPF into your existing one.
    *   *Cloudflare Only:* `v=spf1 include:_spf.mx.cloudflare.net ~all`
    *   *Cloudflare + SMTP Relay (e.g., Brevo):* `v=spf1 include:_spf.mx.cloudflare.net include:spf.brevo.com mx ~all`
    > **Note:** Never have two separate SPF records. They must be combined into a single string.

### 3. Connect Gmail to the Relay
Now, configure Gmail to use the relay's "pipes" instead of its own.

*   **Get SMTP Credentials:** In your relay dashboard, find the SMTP Settings. You will need:
    *   **SMTP Server:** (e.g., `smtp-relay.brevo.com` or `mail.smtp2go.com`)
    *   **Port:** `587` (TLS)
    *   **Username/Password:** Provided by the relay.
*   **Gmail Configuration:**
    *   Go to **Gmail Settings** > **Accounts and Import** > **Send mail as**.
    *   Add your custom email (or edit the existing alias).
    *   Enter the Relay's SMTP server and the SMTP credentials you just gathered (not your Gmail password).
    *   **Cleanup:** Once the relay is successfully set up, make sure to delete the Google App Password created in Section 2, as it is no longer needed.

## Wrap Up
You can now send and receive emails from your custom domain directly within Gmail. When composing a new email, simply click the "From" field to select your custom address.