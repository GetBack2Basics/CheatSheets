# Windows Network Machine Improvement Checklist

## Summary

Optimize Windows 11 network reliability, DNS behavior, and power/background-app stability to reduce connection drops during heavy agent workflows and prevent background tasks from stalling. This is the Windows 11 equivalent of the Linux checklist — same goals, different tools.

> Tested on: Windows 11 25H2 (build 26200), MediaTek Wi-Fi 6E RZ616 (RZ616) PCIe adapter, Ethernet absent (Wi-Fi only). Some commands below behaved differently on this OEM build than on stock Windows — see **Troubleshooting** before assuming a failure.

## Discussion

Long-running model pulls and continuous agent activity expose weak defaults in Wi-Fi power behavior, DNS resolution, and power-management profiles. On the reference machine the router's DNS resolver was taking ~2,265 ms per lookup versus ~30 ms for public resolvers — a **~75× slowdown** paid by every `git`, `pip`, API call, and browser navigation. The same classes of fix that help on Ubuntu help here, but the tooling is PowerShell/CIM (`netsh`, `powercfg`, `Set-NetAdapter*`) rather than NetworkManager/systemd-resolved.

Unlike the Linux box, this machine has **no Google Chrome installed** — it runs **Microsoft Edge** (and the Hermes desktop app itself runs on Edge WebView2, hence dozens of `msedge.exe` processes). Browser resource flags are therefore applied to **Edge**, not Chrome. If you do have Chrome, the equivalent registry path is noted in Step 5.

## Requirements

- Windows 10/11 with administrator rights (most writes need an **elevated** PowerShell — "Access is denied" otherwise)
- PowerShell 5.1+ (or PowerShell 7)
- Network adapter names resolved first (`Get-NetAdapter`)
- For DNS timing checks: `Resolve-DnsName` (built in) or `nslookup`

## Workflow

### Step 1: Disable aggressive Wi-Fi power saving

Windows can drop the Wi-Fi radio into power-save to save battery, which causes instability for persistent sessions and long API calls — exactly the Linux Step 1 problem.

```powershell
# Run in an ELEVATED PowerShell
# First confirm the adapter name and the valid values:
Get-NetAdapterAdvancedProperty -Name 'Wi-Fi' -DisplayName 'Power Saving'

# Disable it:
Set-NetAdapterAdvancedProperty -Name 'Wi-Fi' -DisplayName 'Power Saving' -DisplayValue 'Disabled' -NoRestart

# Verify:
(Get-NetAdapterAdvancedProperty -Name 'Wi-Fi' -DisplayName 'Power Saving').DisplayValue
# -> Disabled
```

> The valid `DisplayValue`s on the RZ616 are `Disabled` / `Auto`. On some machines the value is named differently (e.g. "Power Saving Mode"); substitute the real `DisplayName` from the first command.

### Step 2: Set public DNS resolvers (IPv4 + IPv6)

Switch off the slow router resolver to Cloudflare/Google. This is the Windows equivalent of the Linux Step 3 systemd-resolved change.

```powershell
# Run in an ELEVATED PowerShell
$wifiIdx = (Get-NetAdapter -Name 'Wi-Fi').ifIndex

# IPv4 (Set-DnsClientServerAddress works on most builds):
Set-DnsClientServerAddress -InterfaceIndex $wifiIdx -ServerAddresses ('1.1.1.1','8.8.8.8')

# IPv6 — on some 25H2 builds Set-DnsClientServerAddress does NOT accept -AddressFamily.
# Use netsh instead (pre-check reachability first):
Resolve-DnsName github.com -Server '2606:4700:4700::1111' -ErrorAction SilentlyContinue
netsh interface ipv6 set dns name="Wi-Fi" static 2606:4700:4700::1111
netsh interface ipv6 add dns  name="Wi-Fi" 2606:4700:4700::1001 index=2

# Verify:
Get-DnsClientServerAddress -InterfaceIndex $wifiIdx -AddressFamily IPv4 | Select-Object ServerAddresses
Get-DnsClientServerAddress -InterfaceIndex $wifiIdx -AddressFamily IPv6 | Select-Object ServerAddresses
```

