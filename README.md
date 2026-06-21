# CheatSheets

Cheat sheets to get up and running quickly.

## What is in this repo

This repository contains practical Linux/networking/server setup notes and quick-start guides.

## Guide Index

| Guide | Purpose | Current State |
|---|---|---|
| `Linux_Hermes_Setup_Performance_Improvement.md` | Set up Hermes on Linux and enforce per-folder profile isolation with `direnv`. | Current and useful. |
| `Linux_Local_LLM_Hermes.md` | Quick start for Ollama with local model setup and endpoint usage. | Current and useful. |
| `Linux_Network_Machine_Improvement.md` | Ubuntu networking reliability and DNS/latency troubleshooting checklist. | Current and useful. |
| `Mail_Server_RoundCube.md` | Roundcube/Postfix/Dovecot mail server notes and troubleshooting. | Needs cleanup (contains terminal editor artifacts and truncated content). |
| `cloudflare_tunnel_setup.md` | Cloudflare Tunnel setup and Docker integration notes. | Needs cleanup (contains terminal editor artifacts and truncated content). |

## Recommended Reading Order

1. Start with `Linux_Network_Machine_Improvement.md` if connectivity or install stability is an issue.
2. Continue with `Linux_Local_LLM_Hermes.md` to get local model serving working quickly.
3. Use `Linux_Hermes_Setup_Performance_Improvement.md` for strict per-project profile isolation.
4. Use `cloudflare_tunnel_setup.md` when exposing local services through Cloudflare Tunnel.
5. Use `Mail_Server_RoundCube.md` for self-hosted mail stack setup and troubleshooting.

## Notes and Cautions

- Some commands modify system configuration (`/etc/sysctl.conf`, NetworkManager settings, resolver behavior, mail service configs). Review before execution.
- IPv4-only and IPv6-disable style steps are best treated as diagnostics/workarounds unless you intentionally want permanent policy changes.
- Two guides currently appear to include copied terminal editor output and may need normalization:
  - `Mail_Server_RoundCube.md`
  - `cloudflare_tunnel_setup.md`

## Suggested Next Improvements

- Normalize `Mail_Server_RoundCube.md` and `cloudflare_tunnel_setup.md` into clean markdown sections (Summary, Requirements, Workflow, Verification, Troubleshooting).
- Add rollback commands for every config-changing step.
- Add a quick symptom-to-guide troubleshooting matrix at the top of this README.
