# Claude Code Commands

Shared slash commands for Claude Code.

## Setup

Copy the commands to your Claude Code user commands directory:

```bash
mkdir -p ~/.claude/commands
cp *.md ~/.claude/commands/
```

Or symlink them:

```bash
mkdir -p ~/.claude/commands
ln -sf /opt/claude-commands/build-apk.md ~/.claude/commands/build-apk.md
```

## Commands

### `/build-apk`

Submits the current project to the APK Builder at `https://apk.g3h.cloud` for compilation.

Requires the skill API key at `~/.apk-builder-skill-key`. Get it from the APK Builder admin UI under Settings > Skill API Key.

```bash
echo -n "apb_sk_..." > ~/.apk-builder-skill-key
chmod 600 ~/.apk-builder-skill-key
```
