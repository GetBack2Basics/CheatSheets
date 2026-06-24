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
| `Mail_Server_RoundCube.md` | Postfix/Dovecot/Roundcube setup, verification, and troubleshooting checklist. | Normalized to consistent guide format. |
| `cloudflare_tunnel_setup.md` | Cloudflare Tunnel setup with Docker run/compose examples and troubleshooting. | Normalized to consistent guide format. |
| `Firebase-Auth-User-Administration-Complete-Setup-Guide.md` | Complete Firebase Auth + user administration lifecycle for Vite + React apps, including approval workflow and admin console operations. | Newly added and ready to use. |

## New Guide Summary: Firebase Auth & User Administration

`Firebase-Auth-User-Administration-Complete-Setup-Guide.md` provides a full implementation playbook for authentication and account lifecycle management in a Vite + React + Firebase stack. It covers:

- Email/password sign-up with email verification
- Google OAuth using redirect flow
- Firestore-backed `userAccounts` profile/status model
- Admin approval/deactivation/reactivation and admin role controls
- Password reset and in-app password change with re-authentication
- Login/activity tracking fields and update strategy
- Bootstrap admin flow for first trusted account
- Baseline Firestore security rules and production hardening checklist

## Recommended Reading Order

1. Start with `Linux_Network_Machine_Improvement.md` if connectivity or install stability is an issue.
2. Continue with `Linux_Local_LLM_Hermes.md` to get local model serving working quickly.
3. Use `Linux_Hermes_Setup_Performance_Improvement.md` for strict per-project profile isolation.
4. Use `cloudflare_tunnel_setup.md` when exposing local services through Cloudflare Tunnel.
5. Use `Mail_Server_RoundCube.md` for self-hosted mail stack setup and troubleshooting.
6. Use `Firebase-Auth-User-Administration-Complete-Setup-Guide.md` when implementing authentication, authorization, and admin user operations in Firebase.

## Notes and Cautions

- Some commands modify system configuration (`/etc/sysctl.conf`, NetworkManager settings, resolver behavior, mail service configs). Review before execution.
- IPv4-only and IPv6-disable style steps are best treated as diagnostics/workarounds unless you intentionally want permanent policy changes.
- Replace placeholder credentials/domains before running commands in server guides.

## Suggested Next Improvements

- Add rollback commands for every config-changing step.
- Add a quick symptom-to-guide troubleshooting matrix at the top of this README.
- Add a lightweight "tested on" matrix (Ubuntu version, Docker version, and last validation date) for each guide.
