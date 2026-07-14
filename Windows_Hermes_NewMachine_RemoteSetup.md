# Windows New Machine → Remote Hermes Setup

Summary: From a fresh Windows box, get `gh`, git identity, GitHub auth, an SSH key into the
remote, and a PowerShell `orc` shortcut to jump straight into the remote Hermes box.

Operating mode for Hermes on this work: **Caveman** (see `Linux_Hermes_Setup_Performance_Improvement.md`,
"Operating Procedure Override"). Terse. No forensics. Fail fast & ask. Batch. No mock data.

When the user asks Hermes to "set up a new Windows machine," read THIS file, then ask the
user only for the specifics: remote **host**, remote **user**, where the SSH key lives
(local path), and the **shortcut name** they want for the PowerShell jump function (e.g.
`orc`). Do not guess host/user/shortcut.

**RULE: never `git commit` or `git push` without explicitly asking the user first.** Draft
the change, state what will be committed/pushed, and wait for confirmation.

## Part 1: Prereqs probe (run first)

```powershell
# in PowerShell
ssh -V                       # need OpenSSH present
git --version                # git usually preinstalled via git-for-windows
where puttygen               # need it to inspect/convert .ppk keys
ls ~\.ssh                    # existing keys?
```

If `ssh`/`git` missing, install git-for-windows (ships both). Record host, user, key path
from the user.

## Part 2: Install gh CLI (no admin, zip)

```powershell
# run in bash/git-bash OR via the terminal tool; curl + unzip are present
mkdir -p $HOME/.local
cd $HOME/.local
curl -fsSL -o gh.zip "https://github.com/cli/cli/releases/download/v2.96.0/gh_2.96.0_windows_amd64.zip"
unzip -o -q gh.zip          # extracts flat into $HOME/.local/bin/gh.exe
$HOME/.local/bin/gh.exe --version
# persist PATH
echo 'export PATH="$HOME/.local/bin:$PATH"' >> $HOME/.bashrc
```

Note: Windows `gh` ships as **.zip or .msi**, NOT tar.gz. A `gh_*_windows_amd64.tar.gz`
URL 404s.

## Part 3: Git identity + GitHub auth

```powershell
git config --global user.name "getback2basics"
# privacy noreply email: <github_id>+<login>@users.noreply.github.com
git config --global user.email "17077850+getback2basics@users.noreply.github.com"
git config --global init.defaultBranch main

# device login (background it; it polls). User completes in browser.
gh auth login -p https -h github.com --web -s repo,workflow,read:org
# user opens https://github.com/login/device , enters the printed code, Authorize
gh auth status                # verify
```

If a scope refresh later needs device auth (e.g. `admin:public_key`), same flow.

## Part 4: SSH key into remote

The user supplies a key path (often `C:\Projects\id_rsa.ppk`). Gotchas:

- `.ppk` may be **real PuTTY format** (needs conversion) OR **OpenSSH format mislabeled**.
  Check: if first line is `-----BEGIN OPENSSH PRIVATE KEY-----` it is already OpenSSH.
- Real PuTTY format → `puttygen.exe key.ppk -O private-openssh -o ~/.ssh/id_rsa`
- OpenSSH mislabeled → just copy: `cp key.ppk ~/.ssh/id_rsa`
- Always: `chmod 600 ~/.ssh/id_rsa`

```powershell
mkdir -p ~/.ssh
# CASE A (real ppk):
"/c/Program Files/PuTTY/puttygen.exe" /c/Projects/id_rsa.ppk -O private-openssh -o ~/.ssh/id_rsa
# CASE B (already OpenSSH, mislabeled .ppk):
cp /c/Projects/id_rsa.ppk ~/.ssh/id_rsa
chmod 600 ~/.ssh/id_rsa

# test
ssh -o StrictHostKeyChecking=no -o BatchMode=yes USER@HOST 'echo SSH_OK'
```

Remote `hermes` is NOT on PATH in non-interactive ssh (no .bashrc). Use a login shell:
`ssh USER@HOST 'bash -lc "hermes ..."'`

## Part 5: PowerShell `orc` shortcut

User terminal is **PowerShell**, NOT bash. `.bashrc` aliases do nothing there.

```powershell
# create profile dir if missing, add function (use the SHORTCUT name the user chose, e.g. orc)
if (!(Test-Path (Split-Path $PROFILE))) { New-Item -ItemType Directory -Force -Path (Split-Path $PROFILE) | Out-Null }
Add-Content -Path $PROFILE -Value 'function SHORTCUT { ssh -i $HOME\.ssh\id_rsa USER@HOST }'
```

PowerShell blocks profile scripts by default (ExecutionPolicy). Enable once:
```powershell
Set-ExecutionPolicy -Scope CurrentUser RemoteSigned -Force
```
Then in a new PowerShell: `. $PROFILE` (or reopen) → type `orc` to jump to remote.

## Part 6: Optional — apply per-project memory hardening on remote

Per `Linux_Hermes_Setup_Performance_Improvement.md` Part 5: silence the default profile so
no general memory accumulates (per-folder profiles keep their own memory).

```powershell
ssh USER@HOST 'bash -lc "hermes profile use default && hermes config set memory.memory_enabled false"'
ssh USER@HOST 'bash -lc "grep -A2 -i ^memory: ~/.hermes/config.yaml"'   # verify: memory_enabled: false
```

Verify with `config show` does NOT print the memory line by default — read `config.yaml`
directly to confirm.

## Gotchas (learned the hard way)
- User runs PowerShell, not git-bash. Don't add `.bashrc` aliases expecting them to work.
- PowerShell ExecutionPolicy blocks profiles until `RemoteSigned`.
- `gh` Windows = zip/msi, not tar.gz (404 trap).
- `.ppk` can be OpenSSH-in-disguise; `puttygen` returns exit 1 with no message on it.
- Remote `hermes` needs `bash -lc` over ssh (non-login shell lacks PATH).
- `hermes config get` is NOT a subcommand; use `config show` or read `config.yaml`.
- GitHub SSH-key add (`gh ssh-key add`) needs `admin:public_key` scope + device re-auth;
  HTTPS `gho_` token works without it — skip unless key auth to GitHub is wanted.
