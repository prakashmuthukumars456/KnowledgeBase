# Git Multi-Account SSH Setup

Managing two Git accounts — personal (GitHub) and work (Azure DevOps) — using SSH host aliases.

---

## Accounts

| Account | Email | Platform |
|---|---|---|
| Personal | prakashmuthukumars456@gmail.com | GitHub |
| Work | prakash.m@nafinc.com | Azure DevOps (naftech) |

---

## Part 1 — SSH Host Aliases

### Step 1 — Generate SSH Keys

```bash
# Personal key for GitHub
ssh-keygen -t ed25519 -C "prakashmuthukumars456@gmail.com" -f ~/.ssh/id_ed25519_personal

# Work key for Azure DevOps
ssh-keygen -t ed25519 -C "prakash.m@nafinc.com" -f ~/.ssh/id_ed25519_work
```

Files created:

```
~/.ssh/id_ed25519_personal      ← private key (never share)
~/.ssh/id_ed25519_personal.pub  ← public key → goes to GitHub
~/.ssh/id_ed25519_work          ← private key (never share)
~/.ssh/id_ed25519_work.pub      ← public key → goes to Azure DevOps
```

---

### Step 2 — Add Keys to SSH Agent

```bash
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/id_ed25519_personal
ssh-add ~/.ssh/id_ed25519_work
```

To persist across reboots, add the `ssh-add` lines to `~/.bashrc` or `~/.zshrc`.

---

### Step 3 — Add Public Keys to Platforms

Copy each public key:

```bash
cat ~/.ssh/id_ed25519_personal.pub
cat ~/.ssh/id_ed25519_work.pub
```

**GitHub (personal):**
- Settings → SSH and GPG Keys → New SSH Key
- Paste contents of `id_ed25519_personal.pub`

**Azure DevOps (work):**
- `dev.azure.com` → User Settings (top right) → SSH Public Keys → New Key
- Paste contents of `id_ed25519_work.pub`

> The `.pub` file is safe to share with platforms. Never share the private key (the one without `.pub`).

---

### Step 4 — Create SSH Config

```bash
nano ~/.ssh/config
```

```
# Personal - GitHub
Host github-personal
  HostName github.com
  User git
  IdentityFile ~/.ssh/id_ed25519_personal

# Work - Azure DevOps
Host azuredevops-work
  HostName ssh.dev.azure.com
  User git
  IdentityFile ~/.ssh/id_ed25519_work
  IdentitiesOnly yes
```

Set correct permissions:

```bash
chmod 600 ~/.ssh/config
```

---

### Step 5 — Test Connections

```bash
ssh -T git@github-personal
# Expected: Hi prakashmuthukumars456! You've successfully authenticated...

ssh -T git@azuredevops-work
# Expected: remote: Shell access is not supported  ← this is normal, means auth worked
```

If you see `Permission denied (publickey)` — the public key was not added correctly in Step 3.

---

### Step 6 — Update Existing Repo Remotes

```bash
# Personal repos
git remote set-url origin git@github-personal:prakashmuthukumars456/your-repo.git

# Work repos
git remote set-url origin git@azuredevops-work:v3/OrgName/ProjectName/RepoName
```

To find your Azure DevOps SSH URL: repo → Clone → SSH → copy URL, then replace `ssh.dev.azure.com` with alias `azuredevops-work`.

Verify:

```bash
git remote -v
# should show git@github-personal:... or git@azuredevops-work:...
```

---

### New Clones Going Forward

Clone directly using the alias — no remote fix needed after:

```bash
# Personal
git clone git@github-personal:prakashmuthukumars456/repo.git

# Work
git clone git@azuredevops-work:v3/OrgName/ProjectName/RepoName
```

---

## Part 2 — Commit Identity (one-time per repo)

Set local identity once after cloning any repo:

```bash
# In a personal repo
git config --local user.email "prakashmuthukumars456@gmail.com"
git config --local user.name "Prakash Muthukumar"

# In a work repo
git config --local user.email "prakash.m@nafinc.com"
git config --local user.name "Prakash M"
```

This writes to `.git/config` inside that repo and stays permanently.

Verify:

```bash
git config user.email
git config user.name
```

---

## Quick Reference

| Task | Command |
|---|---|
| Check identity in current repo | `git config user.email` |
| Check remote URL | `git remote -v` |
| Test GitHub SSH | `ssh -T git@github-personal` |
| Test Azure DevOps SSH | `ssh -T git@azuredevops-work` |
| Update remote to alias | `git remote set-url origin git@<alias>:<path>` |
| List SSH keys loaded | `ssh-add -l` |

---

## How It Works

```
git push
   │
   ├── reads remote URL → git@github-personal:...
   │         │
   │         └── SSH config maps github-personal → id_ed25519_personal
   │
   └── reads remote URL → git@azuredevops-work:...
             │
             └── SSH config maps azuredevops-work → id_ed25519_work
```

Authentication is fully automatic — no manual switching needed once set up.