Quick timing check (compare before/after):

```powershell
foreach ($s in @('192.168.1.1','1.1.1.1','8.8.8.8')) {
  $t = Measure-Command { Resolve-DnsName github.com -Server $s -ErrorAction SilentlyContinue | Out-Null }
  Write-Host "$s -> $([math]::Round($t.TotalMilliseconds,1)) ms"
}
```

### Step 3: Use the Ultimate Performance power scheme

On a plugged-in desktop/laptop you don't want Windows capping clocks for "balance." Note: the classic **High Performance** GUID (`8c5e7fda-...`) is **often absent on OEM Windows 11 builds** — `powercfg /setactive` to it silently no-ops. Use **Ultimate Performance** instead.

```powershell
# Run in an ELEVATED PowerShell
$upGuid = 'e9a42b02-d5df-448d-aa00-03f14749eb61'

# duplicatescheme mints a NEW guid each time it succeeds — capture the returned GUID:
$dup = powercfg -duplicatescheme $upGuid
# -> "Power Scheme GUID: <NEW-GUID>  (Ultimate Performance)"

# Extract the new GUID and activate it:
if ($dup -match 'GUID:\s*([0-9a-f\-]+)') { $new = $Matches[1] } else { $new = $upGuid }
powercfg /setactive $new

# Verify:
powercfg /getactivescheme
```

To revert: `powercfg /setactive 381b4222-f694-41f0-9685-ff5bb260df2e` (Balanced).

### Step 4: Remove stale/duplicate network adapters (bloat & route noise)

Leftover VPN TAP adapters (e.g. a disconnected McAfee/OpenVPN tap) add route-table noise and needless startup weight — the Windows analog of pruning unused interfaces.

```powershell
# List all adapters and their state:
Get-NetAdapter | Select-Object Name,Status,InterfaceDescription

# Disable a lingering, disconnected adapter (do NOT disable your real Wi-Fi/Ethernet!):
Disable-NetAdapter -Name 'McAfee VPN' -Confirm:$false

# Verify:
Get-NetAdapter -Name 'McAfee VPN' | Select-Object Name,Status
# -> Disabled
```

### Step 5: Browser resource flags (Edge/Chrome low-end device mode)

The Linux checklist forces Chromium flags (`--enable-low-end-device-mode`, `--js-flags="--max-old-space-size=2048"`, `--process-per-site`) via a wrapper script / `.desktop` patch. On Windows there is no wrapper script; the equivalent is a **Group Policy registry key** that injects launch parameters into every browser start.

This reference machine runs **Edge**, so apply to Edge. (If you actually have Chrome, swap the registry path to `HKLM:\SOFTWARE\Policies\Google\Chrome`.)

```powershell
# Run in an ELEVATED PowerShell
$flags = '--enable-low-end-device-mode --js-flags="--max-old-space-size=2048" --process-per-site'

# EDGE:
$key = 'HKLM:\SOFTWARE\Policies\Microsoft\Edge'
if (-not (Test-Path $key)) { New-Item -Path $key -Force | Out-Null }
Set-ItemProperty -Path $key -Name 'AdditionalLaunchParameters' -Value $flags -Type String

# CHROME (only if Chrome is installed):
# $key = 'HKLM:\SOFTWARE\Policies\Google\Chrome'
# (same Set-ItemProperty call)

# Verify:
(Get-ItemProperty -Path $key -Name 'AdditionalLaunchParameters').AdditionalLaunchParameters
```

Takes effect on **next browser launch**. Confirm at `edge://version` (or `chrome://version`) — the flags should appear in the "Command Line" / active-args list. To undo, delete the `AdditionalLaunchParameters` value (`Remove-ItemProperty`).

### Step 6 (optional): Latency/chaos testing

