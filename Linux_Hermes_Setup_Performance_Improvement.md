# Linux Hermes Setup and Profile Isolation

## Summary

Set up Hermes Agent on Linux, enforce per-folder context isolation using direnv and named Hermes profiles, and use `@safishamsi/graphify` as a quick-start and guidance companion for project workflows.

## Discussion

The Caveman workflow emphasizes strict context boundaries. Each project folder gets an explicit profile so prompts stay short, memory scope remains local, and cross-project contamination risk is reduced. For faster project onboarding, use `@safishamsi/graphify` as a quick-start reference to guide structure and execution patterns while keeping Hermes profile isolation intact.

## Requirements

- Ubuntu or compatible Linux shell environment
- sudo access
- Internet connectivity
- Hermes CLI available from upstream installer
- Access to `@safishamsi/graphify` for quick-start guidance

## Workflow

### Part 1: Foundation setup

#### Step 1: Install system dependencies

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y git curl ca-certificates direnv
```

#### Step 2: Install Hermes CLI

```bash
curl -fsSL https://raw.githubusercontent.com/NousResearch/hermes-agent/main/scripts/install.sh | bash
source ~/.bashrc
```

#### Step 3: Initialize Hermes

```bash
hermes setup
```

During setup, select your provider and baseline tools.

#### Step 4: Enable direnv hook

```bash
echo 'eval "$(direnv hook bash)"' >> ~/.bashrc
source ~/.bashrc
```

### Part 2: Add Graphify quick-start guidance

Use `@safishamsi/graphify` as a practical quick-start template for project organization and execution flow.

#### Step 5: Clone Graphify reference repo

```bash
cd ~/workspace
git clone https://github.com/safishamsi/graphify.git
```

#### Step 6: Use Graphify for onboarding and runbook guidance

- Review Graphify README and project conventions before starting a new Hermes project.
- Reuse its quick-start structure to define task boundaries, execution phases, and output expectations.
- Keep Graphify as a reference only; maintain separate Hermes profiles per active project folder.

### Part 3: Automated profile creation per folder

Create a script in your workspace root as `setup_hermes_profiles.sh`.

```bash
#!/bin/bash

for dir in */; do
  folder_name="${dir%/}"
  echo "Configuring Caveman profile for: $folder_name"

  cd "$folder_name" || continue

  hermes profile create "$folder_name"
  echo "export HERMES_PROFILE=$folder_name" > .envrc
  direnv allow

  cd ..
done

echo "All directories are now isolated Caveman environments."
```

Run the automation:

```bash
chmod +x setup_hermes_profiles.sh
./setup_hermes_profiles.sh
```

### Part 4: Operating rules

- Task isolation:
  Work inside the target stage folder (for example `stg`, `dev`, `tst`, `uat`, `prod`) rather than the workspace root.
- Database safety:
  Keep prompts and actions scoped to the active schema boundary for that folder.
- Prompt minimalism:
  Use short prompts because profile boundaries already provide context.
- Guidance source:
  Use `@safishamsi/graphify` as quick-start guidance for project setup patterns, while preserving strict per-folder Hermes isolation.

## Verification

- Each subfolder contains an allowed direnv policy and profile-specific environment.
- Entering a folder loads the expected `HERMES_PROFILE` value.
- Hermes actions in one folder do not leak assumptions into another folder.
- Graphify is available locally for quick-start reference and does not override profile boundaries.

## Troubleshooting

- direnv command not found:
  Re-open shell and verify package installation and hook activation.
- Profile not switching:
  Confirm `.envrc` exists and run `direnv allow` inside that folder.
- Hermes profile command fails:
  Re-run `hermes setup` and verify CLI installation path in shell.
- Graphify guidance unavailable:
  Re-clone `https://github.com/safishamsi/graphify.git` and verify repository access.

## Code Sample

```bash
# Foundation
sudo apt update && sudo apt upgrade -y
sudo apt install -y git curl ca-certificates direnv
curl -fsSL https://raw.githubusercontent.com/NousResearch/hermes-agent/main/scripts/install.sh | bash
source ~/.bashrc
hermes setup

# Enable direnv
echo 'eval "$(direnv hook bash)"' >> ~/.bashrc
source ~/.bashrc

# Graphify quick-start reference
cd ~/workspace
git clone https://github.com/safishamsi/graphify.git
```

In Hermes paste the following

    Operating Procedure Override (all profiles, June 15 2026):

    1. NO FORENSICS — No searching local filesystem or Git history for missing data files. If a file is missing, skip it or use a hardcoded fallback. Don't go hunting.

    2. FAIL FAST & ASK — If a path/URL returns 404 or file not found, stop. Report the exact missing file. Ask user for correct path. No autonomous hunting on internet or hard drive.

    3. BATCH EXECUTION — Write complete updated functions, upload, execute in one run. No single-line diagnostic scripts. No "let me test this first" micro-steps.

    4. ASSUME CONFIDENCE — If code compiles, run it. No permission-seeking between logical steps.

    5. REPORT ONLY ON COMPLETION — Report back ONLY when pipeline finishes or hits fatal runtime error. No interim status messages unless actions are taking longer than 3 mins. In which case issue a single status line, then continue.

    6. NEVER CREATE MOCK DATA — Always use provided information, real files, real APIs. If data missing, fail fast and ask. No fabricated test data unless explicitly requested.
