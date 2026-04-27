# Using Multiple OpenRouter API Keys Across Projects with direnv

This guide walks you through setting up `direnv` so that each project automatically loads its own OpenRouter API key when you `cd` into it — no manual `export` needed, no risk of using the wrong key.

---

## How It Works

```
~/projects/
├── project-a/     ← loads KEY_A automatically
│   └── .env.local
└── project-b/     ← loads KEY_B automatically
    └── .env.local
```

`direnv` watches your current directory. When you enter a folder, it loads that folder's environment variables. When you leave, it unloads them. Your API key switches automatically as you move between projects.

---

## Step 1 — Install direnv

### macOS
```bash
brew install direnv
```

### Ubuntu / Debian
```bash
sudo apt install direnv
```

### Arch Linux
```bash
sudo pacman -S direnv
```

### Windows (WSL — Recommended)
```bash
sudo apt install direnv
```

### Windows (Native — via Scoop)
```powershell
scoop install direnv
```

---

## Step 2 — Hook direnv Into Your Shell

This is required. Without it, direnv installs but does nothing.

### Zsh (most macOS users)
```bash
echo 'eval "$(direnv hook zsh)"' >> ~/.zshrc
source ~/.zshrc
```

### Bash
```bash
echo 'eval "$(direnv hook bash)"' >> ~/.bashrc
source ~/.bashrc
```

### Fish
```bash
echo 'direnv hook fish | source' >> ~/.config/fish/config.fish
```

### PowerShell (Windows native)
```powershell
Add-Content $PROFILE 'Invoke-Expression "$(direnv hook pwsh)"'
. $PROFILE
```

Verify it's working:
```bash
direnv version
# Should print something like: 2.34.0
```

---

## Step 3 — Set Up Global Codex Config

Create your global Codex config once. This sets the provider and which environment variable to read the API key from.

```bash
mkdir -p ~/.codex
nano ~/.codex/config.toml
```

Paste this:

```toml
model_provider = "openrouter"
model_reasoning_effort = "medium"
model = "openai/gpt-4o"

[model_providers.openrouter]
name = "openrouter"
base_url = "https://openrouter.ai/api/v1"
env_key = "OPENROUTER_API_KEY"
```

Save with `Ctrl+X` → `Y` → `Enter`.

> The `env_key` field tells Codex which environment variable holds the API key. `direnv` will inject the right one per project.

---

## Step 4 — Set Up Project A

```bash
cd ~/path/to/project-a
```

**Create the `.env.local` file with Project A's key:**
```bash
nano .env.local
```

Add this line (replace with your actual key):
```env
OPENROUTER_API_KEY=sk-or-v1-projectA-key-xxxxxxxxxxxxxxxxxxxx
```

Save with `Ctrl+X` → `Y` → `Enter`.

**Create the `.envrc` file to tell direnv to load `.env.local`:**
```bash
echo 'dotenv_if_exists .env.local' > .envrc
```

**Approve it:**
```bash
direnv allow
```

You should see:
```
direnv: loading .envrc
direnv: export +OPENROUTER_API_KEY
```

**Verify the correct key loaded:**
```bash
echo $OPENROUTER_API_KEY
# Should print Project A's key
```

---

## Step 5 — Set Up Project B

```bash
cd ~/path/to/project-b
```

**Create the `.env.local` file with Project B's key:**
```bash
nano .env.local
```

Add this line:
```env
OPENROUTER_API_KEY=sk-or-v1-projectB-key-xxxxxxxxxxxxxxxxxxxx
```

Save with `Ctrl+X` → `Y` → `Enter`.

**Create the `.envrc` and approve:**
```bash
echo 'dotenv_if_exists .env.local' > .envrc
direnv allow
```

**Verify:**
```bash
echo $OPENROUTER_API_KEY
# Should print Project B's key
```

---

## Step 6 — Verify Keys Switch Automatically

Now test that the keys swap correctly as you move between projects:

```bash
cd ~/path/to/project-a
echo $OPENROUTER_API_KEY   # prints Project A key

cd ~/path/to/project-b
echo $OPENROUTER_API_KEY   # prints Project B key

cd ~
echo $OPENROUTER_API_KEY   # prints nothing (unloaded)
```

This is direnv working correctly. ✅

---

## Step 7 — Protect Your Keys with .gitignore

**Critical** — never commit API keys to version control.

In each project, add `.env.local` to `.gitignore`:

```bash
echo '.env.local' >> .gitignore
```

You may also want to ignore `.envrc` if it contains sensitive logic, though in our setup it only contains `dotenv_if_exists .env.local` which is safe to commit:

```bash
# Optional — only needed if .envrc itself contains secrets
echo '.envrc' >> .gitignore
```

---

## Daily Workflow

Once set up, your daily workflow is just:

```bash
# Project A session
cd ~/path/to/project-a    # key A loads automatically
codex                      # uses key A

# Project B session
cd ~/path/to/project-b    # key B loads automatically
codex                      # uses key B
```

No manual `export`. No copy-pasting keys. No wrong-key accidents.

---

## Adding More Projects

Every new project follows the same two steps:

```bash
cd ~/path/to/new-project

# 1. Add the key
echo 'OPENROUTER_API_KEY=sk-or-v1-new-project-key-xxxx' > .env.local

# 2. Enable direnv
echo 'dotenv_if_exists .env.local' > .envrc
direnv allow

# 3. Protect the key
echo '.env.local' >> .gitignore
```

---

## Troubleshooting

### "direnv: command not found" after install
The hook wasn't added or the shell wasn't reloaded. Re-run:
```bash
# Zsh
echo 'eval "$(direnv hook zsh)"' >> ~/.zshrc && source ~/.zshrc

# Bash
echo 'eval "$(direnv hook bash)"' >> ~/.bashrc && source ~/.bashrc
```

### "direnv: error .envrc is blocked"
You need to approve the file before direnv will load it:
```bash
direnv allow
```

### Key not loading (OPENROUTER_API_KEY is empty)
Check that your `.env.local` has no spaces around the `=`:
```bash
# ✅ Correct
OPENROUTER_API_KEY=sk-or-v1-abc123

# ❌ Wrong — spaces break parsing
OPENROUTER_API_KEY = sk-or-v1-abc123
```

Also verify `.env.local` exists in the current directory:
```bash
ls -la .env.local
cat .envrc   # should show: dotenv_if_exists .env.local
```

### Key not switching when changing directories
Make sure you're opening a new terminal or that the hook is correctly installed. Test by running:
```bash
cd /tmp && echo $OPENROUTER_API_KEY   # should be empty
cd ~/path/to/project-a && echo $OPENROUTER_API_KEY  # should load
```

### "direnv: .env.local not found" warning
This is just a warning, not an error — `dotenv_if_exists` silently skips missing files. If you used `dotenv` instead of `dotenv_if_exists`, change it:
```bash
echo 'dotenv_if_exists .env.local' > .envrc
direnv allow
```

---

## File Structure Summary

```
~/.codex/
└── config.toml          ← global config (provider, model, env_key name)

~/path/to/project-a/
├── .env.local           ← OPENROUTER_API_KEY=sk-or-v1-key-A (gitignored)
├── .envrc               ← dotenv_if_exists .env.local
└── .gitignore           ← includes .env.local

~/path/to/project-b/
├── .env.local           ← OPENROUTER_API_KEY=sk-or-v1-key-B (gitignored)
├── .envrc               ← dotenv_if_exists .env.local
└── .gitignore           ← includes .env.local
```

---

## Security Notes

- **Never hardcode API keys** in source files, configs, or commit messages
- **Always use `.gitignore`** to exclude `.env.local` before your first commit
- **Rotate keys regularly** on the [OpenRouter dashboard](https://openrouter.ai/keys) — if a key is exposed, delete it and create a new one
- **Use separate keys per project** so you can revoke one without affecting others — this is exactly what this guide enables
