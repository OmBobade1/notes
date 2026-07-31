# Rooting an Android Device

## What "root" actually means
On Android (built on Linux), the "root" user is the superuser account with unrestricted access to the entire filesystem and every system function — the same concept as `root` on any Linux system, or Administrator on Windows. Android devices ship with root access disabled for the end user by default, specifically to enforce the app sandboxing model covered in file `01`. Rooting is the process of deliberately gaining that superuser access back.

## Why a security tester needs a rooted device
Several categories of testing are simply impossible on a stock, non-rooted device:
- Bypassing an app's own root-detection logic to test what it does when that check is defeated
- Reading another app's private data directory (`/data/data/<package>/`) to inspect how it stores things locally
- Installing a CA certificate at the system trust store level (needed to intercept traffic from apps that don't trust user-installed certificates by default — see file `03`)
- Using tools like Frida in non-embedded mode, which need root to attach to and instrument another app's running process

## Types of root, and the real differences between them

### 1. Systemless Root (Magisk) — the modern standard
**What it is:** Instead of directly modifying the read-only system partition, Magisk intercepts the boot process and applies root access as an overlay, without ever touching `/system` itself.

**Why it's the standard now:** Because the system partition stays untouched, apps that check the system partition's integrity (many banking apps do exactly this) have a harder time detecting it compared to older root methods. Magisk also supports **modules** — installable packages that extend functionality (e.g. Frida server auto-start, systemless Xposed framework support) — and includes **MagiskHide/Zygisk**, which selectively hides root status from specific apps that request it, letting you keep root for testing while still running banking/payment apps that refuse to launch on a detected-rooted device.

**How it's done (high-level):**
1. Unlock the device's bootloader (varies by manufacturer; typically `fastboot flashing unlock`) — this itself wipes the device and is detectable by some apps independently of root.
2. Extract the device's stock `boot.img` (from the manufacturer's firmware package).
3. Patch that `boot.img` using the Magisk app.
4. Flash the patched image back via `fastboot flash boot patched_boot.img`.
5. Reboot — Magisk is now active, granting root selectively to apps that request it via the Magisk app's permission prompt.

### 2. SuperSU — the older standard (largely legacy now)
**What it is:** An older root management approach, historically the default before Magisk became dominant. Directly modifies system files in some configurations, making it more easily detected and generally considered obsolete for modern testing due to weaker detection-evasion and lack of active maintenance.

### 3. KernelSU — a newer, kernel-level alternative
**What it is:** Grants root at the kernel level via a custom kernel module, rather than patching the boot image the way Magisk does. Requires a custom kernel to be built or provided by the device's ROM, so it's less universally applicable than Magisk (which works against the stock kernel), but offers a different detection profile — some root-detection logic specifically checks for Magisk's known artifacts and won't catch KernelSU the same way.

### 4. Full/System Root (older, direct modification)
**What it is:** Directly writing su binaries into the system partition itself (`/system/xbin/su` or similar), the original rooting approach before systemless methods existed. Trivially detected by any app that checks for the presence of a `su` binary in common paths or verifies system partition integrity — essentially obsolete for serious testing today, but worth knowing as the historical baseline every later method improved on.

## Quick comparison

| Method | Modifies /system? | Detection difficulty (for the app) | Current relevance |
|---|---|---|---|
| Full/System Root | Yes | Easy to detect | Legacy/obsolete |
| SuperSU | Partially, depending on config | Moderate | Legacy |
| Magisk (systemless) | No | Harder — requires deeper checks | Current standard |
| KernelSU | No (kernel-level instead) | Different signature than Magisk | Emerging alternative |

## Why this matters for testing banking apps specifically
Banking apps are exactly the category most likely to implement root detection (covered in a later file in this sequence), so understanding *which* rooting method you're using — and what specific artifacts each leaves behind — directly determines whether you can even get the app running in a testable state at all. A tester who only knows "how to root" without understanding detection differences between methods will hit a wall the first time an app refuses to launch, with no idea why.

## Ethical and safety notes
- Rooting voids most manufacturer warranties and can brick a device if done incorrectly — always use a dedicated test device, never a primary personal phone.
- Only root devices you own or have explicit authorization to test.
- Keep a full backup/stock firmware image available before starting, so the device can always be restored to factory state.
