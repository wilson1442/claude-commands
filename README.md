# Claude Code Commands

Shared slash commands for Claude Code.

## Quick Start

```bash
# Clone the repo
git clone git@github.com:wilson1442/claude-commands.git /opt/claude-commands

# Symlink into Claude Code
mkdir -p ~/.claude/commands
ln -sf /opt/claude-commands/build-apk.md ~/.claude/commands/build-apk.md

# Add the skill API key (get it from https://apk.g3h.cloud → Settings → Skill API Key)
echo -n "apb_sk_..." > ~/.apk-builder-skill-key
chmod 600 ~/.apk-builder-skill-key
```

## Commands

### `/build-apk`

Submits the current project to the APK Builder at `https://apk.g3h.cloud` for compilation.

**What it does:**

1. Reads the skill API key from `~/.apk-builder-skill-key`
2. Checks if an app type already exists for this project — if not, analyzes the source code and creates one (config schema, template dir, platform, etc.)
3. Extracts build config from the project (app name, package ID, version, colors, API URLs, etc.)
4. Asks which icon/logo and keystore to use
5. Shows a summary and asks for confirmation
6. Submits the build, polls for completion, downloads the APK to the project root

**Usage:**

```
/build-apk           # build with version from source
/build-apk 2.1.0     # override version
```

**Requirements:**

- Skill API key at `~/.apk-builder-skill-key`
- `curl` available on the system

**Getting the skill key:**

1. Go to https://apk.g3h.cloud
2. Log in to the admin UI
3. Settings → Skill API Key → click the eye icon to reveal, or Roll Key to generate a new one
4. Copy the key and save it:

```bash
echo -n "apb_sk_your_key_here" > ~/.apk-builder-skill-key
chmod 600 ~/.apk-builder-skill-key
```

## Updating

```bash
cd /opt/claude-commands && git pull
```

Symlinked commands update automatically.
