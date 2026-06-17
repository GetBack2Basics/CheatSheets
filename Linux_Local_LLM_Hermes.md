# Linux Local LLM Quick Start (Hermes + Qwen)

## Summary

Install Ollama on Ubuntu, verify the local service, run Hermes and Qwen, and point OpenAI-compatible tools at your local Ollama endpoint.

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

If you see `address already in use`, the service is already running.

### Step 3: Pull and run Hermes

The run command pulls the model if it is not present and then starts an interactive chat.

```bash
ollama run hermes3
```

### Step 4: Pull and run Qwen

```bash
ollama run qwen2.5
```

### Step 5: Local endpoint for Hermes and OpenAI-compatible tools

Use the local Ollama OpenAI-compatible endpoint:

```text
http://localhost:11434/v1
```

If a client or app asks for the Ollama host instead of the OpenAI-compatible API base URL, use:

```text
http://localhost:11434
```

Example environment variables:

```bash
# OpenAI-compatible clients
export OPENAI_BASE_URL=http://localhost:11434/v1

# Tools that connect to Ollama directly
export OLLAMA_HOST=http://localhost:11434
```

## Verification

- `ollama run hermes3` launches an interactive prompt without errors.
- `ollama run qwen2.5` launches an interactive prompt without errors.
- OpenAI-compatible tools can connect when pointed at `http://localhost:11434/v1`.
- Type `/bye` to exit model chat and return to the shell.

## Troubleshooting

- **Could not resolve host during install:** Retry the install with the IPv4 command shown above.
- **`ollama serve` fails unexpectedly:** Check service state and logs with your init system.
- **Model name not found:** Run `ollama list` and verify available tags.
- **Client cannot connect to local Ollama:** Confirm Ollama is running and that the client is using `http://localhost:11434/v1` for OpenAI-compatible access.

## Quick Command Reference

```bash
# Install Ollama with IPv4 fallback
curl -4 -fsSL https://ollama.com/install.sh | sh

# Start service (if not already running)
ollama serve

# Run models
ollama run hermes3
ollama run qwen2.5

# Optional environment variables for local integrations
export OPENAI_BASE_URL=http://localhost:11434/v1
export OLLAMA_HOST=http://localhost:11434
```
