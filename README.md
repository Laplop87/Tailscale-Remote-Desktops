```markdown
# 🖥️ Remote Desktops via Tailscale on GitHub Actions ☁️

This repository provides two GitHub Actions workflows that spin up **fresh, on-demand remote desktop sessions** — one Windows, one Ubuntu — both accessible securely over your **private Tailscale network**. No RDP/VNC ports are ever exposed to the public internet. 🛡️

Whenever you need a temporary desktop, launch a workflow, grab the Tailscale IP from the log, connect with your preferred client, and use the session for up to **6 hours**.

---

## 📦 What's in This Repo

| Workflow File | OS | Protocol | Client on Windows 11 |
| :--- | :--- | :--- | :--- |
| `windows-rdp-tailscale.yml` | Windows Server (latest) | RDP | **Remote Desktop Connection** (`mstsc`) |
| `ubuntu-rdp-tailscale.yml` | Ubuntu + XFCE | VNC | **RealVNC Viewer** |

Both workflows share the same `TAILSCALE_AUTH_KEY` secret and the same Tailscale account. You only configure it once.

---

## 📋 Prerequisites

- A **GitHub account**
- A **Tailscale account** (free personal plan is fine)
- **Tailscale client** installed on your Windows 11 PC — download: https://tailscale.com/download/windows
- For **Windows RDP**: Remote Desktop Connection (`mstsc`) — built into Windows 11
- For **Ubuntu VNC**: RealVNC Viewer — download: https://www.realvnc.com/en/connect/download/viewer/

---

## 🔑 One-Time Setup (Shared by Both Workflows)

### Step 1: Generate a Tailscale Auth Key

1. Log in to Tailscale and open the **Keys** page: https://login.tailscale.com/admin/settings/keys
2. Click **Generate auth key**
3. Recommended settings: **Reusable = ON**, **Ephemeral = OFF**
4. Copy the key (starts with `tskey-auth-...`)

### Step 2: Add the Key to GitHub Secrets

1. In your repository, go to **Settings → Secrets and variables → Actions**
2. Click **New repository secret**
3. Name: `TAILSCALE_AUTH_KEY`
4. Paste your Tailscale auth key as the value
5. Click **Add secret**

That's the only secret required for both workflows.

### Step 3: Confirm Tailscale Is Running on Your Windows 11 PC

Before connecting to any session, make sure the Tailscale client on your PC is **logged in to the same Tailscale account** you used to generate the auth key. Without this, you won't be able to reach `100.x.x.x` addresses.

Quick check in PowerShell:

```powershell
tailscale status
```

You should see a list of devices — your GitHub runner will appear there while a session is active.

---

## 🖥️ Option 1: Windows RDP

### Run the Workflow

1. Go to the **Actions** tab of this repository
2. Select **Pure Windows RDP via Tailscale** in the left sidebar
3. Click **Run workflow → Run workflow** (green button)
4. Wait ~2–3 minutes for the runner to start and configure itself

### Get the Connection Details

Open the running job → expand the **Display Connection Details** step. You'll see:

```
================ RDP IS READY ================
IP Address: 100.XX.XX.XX
Username  : runneradmin
Password  : WindowsRDP#2025
==============================================
```

Copy the **IP address**.

### Connect from Windows 11

1. Press **Win + R**, type `mstsc`, hit **Enter**
2. In **Computer**, enter the Tailscale IP: `100.XX.XX.XX`
3. Click **Connect**
4. When prompted:
   - **Username:** `runneradmin`
   - **Password:** `WindowsRDP#2025`
5. Accept the certificate warning
6. You're on the Windows desktop 🎉

**Tips:**
- Use `mstsc /v:100.XX.XX.XX /cert-ignore` to skip the certificate warning next time
- Save the session as an `.rdp` file if you'll reconnect to the same IP (rare — the IP changes each run)

---

## 🐧 Option 2: Ubuntu VNC (XFCE Desktop)

### Run the Workflow

1. Go to the **Actions** tab of this repository
2. Select **Pure Ubuntu RDP via Tailscale** in the left sidebar
3. Click **Run workflow → Run workflow**
4. Wait ~3–4 minutes (XFCE installation is a bit longer than Windows)

### Get the Connection Details

Open the running job → expand the **Display Connection Details** step. You'll see:

```
================ VNC IS READY ================
IP Address : 100.XX.XX.XX
Port       : 5901
Password   : vnc2025
==============================================
```

Copy the **IP address**.

### Connect from Windows 11

1. Open **RealVNC Viewer**
2. In the address bar, enter: `100.XX.XX.XX:5901`
3. Press **Enter**
4. **If prompted about unencrypted connection** → click **Continue / Yes**
   - This is expected. The connection is already encrypted end-to-end by Tailscale's WireGuard tunnel — the VNC layer itself doesn't add a second encryption layer, and that's fine here.
5. **Password:** `vnc2025`
6. You're on the XFCE desktop 🎉

**Tips:**
- To open a terminal: **Applications → Terminal Emulator**, or right-click the desktop → **Open Terminal Here**
- VNC passwords in TigerVNC are **limited to 8 characters** — `vnc2025` fits

