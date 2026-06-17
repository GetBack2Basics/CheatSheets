# Linux Local LLM Quick Start (Hermes + Qwen)

## Summary

Install Ollama on Ubuntu, confirm the service is available, and run Hermes and Qwen models with stable commands.

## Discussion

This guide prioritizes reliability on fresh Ubuntu installs where DNS or IPv6 routing can cause installation errors.

## Requirements

- Ubuntu system with sudo access
- Internet connectivity
- Terminal access

## Workflow

### Step 1: Install Ollama (IPv4 fallback)

Use IPv4 to avoid early resolver or IPv6 path issues.

```bash
curl -4 -fsSL https://ollama.com/install.sh | sh
```

### Step 2: Verify Ollama service

In a separate terminal, you can start the service if needed but it should be started.

```bash
ollama serve
```

If you see address already in use, the service is already running.

### Step 3: Pull and run Hermes

The run command pulls the model if not present and then starts an interactive chat.

```bash
ollama run hermes3
```

### Step 4: Pull and run Qwen

```bash
ollama run qwen2.5
```

## Verification

- Ollama responds to model launch commands without errors.
- You enter an interactive prompt after each run command.
- Type /bye to exit model chat and return to shell.

## Troubleshooting

- Could not resolve host during install:
	Retry install with the IPv4 command shown above.
- ollama serve fails unexpectedly:
	Check service state and logs with your init system.
- Model name not found:
	Run ollama list and verify available tags.

## Code Sample

```bash
# Install Ollama with IPv4 fallback
curl -4 -fsSL https://ollama.com/install.sh | sh

# Start service (if not already running)
ollama serve

# Run models
ollama run hermes3
ollama run qwen2.5
```

## Setup Hermes to use it
Hermes model

# Issues

free -h

Add a Swap File
Adding a swap file gives the kernel breathing room to page out inactive memory, allowing the model to load successfully without triggering the OOM killer.

Since you are already logged in as root, run these commands to create and enable an 8GB swap file:

Bash
# 1. Create the swap file
fallocate -l 8G /swapfile

# 2. Set the correct permissions (crucial for security)
chmod 600 /swapfile

# 3. Format the file as swap space
mkswap /swapfile

# 4. Enable the swap space
swapon /swapfile

# 5. Make it permanent (so it survives a reboot)
echo '/swapfile none swap sw 0 0' >> /etc/fstab
Run a smaller model parameter size: By default, ollama run qwen2.5 usually pulls the 7B parameter model, which requires around 5GB+ of RAM/VRAM to run comfortably. You can explicitly request a smaller, lighter version of the model:

Bash
ollama run qwen2.5:3b    # Requires ~3GB RAM
ollama run qwen2.5:1.5b  # Requires ~2GB RAM
ollama run qwen2.5:0.5b  # Requires <1GB RAM


