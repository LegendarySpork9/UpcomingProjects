# Cloudflare Gateway Parental Controls Setup Guide

A guide to setting up Cloudflare Zero Trust Gateway with the WARP client to filter and restrict internet access on children's devices. Works on home WiFi, mobile data, and any other network.

---

## How It Works

```
Child's Device                     Cloudflare Network             Parent Dashboard
┌─────────────────┐              ┌──────────────────┐           ┌──────────────┐
│ WARP client      │──DNS query──▶│ Gateway checks    │           │ Block lists   │
│ (runs as local   │              │ domain against    │◀──────── │ SafeSearch    │
│  VPN, locked on) │◀─allow/block─│ your policies     │           │ Categories    │
└─────────────────┘              └──────────────────┘           │ Activity logs │
                                                                 └──────────────┘
```

- The WARP client runs on every device you want to protect.
- It intercepts all DNS lookups and routes them through Cloudflare Gateway.
- Gateway checks each lookup against your filtering policies and blocks or allows accordingly.
- Running in **DNS-only mode** means only the tiny DNS queries go through Cloudflare — actual web traffic (video streaming, downloads, etc.) goes directly to the internet, so there is negligible data usage against the free tier's 10GB transfer limit.
- Because WARP occupies the device's VPN slot and is locked on, the child cannot install a bypass VPN.

---

## What You Can Filter

| Type | Examples | How |
|------|----------|-----|
| Content categories | Adult content, gambling, drugs, violence, dating | Gateway blocks entire categories of domains |
| Specific domains | tiktok.com, reddit.com, etc. | Custom block list |
| Search results | Explicit results on Google, Bing, YouTube, DuckDuckGo | SafeSearch enforcement |
| Security threats | Malware, phishing, botnets, command & control | Security category blocking |

---

## What You Need

- A Cloudflare account (free)
- Cloudflare Zero Trust (free tier — up to 50 users, more than enough for a family)
- The children's devices: iOS, Android, and/or Windows
- Admin/parent access to each device

---

## Step 1 — Set Up Cloudflare Zero Trust