---

## ⏱️ Session Lifetime & Reconnects

- Each session runs for **up to 6 hours** (a `sleep 21600` step keeps the runner alive)
- When the job ends, the runner is destroyed and the Tailscale IP becomes invalid
- To start a new session: **Actions → pick the workflow → Run workflow** again
- **The Tailscale IP changes every run** — always grab the fresh IP from the new run's log

To end a session early, open the running workflow in **Actions** and click **Cancel workflow**.

---

## ⚠️ Important Notes

### Tailscale Must Stay Running on Your PC

If you close the Tailscale client or log out of your account, the connection to `100.x.x.x` will drop immediately. Keep Tailscale running while using the session.

### Session Is Ephemeral

Anything you save on the Windows or Ubuntu desktop is lost when the workflow ends. **Do not store sensitive data, passwords, or long-term work in these sessions** — treat them as throwaway environments.

### Security & Privacy

- The RDP (port 3389) and VNC (port 5901) ports are **never exposed to the public internet**. Only devices on your Tailscale tailnet can reach them.
- The default passwords (`WindowsRDP#2025`, `vnc2025`) are **hardcoded in the workflows** and visible to anyone who has access to this repo. If you share the repo publicly, **fork it and change the passwords** before use.
- For additional security, set the Tailscale ACLs to restrict which devices can reach these runners.

### macOS Support

A macOS workflow was tested and **is not viable on GitHub-hosted runners**. macOS runners are headless, so Screen Sharing captures a black framebuffer, and the `runner` account's secure-token prevents password changes needed for VNC auth. If you genuinely need an interactive macOS desktop, use a paid service like MacStadium or MacRun.

---

## 🛠️ Troubleshooting

| Symptom | Fix |
| :--- | :--- |
| **Job sits in "Waiting for a runner" for 10+ min** | GitHub capacity issue — cancel and retry, or try during off-peak hours |
| **"Connection refused" from `mstsc` or RealVNC** | Tailscale on your PC isn't connected — run `tailscale status` and log in |
| **`mstsc` says "credentials did not work"** | Passwords are case-sensitive. `runneradmin` / `WindowsRDP#2025` |
| **VNC shows "authentication failed"** | VNC password is `vnc2025` (no username field). If your client asks for a username, leave it blank |
| **VNC shows a black screen** | The XFCE session didn't start — check the "Install XFCE" step for errors in the run log |
| **Connection drops mid-session** | The 6-hour limit was reached, or you cancelled the workflow |
| **Session ends and IP changes** | Expected — re-run the workflow and use the new IP |

---

## 📌 Quick Reference

| | Windows RDP | Ubuntu VNC |
| :--- | :--- | :--- |
| **Workflow** | `windows-rdp-tailscale.yml` | `ubuntu-rdp-tailscale.yml` |
| **Client (Win 11)** | `mstsc` | RealVNC Viewer |
| **Username** | `runneradmin` | *(leave blank)* |
| **Password** | `WindowsRDP#2025` | `vnc2025` |
| **Port** | 3389 (via RDP) | 5901 (VNC) |
| **Session time** | ~6 hours | ~6 hours |
| **Setup time** | ~2–3 min | ~3–4 min |

---

## 🙏 Attribution

The `windows-rdp-tailscale.yml` workflow is based on the pattern from the original [Original-Windows-11-Tailscale](https://github.com/ditzz-dev/Original-Windows-11-Tailscale) repository by [@ditzz-dev](https://github.com/ditzz-dev), adapted for clarity and renamed workflow files.

The `ubuntu-rdp-tailscale.yml` workflow was built from scratch, incorporating community-known fixes for TigerVNC 1.15+ (XDG path migration, password truncation, D-Bus session handling for XFCE).

The Tailscale connection uses the official [tailscale/github-action](https://github.com/tailscale/github-action).
```

### 🎯 What Changed vs. the Original README

| Section | Original | Rewritten |
| :--- | :--- | :--- |
| **Scope** | Windows-only | Windows **and** Ubuntu, side by side |
| **Setup** | Windows-specific | Unified — one `TAILSCALE_AUTH_KEY` secret for both |
| **Prerequisites** | Implicit | Explicit — includes RealVNC Viewer download link |
| **Connection instructions** | RDP only | RDP (`mstsc`) **and** VNC (RealVNC) with exact addresses and credentials |
| **Troubleshooting** | None | New table covering both workflows |
| **Quick Reference** | None | Side-by-side comparison table |
| **macOS** | Not mentioned | Documented as tested-but-not-viable (so future-you doesn't retry) |
| **Attribution** | Original only | Both the original author **and** the Ubuntu work |

### 📝 Small Edit Before You Save

If you've already renamed the repo to `tailscale-remote-desktops` (or similar), the README's internal links work as-is. If not, the README doesn't reference the repo name anywhere critical, so it works either way.

You can also delete the file `🖥️ Pure Windows RDP via Tailscale on GitHub.txt` from the repo root — it was just a scratch copy. The canonical README is `README.md`. 🚀
