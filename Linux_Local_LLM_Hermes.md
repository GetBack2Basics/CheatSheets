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

In a separate terminal, start the service if needed.

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