1. Go to the [Cloudflare Zero Trust dashboard](https://one.dash.cloudflare.com).
2. Log in with your existing Cloudflare account, or create one.
3. You will be prompted to choose a **team name**. This is the identifier your devices will use to connect.
   - Example: `hunter-family`
   - This cannot be easily changed later, so choose something sensible.
4. Select the **Free plan** (up to 50 users, no credit card required).

---

## Step 2 — Create DNS Filtering Policies

Navigate to **Gateway > Firewall Policies > DNS** in the Zero Trust dashboard.

Create the following policies in order. Policies are evaluated by precedence — put the most important ones first.

### Policy 1: Block Security Threats

This catches malware, phishing, botnets, and other threats.

| Field | Value |
|-------|-------|
| Name | Block Security Threats |
| Selector | Security Categories |
| Operator | in |
| Values | Select all available security categories |
| Action | Block |

### Policy 2: Block Inappropriate Content Categories

| Field | Value |
|-------|-------|
| Name | Block Inappropriate Content |
| Selector | Content Categories |
| Operator | in |
| Values | Select categories to block (see list below) |
| Action | Block |

**Recommended categories to block:**

- Adult Themes
- Gambling
- Drugs
- Violence
- Weapons
- Hate & Discrimination
- Nudity
- Pornography
- Dating
- Questionable Content
- Proxy/Anonymiser (prevents web-based VPN/proxy bypass)

Review all available categories and select what is appropriate for your children's ages.

### Policy 3: Enforce SafeSearch

This forces safe search results on major search engines and YouTube.

| Field | Value |
|-------|-------|
| Name | Enforce SafeSearch |
| Selector | Content Categories |
| Operator | in |
| Values | Search Engines (category 145) |
| Action | Safe Search |

This covers Google, Bing, Yandex, YouTube, and DuckDuckGo.

### Policy 4 (Optional): Block Specific Domains

For any specific sites you want to block that may not fall into a category.

| Field | Value |
|-------|-------|
| Name | Block Specific Sites |
| Selector | Domain |
| Operator | in |
| Values | List of domains, e.g. `tiktok.com`, `reddit.com` |
| Action | Block |

---

## Step 3 — Enable Activity Logging

1. Navigate to **Traffic policies > Traffic settings** in the dashboard.
2. Enable **Activity logging** for DNS logs.
3. This lets you see what domains your children's devices are accessing and what is being blocked.
4. Note: the free tier retains logs for 24 hours.

---

## Step 4 — Configure WARP Client Settings

Navigate to **Settings > WARP Client** in the dashboard and configure the device profile.

### Service Mode

Set to **DNS only**. This ensures only DNS queries go through Cloudflare, keeping data usage negligible.

### Lock Settings

| Setting | Value | Purpose |
|---------|-------|---------|
| Lock WARP switch | Enabled | Prevents the child from disconnecting WARP |
| Auto connect | 0 minutes | Reconnects immediately if somehow disconnected |
| Admin override | Set a timeout (e.g. 5 minutes) | Gives you an hourly-rotating code to temporarily disable WARP for troubleshooting |

With these settings, the child cannot turn off the filtering. You can use the admin override code if you ever need to temporarily disable it (e.g. if a legitimate site is incorrectly blocked).

---

## Step 5 — Install WARP on Devices

### iOS / iPadOS

#### Install the app

1. On the child's device, open the **App Store**.
2. Search for **"Cloudflare One Agent"** and install it.
3. Open the app.
4. Tap the menu, go to **Settings > Account > Login to Zero Trust**.
5. Enter your team name (e.g. `hunter-family`).
6. The app will prompt to install a VPN profile — approve it.
7. Verify the connection is active (the app should show "Connected").

#### Lock it down with Screen Time

Since most family iOS devices won't be under full MDM, use Screen Time to prevent bypass:

1. On the child's device, go to **Settings > Screen Time**.
2. Set a **Screen Time Passcode** (different from the device passcode — only you know this).
3. Under **Content & Privacy Restrictions**, enable restrictions and configure:
   - **iTunes & App Store Purchases > Deleting Apps** → Don't Allow (prevents uninstalling WARP)
   - **Allowed Apps** → Consider disabling access to **Settings** changes if appropriate
4. Under **Content & Privacy Restrictions > Content Restrictions**:
   - Review and restrict as appropriate for the child's age
5. The combination of WARP (locked switch) + Screen Time (can't delete app) means the child cannot remove the filtering.

#### Additional iOS hardening

- If the child knows the device passcode, they could potentially remove the VPN profile from **Settings > General > VPN & Device Management**. To prevent this, use Screen Time to restrict access or consider Apple Configurator supervision for stronger control.
- **Apple Configurator** (free Mac app) can put the device into **Supervised Mode**, which gives you much tighter control including preventing VPN profile removal. This requires wiping the device, so do it during initial setup.

---

### Android

#### Install the app

1. On the child's device, open the **Google Play Store**.
2. Search for **"Cloudflare One Agent"** and install it.
3. Open the app.
4. Tap the menu, go to **Settings > Account > Login to Zero Trust**.
5. Enter your team name (e.g. `hunter-family`).
6. The app will request VPN permissions — approve.
7. Verify the connection is active.

#### Lock it down with Google Family Link

1. If not already set up, install **Google Family Link** on your (parent) device.
2. Add the child's device/account to Family Link.
3. In Family Link settings:
   - **App management** → Require parent approval for all app installs (prevents installing VPN bypass apps)
   - **App restrictions** → Ensure Cloudflare One Agent cannot be uninstalled
   - **Device settings** → Restrict changes to network/VPN settings if available
4. The combination of WARP (locked switch) + Family Link (can't uninstall, can't install bypass apps) provides robust protection.

#### Important Android note

Google Family Link restricts the installation of root CA certificates on child accounts. This is fine for DNS-only mode (no certificate needed), but means you cannot use HTTP inspection mode on Family Link managed Android devices. DNS-only mode is the correct choice here.

---

### Windows

#### Install WARP

**Option A — Silent install with MDM parameters (recommended):**

Open an **Administrator** Command Prompt or PowerShell and run:

```
msiexec /i "Cloudflare_WARP_<VERSION>.msi" /qn ORGANIZATION="hunter-family" SERVICE_MODE="1dot1" SWITCH_LOCKED="True" AUTO_CONNECT="0"
```

Download the MSI from the [Cloudflare WARP downloads page](https://developers.cloudflare.com/warp-client/get-started/windows/).

Replace `<VERSION>` with the actual version number of the downloaded file.

The parameters:
- `ORGANIZATION` — your Zero Trust team name
- `SERVICE_MODE` — set to `1dot1` for DNS-only mode
- `SWITCH_LOCKED` — prevents the user from disconnecting
- `AUTO_CONNECT` — reconnects immediately if disconnected

**Option B — Manual install:**

1. Download and run the WARP installer.
2. After installation, open WARP.
3. Go to **Settings > Account > Login to Zero Trust**.
4. Enter your team name.
5. The locked switch and service mode settings will be applied from the dashboard profile.

#### Lock it down on Windows

1. Ensure the child's Windows account is a **Standard User** (not Administrator).
   - Go to **Settings > Accounts > Family & other users**.
   - The child's account type should be "Standard User".
2. As a standard user, the child cannot:
   - Uninstall WARP (requires admin)
   - Stop the Cloudflare WARP service (requires admin)
   - Edit the MDM configuration at `C:\ProgramData\Cloudflare\mdm.xml` (requires admin)
   - Change system DNS settings (requires admin)
3. Optionally, set up **Microsoft Family Safety** for additional controls (screen time, app restrictions, activity reports).

#### MDM configuration file (Windows)

If you need to adjust settings after install, edit `C:\ProgramData\Cloudflare\mdm.xml` as Administrator:

```xml
<dict>
  <key>organization</key>
  <string>hunter-family</string>
  <key>service_mode</key>
  <string>1dot1</string>
  <key>switch_locked</key>
  <true/>
  <key>auto_connect</key>
  <integer>0</integer>
  <key>onboarding</key>
  <false/>
</dict>
```

The WARP client picks up changes to this file automatically.

---

## Step 6 — Verify Everything Is Working

After setting up each device:

1. **Check connection**: Open the Cloudflare One Agent app on the device. It should show as connected.
2. **Test a blocked site**: Try visiting a site that should be blocked (e.g. a gambling site). You should see a block page or a connection refused error.
3. **Test SafeSearch**: Go to Google and search for something that would normally return explicit results. SafeSearch should be enforced — you should see filtered results only.
4. **Check logs**: Go to **Insights > Logs > DNS** in the Zero Trust dashboard. You should see DNS queries from the device, including any blocked requests.
5. **Test the lock**: Try to disconnect WARP from the child's device. It should not allow disconnection.
6. **Test uninstall prevention**: Try to delete/uninstall the WARP app from the child's device. Screen Time (iOS), Family Link (Android), or standard user restrictions (Windows) should prevent this.

---

## Ongoing Management

### Adding or removing blocked sites

1. Log in to the [Zero Trust dashboard](https://one.dash.cloudflare.com).
2. Go to **Gateway > Firewall Policies > DNS**.
3. Edit your policies to add/remove domains or categories.
4. Changes take effect within minutes — no action needed on the devices.

### Checking activity

1. Go to **Insights > Logs > DNS** in the dashboard.
2. You can see all DNS queries, filtered by device, time, and whether they were allowed or blocked.
3. Free tier retains logs for 24 hours.

### Temporarily disabling filtering

If a legitimate site is blocked and the child needs access:

1. In the dashboard, go to **Settings > WARP Client**.
2. Use the **Admin override code** (rotates hourly) to temporarily disable WARP on the specific device.
3. The client will automatically reconnect after the timeout you configured.

Alternatively, add the site to an **Allow** policy in Gateway so it bypasses your block rules permanently.

### Updating the WARP client

The client auto-updates on most platforms. You can control update behaviour in the dashboard under **Settings > WARP Client > Auto-update**.

---

## Limitations and Gaps

| Limitation | Detail | Mitigation |
|------------|--------|------------|
| Encrypted messaging apps | Content inside WhatsApp, Signal, etc. is not inspectable | Block the app domains if you want to prevent access entirely, or use platform parental controls for app-level restrictions |
| Friend's device | No protection on devices you don't control | Parenting conversation |
| Factory reset | Wipes WARP and all settings | iOS: enable Activation Lock (tied to your Apple ID). Android: enable Factory Reset Protection. Both require your credentials to set up the device again after reset |
| Web-based proxies | Some proxy sites may not be categorised | The "Proxy/Anonymiser" content category blocks known proxy sites. Monitor logs for unusual domains |
| DNS-only mode gaps | Cannot inspect full URLs, only domain names | Sufficient for parental controls — you block entire domains, not individual pages. SafeSearch handles search engine filtering |
| Creative DNS bypass | A knowledgeable child could try to manually set DNS to 8.8.8.8 | WARP overrides system DNS settings while active. As long as WARP is locked on, DNS bypass is not possible |
| Tor Browser | Routes traffic through the Tor network, bypassing DNS filtering | Block the Tor Project domain. Prevent installation of new apps via Screen Time / Family Link. On Windows, standard user account prevents installing Tor |

---

## Quick Reference

| Item | Value |
|------|-------|
| Dashboard URL | https://one.dash.cloudflare.com |
| Team name | `hunter-family` (replace with yours) |
| Free tier limit | 50 users |
| Service mode | DNS only |
| Log retention | 24 hours (free tier) |
| WARP app name (App Store / Play Store) | Cloudflare One Agent |
| Windows config file | `C:\ProgramData\Cloudflare\mdm.xml` |
| Admin override | Hourly rotating code, found in dashboard under Settings > WARP Client |
