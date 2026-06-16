# CheatSheets

Practical Linux-focused cheat sheets for quickly setting up local AI tooling and improving network reliability.

## What is in this repo

This repository currently contains three Linux guides:

1. **Hermes setup and profile isolation** for multi-folder workflows.
2. **Local LLM quick start** using Ollama with Hermes and Qwen models.
3. **Network troubleshooting and performance improvements** for Ubuntu systems.

## Guide Index

| Guide | Purpose | Current State |
|---|---|---|
| `Linux_Hermes_Setup_Performance_Improvement.md` | End-to-end Hermes setup + automated per-folder profile isolation via `direnv`. | Normalized to a consistent reference format. |
| `Linux_Local_LLM_Hermes.md` | Fast setup path for Ollama, then pull/run Hermes and Qwen models. | Normalized to a consistent reference format. |
| `Linux_Network_Improvement.md` | Ubuntu Wi-Fi, IPv6, DNS, and bufferbloat troubleshooting checklist. | Normalized to a consistent reference format. |

## Recommended Reading Order

1. Start with `Linux_Network_Improvement.md` if initial install commands fail due to connectivity issues.
2. Continue with `Linux_Local_LLM_Hermes.md` to get Ollama and models running quickly.
3. Use `Linux_Hermes_Setup_Performance_Improvement.md` when you want strict per-project profile isolation.

## Notes and Cautions

- Some commands modify system configuration (`/etc/sysctl.conf`, NetworkManager settings, DNS resolver behavior). Review each command before execution.
- IPv6 disablement is included as a diagnostic/workaround step; prefer fixing upstream router/ISP configuration when possible.
- All guides now use markdown file extensions for consistent editor rendering.

## Suggested Next Improvements

- Add a troubleshooting matrix mapping common symptoms to the right guide and step.
- Add a safety profile section that marks each command as low, medium, or high system impact.
- Add rollback commands for every configuration-changing step.
