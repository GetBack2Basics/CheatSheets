# Linux Local LLM Quick Start (Hermes + Gemma)

## Summary

Install Ollama on Ubuntu, verify the local service, create a custom local model with a larger context window, and point Hermes or other OpenAI-compatible tools at your local Ollama endpoint.

## Discussion

This guide prioritizes reliability on fresh Ubuntu installs where DNS or IPv6 routing can cause installation errors. Hermes is the agentic agent, while the local Ollama model is customized from `orieg/gemma3-tools:12b-ft` with a larger context window.

## Requirements

- Ubuntu system with sudo access
- Internet connectivity
- Terminal access
- Ollama installed locally

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

### Step 3: Create a custom model with a larger context window

Create a file named `Modelfile` with the following contents:

```text
FROM orieg/gemma3-tools:12b-ft
PARAMETER num_ctx 65536
```

Then build the custom model locally:

```bash
ollama create gemma3-tools-hermes -f Modelfile
```

### Step 4: Update Hermes to use the custom model

Update your Hermes configuration to use the new model name:

```yaml
model:
  name: "gemma3-tools-hermes"
```

### Step 5: Local endpoint for Hermes and OpenAI-compatible tools

Use the local Ollama OpenAI-compatible endpoint in your Hermes or client configuration:

```text
http://localhost:11434/v1
```

If a client or app asks for the Ollama host instead of the OpenAI-compatible API base URL, use:

```text
http://localhost:11434
```

## Verification

- `ollama list` shows `gemma3-tools-hermes` as available locally.
- Hermes is configured to reference `gemma3-tools-hermes`.
- The local endpoint responds at `http://localhost:11434/v1`.

## Troubleshooting

- **Could not resolve host during install:** Retry the install with the IPv4 command shown above.
- **`ollama serve` fails unexpectedly:** Check service state and logs with your init system.
- **Model name not found:** Run `ollama list` and verify that `gemma3-tools-hermes` was created successfully.
- **Client cannot connect to local Ollama:** Confirm Ollama is running and that the client is using `http://localhost:11434/v1` for OpenAI-compatible access.
- **Modelfile errors:** Confirm the file is named exactly `Modelfile` with no extension and that the `FROM` line matches the source model.
