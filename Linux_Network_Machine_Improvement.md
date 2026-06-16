# Linux Network Machine Improvement Checklist

## Summary

Optimize Ubuntu network reliability, DNS behavior, and shell environment stability to reduce connection drops during heavy agent workflows and prevent background tasks from stalling.

## Discussion

Long-running model pulls and continuous agent activity can expose weak defaults in Wi-Fi power behavior, DNS resolution, and shell startup configuration. The workflow below prioritizes resilient connectivity and repeatable diagnostics.

## Requirements

- Ubuntu system with sudo access
- NetworkManager in use
- Terminal access
- `dig` available for DNS timing checks

## Workflow

### Step 1: Disable aggressive Wi-Fi power saving (laptops)

Ubuntu can reduce Wi-Fi radio performance to save power, which often causes instability for persistent SSH sessions and long API calls.

```bash
sudo nano /etc/NetworkManager/conf.d/default-wifi-powersave-on.conf
```

Set:

```ini
[connection]
wifi.powersave = 2
```

Apply:

```bash
sudo systemctl restart NetworkManager
```

### Step 2: Force IPv4 for apt when downloads hang

If package downloads stall or become extremely slow due to bad IPv6 routing, force IPv4 for apt.

```bash
sudo nano /etc/apt/apt.conf.d/99force-ipv4
```

Add:

```plaintext
Acquire::ForceIPv4 "true";
```

Test:

```bash
sudo apt update
```

### Step 3: Update DNS for resolver consistency

Switch to reliable public DNS resolvers to reduce lookup failures for API and Git endpoints.

```bash
sudo nano /etc/systemd/resolved.conf
```

Set:

```ini
DNS=1.1.1.1 8.8.8.8
```

Apply:

```bash
sudo systemctl restart systemd-resolved
```

Optional quick check:

```bash
dig github.com | grep "Query time"
```

### Step 4: Simulate latency to test agent resilience

Use traffic control to validate retry behavior and timeout handling under artificial delay.

Add 100 ms delay:

```bash
sudo tc qdisc add dev eth0 root netem delay 100ms
```

Remove delay:

```bash
sudo tc qdisc del dev eth0 root
```

Find your real interface first if `eth0` is not present:

```bash
ip a
```

### Step 5: Keep shell hooks at the end of startup files

For Hermes-like workflows, `direnv` or Conda hooks should be placed at the bottom of `~/.bashrc` or `~/.zshrc` so they are not overridden later in initialization.

Example:

```bash
eval "$(direnv hook bash)"
```

Reload shell:

```bash
source ~/.bashrc
```

## Verification

- SSH and remote terminal sessions remain stable during long model downloads.
- `apt update` no longer hangs when the network path has poor IPv6 behavior.
- DNS query time is consistently low and endpoint resolution failures decrease.
- Agent scripts continue running under induced latency without silent stalls.
- Environment tools initialize correctly in new shell sessions.

## Troubleshooting

- DNS changes do not stick:
	Confirm `systemd-resolved` is active and no other network manager is overriding resolver settings.
- `tc` command fails:
	Install `iproute2` and run with sudo.
- Wi-Fi instability remains:
	Confirm the power-save file exists and that NetworkManager controls the adapter.
- Shell hooks still not loading:
	Check for early `return` statements in startup files and move hook lines to the end.

## Code Sample

```bash
# Wi-Fi power save tuning
sudo nano /etc/NetworkManager/conf.d/default-wifi-powersave-on.conf
sudo systemctl restart NetworkManager

# Force IPv4 for apt
echo 'Acquire::ForceIPv4 "true";' | sudo tee /etc/apt/apt.conf.d/99force-ipv4
sudo apt update

# DNS override via systemd-resolved
sudo nano /etc/systemd/resolved.conf
sudo systemctl restart systemd-resolved

# Latency simulation (replace eth0 with your interface)
sudo tc qdisc add dev eth0 root netem delay 100ms
sudo tc qdisc del dev eth0 root
```

On Linux, the standard way to enforce system-wide defaults for Chromium-based browsers is by modifying the browser's global environmental wrapper script.Step 1: Update the Global Wrapper FlagsChrome reads a default configuration wrapper script located in /opt. Open the wrapper configuration using your preferred terminal text editor (like nano or vim) with sudo:bashsudo nano /opt/google/chrome/google-chrome
Use code with caution.Scroll all the way to the very bottom of that file. You will see a line that looks exactly like this:bashexec -a "$0" "$HERE/chrome" "$@"
Use code with caution.Modify that exact line to append your resource-saving flags right before the trailing arguments. Change it to:bashexec -a "$0" "$HERE/chrome" --enable-low-end-device-mode --js-flags="--max-old-space-size=2048" --process-per-site "$@"
Use code with caution.Save and exit the editor (in nano, press Ctrl+O, Enter, then Ctrl+X).Step 2: Enforce a Global Systemd Hard RAM Cap (Optional but Recommended)Because you have root access, you can ensure that Chrome never crashes your entire OS if a massive spatial script or AI tool leaks memory. You can configure systemd to automatically sandbox the Chrome application binary slice.Run this command to create a global systemd override for the desktop display manager:bashsudo mkdir -p /etc/systemd/system/user-.slice.d/
Use code with caution.Now, create a global resource configuration file:bashsudo nano /etc/systemd/system/user-.slice.d/chrome-limit.conf
Use code with caution.Paste the following lines into it. This ensures that any graphical application launched by any user is restricted from completely exhausting the system's 16GB swap and RAM layout:ini[Slice]
MemoryHigh=12G
MemoryMax=14G
Use code with caution.Save and close the file, then tell systemd to reload its configuration:bashsudo systemctl daemon-reload
Use code with caution.
Step 1: Force System-Wide Environment FlagsCreate a dedicated configurations file that Chrome checks on every boot. Run this command with root privileges:bashsudo nano /etc/default/google-chrome
Use code with caution.(If the file is blank, that is completely normal).Paste the exact configuration line below into the file:bashexport CHROMIUM_USER_FLAGS="--enable-low-end-device-mode --js-flags=\"--max-old-space-size=2048\" --process-per-site"
Use code with caution.Save and close the file (Ctrl+O, Enter, then Ctrl+X if using nano).Step 2: The Direct Binary Symlink Patch (If Step 1 fails)If your specific Linux distribution's package script ignores the /etc/default file, you can directly intercept the launcher line inside the global application configuration.Open the main root desktop environment file:bashsudo nano /usr/share/applications/google-chrome.desktop
Use code with caution.Press Ctrl+W to search, type Exec=, and look at the line. Update all three instances of the Exec= lines inside this file to append the flags right after the executable.Change them from this:iniExec=/usr/bin/google-chrome-stable %U
Use code with caution.To this:iniExec=/usr/bin/google-chrome-stable --enable-low-end-device-mode --js-flags="--max-old-space-size=2048" --process-per-site %U
Use code with caution.Repeat this for the other two Exec= lines further down the file (the ones under Actions=new-window and Actions=new-private-window). Save and close.Step 3: Clear the Cache and RestartForce your desktop manager to drop its old configurations cache and load your new strict resource rules:bashsudo update-desktop-database /usr/share/applications/
Use code with caution.VerificationFully close all open Chrome windows (Ctrl+Q), re-open Chrome from your application launcher, and head back to chrome://version.You should now explicitly see --enable-low-end-device-mode, --js-flags="--max-old-space-size=2048", and --process-per-site printed alongside your --ozone-platform=wayland string.