Linux Step 4 uses `tc netem` to inject delay. Windows has no `tc`; use **Clumsy** (free, https://jagt.github.io/clumsy/) to add latency/loss/drop to a chosen interface and validate that agent retry/timeout logic holds up. No system change required — purely a test harness.

## Verification

- Wi-Fi `Power Saving` reports `Disabled` and SSH/remote sessions + long model pulls stay connected.
- `Get-DnsClientServerAddress` shows public resolvers (1.1.1.1 / 8.8.8.8 / Cloudflare v6) instead of the router.
- DNS query time drops from seconds (router) to tens of ms (public) — measured with the `Measure-Command`/`Resolve-DnsName` loop above.
- `powercfg /getactivescheme` reports **Ultimate Performance**.
- Stale VPN adapters report `Disabled`.
- Browser `edge://version` (or `chrome://version`) lists the injected resource flags after restart.

## Troubleshooting

- **"Access is denied" / "Access to a CIM resource was not available"** on `Set-NetAdapter*` or `Set-DnsClientServerAddress`:
  You are not elevated. Re-launch PowerShell as Administrator (or `Start-Process powershell -Verb RunAs`).
- **`Set-DnsClientServerAddress : A parameter cannot be found that matches parameter name 'AddressFamily'`** (IPv6):
  Some 25H2 builds dropped that parameter. Use `netsh interface ipv6 set dns ...` / `add dns ...` instead (Step 2).
- **`powercfg -duplicatescheme` returns "Unable to create a new power scheme" for the High Performance GUID**:
  High Performance isn't shipped on this OEM image. Use the **Ultimate Performance** GUID (`e9a42b02-d5df-448d-aa00-03f14749eb61`) — but note `duplicatescheme` mints a *fresh* GUID each run, so you must capture the returned GUID and `powercfg /setactive` to **that**, not to the canonical one.
- **Chrome flags didn't take effect / no Chrome installed**:
  The machine may run Edge, not Chrome. Apply the registry key under `...\Policies\Microsoft\Edge` instead, and confirm via `edge://version`.
- **Browser flags applied but not visible**:
  Fully quit the browser (Taskbar + tray) and reopen. WebView2-hosted apps (e.g. some desktop wrappers) read the policy at process start.

## Code Sample

```powershell
# Run the whole block in an ELEVATED PowerShell

# --- Step 1: Wi-Fi power saving off ---
Set-NetAdapterAdvancedProperty -Name 'Wi-Fi' -DisplayName 'Power Saving' -DisplayValue 'Disabled' -NoRestart

# --- Step 2: public DNS (IPv4 via cmdlet, IPv6 via netsh) ---
$wifiIdx = (Get-NetAdapter -Name 'Wi-Fi').ifIndex
Set-DnsClientServerAddress -InterfaceIndex $wifiIdx -ServerAddresses ('1.1.1.1','8.8.8.8')
netsh interface ipv6 set dns name="Wi-Fi" static 2606:4700:4700::1111
netsh interface ipv6 add dns  name="Wi-Fi" 2606:4700:4700::1001 index=2

# --- Step 3: Ultimate Performance (capture the minted GUID) ---
$dup = powercfg -duplicatescheme e9a42b02-d5df-448d-aa00-03f14749eb61
if ($dup -match 'GUID:\s*([0-9a-f\-]+)') { $new = $Matches[1] } else { $new = 'e9a42b02-d5df-448d-aa00-03f14749eb61' }
powercfg /setactive $new

# --- Step 4: disable a lingering VPN adapter (check name first!) ---
Disable-NetAdapter -Name 'McAfee VPN' -Confirm:$false

# --- Step 5: Edge resource flags ---
$key = 'HKLM:\SOFTWARE\Policies\Microsoft\Edge'
if (-not (Test-Path $key)) { New-Item -Path $key -Force | Out-Null }
Set-ItemProperty -Path $key -Name 'AdditionalLaunchParameters' `
  -Value '--enable-low-end-device-mode --js-flags="--max-old-space-size=2048" --process-per-site' -Type String

# --- Verify ---
(Get-NetAdapterAdvancedProperty -Name 'Wi-Fi' -DisplayName 'Power Saving').DisplayValue
Get-DnsClientServerAddress -InterfaceIndex $wifiIdx | Select-Object AddressFamily,ServerAddresses
powercfg /getactivescheme
(Get-ItemProperty -Path $key -Name 'AdditionalLaunchParameters').AdditionalLaunchParameters
```
