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