# Claude Code Skills

A collection of reusable skills for [Claude Code](https://docs.anthropic.com/en/docs/claude-code). Each skill lives in its own directory under `skills/` and can be installed into any project.

## Available Skills

### `build-apk`

Submits a project to the [APK Builder](https://apk.g3h.cloud) service for Android compilation. Handles app type detection/creation, config extraction, icon management, build submission, polling, and APK download — all from a single `/build-apk` command.

**Requirements:** Skill API key saved at `~/.apk-builder-skill-key`

### `nocodb`

Scaffolds full NocoDB CRUD integration into any web project. Auto-detects the framework (Next.js, Express, React SPA, Python, etc.) and generates server-side API routes, a typed client, environment config, and usage examples.

**Supported frameworks:** Next.js (App & Pages Router), Express, Fastify, React/Vue SPA, FastAPI/Flask, standalone Node.js

## Installation

### Option 1: Install a single skill

Clone the repo, then symlink the skill directory into your project's `.claude/skills/`:

```bash
# Clone the repo (one-time setup)
git clone https://github.com/wilson1442/claude-commands.git /opt/claude-commands

# From your project root, symlink the skill you want
mkdir -p .claude/skills
ln -sf /opt/claude-commands/skills/build-apk .claude/skills/build-apk
ln -sf /opt/claude-commands/skills/nocodb .claude/skills/nocodb
```

### Option 2: Install all skills

Symlink the entire `skills/` directory contents into your project:

```bash
# Clone the repo (one-time setup)
git clone https://github.com/wilson1442/claude-commands.git /opt/claude-commands

# From your project root, symlink all skills at once
mkdir -p .claude/skills
for skill in /opt/claude-commands/skills/*/; do
  ln -sf "$skill" .claude/skills/
done
```

### Skill-specific setup

**build-apk** — requires an API key:

```bash
# Get your key from https://apk.g3h.cloud -> Settings -> Skill API Key
echo -n "apb_sk_your_key_here" > ~/.apk-builder-skill-key
chmod 600 ~/.apk-builder-skill-key
```

**nocodb** — no extra setup needed. The skill will prompt for your NocoDB URL, table ID, and API token when you use it.

## Updating

```bash
cd /opt/claude-commands && git pull
```

Symlinked skills update automatically.

## Structure

```
skills/
├── build-apk/
│   └── SKILL.md              # Skill definition
└── nocodb/
    ├── SKILL.md              # Skill definition
    └── references/           # Framework-specific code templates
        ├── express.md
        ├── nextjs-app.md
        ├── nextjs-pages.md
        ├── python.md
        ├── spa.md
        └── standalone.md
```

## Adding a new skill

1. Create a directory under `skills/` with your skill name
2. Add a `SKILL.md` with the skill definition (frontmatter + instructions)
3. Optionally add a `references/` subdirectory for supporting templates
4. Update this README with a description and any setup steps
