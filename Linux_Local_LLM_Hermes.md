# Linux Local LLM Quick Start (Hermes + Gemma)

## Summary

Install Ollama on Ubuntu, verify the local service, pull the Gemma model, and point Hermes or other OpenAI-compatible tools at your local Ollama endpoint.

## Discussion

This guide prioritizes reliability on fresh Ubuntu installs where DNS or IPv6 routing can cause installation errors. Hermes is the agentic agent, while Gemma 3 12B is the local model served by Ollama.

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

### Step 3: Pull the Gemma model

Pull the model used locally by Hermes:

```bash
ollama pull gemma3:12b
```

### Step 4: Local endpoint for Hermes and OpenAI-compatible tools

Use the local Ollama OpenAI-compatible endpoint in 'Hermes model':

```text
http://localhost:11434/v1
```

If a client or app asks for the Ollama host instead of the OpenAI-compatible API base URL, use:

```text
http://localhost:11434
```

## Verification
- `ollama list` shows `gemma3:12b` as available locally.

## Troubleshooting

- **Could not resolve host during install:** Retry the install with the IPv4 command shown above.
- **`ollama serve` fails unexpectedly:** Check service state and logs with your init system.
- **Model name not found:** Run `ollama list` and verify that `gemma3:12b` is available or you want to try and updated version.
- **Client cannot connect to local Ollama:** Confirm Ollama is running and that the client is using `http://localhost:11434/v1` for OpenAI-compatible access.

