# Build APK

Submit the current project to the APK Builder service at `https://apk.g3h.cloud` for compilation.

## Constants

- **APK Builder URL:** `https://apk.g3h.cloud`
- **Skill key file:** `/root/.apk-builder-skill-key` (contains the skill API key)

## Instructions

### Step 1: Find the skill key

Read the skill key from `/root/.apk-builder-skill-key`.

If the file doesn't exist or is empty, stop and tell the user:

> No APK Builder skill key found.
> Go to https://apk.g3h.cloud → Settings → Skill API Key → Generate/Roll Key
> Then save it: `echo -n "apb_sk_..." > /root/.apk-builder-skill-key && chmod 600 /root/.apk-builder-skill-key`

All API calls below use the header `X-API-Key: $SKILL_KEY`.

### Step 2: Check if an app type exists for this project

First, fetch existing app types:
```bash
curl -s https://apk.g3h.cloud/api/v1/app-types -H "X-API-Key: $SKILL_KEY"
```

Look through the list — if one already matches this project (by name, package ID, or template dir), use it and skip to **Step 4**.

### Step 3: Create a new app type

If no existing app type fits, analyze the project source code to create one.

**Read the codebase to determine:**

- **Name** — A short label for this app/project (e.g., "Rovers Route", "My VPN")
- **Description** — One-line description
- **Package ID** — Android package (e.g., `com.roversroute.app`) from AndroidManifest.xml, app.json, build.gradle, etc.
- **Platform** — `android` (native Gradle) or `expo` (React Native / Expo)
- **Template directory** — Where the project source lives on the build server. Ask the user if unsure. Convention is `/opt/apk-builder/template-{project-slug}`.
- **Config schema** — What values need to be injected at build time. Scan the source for:
  - Placeholder tokens (e.g., `__API_BASE_URL__`, `__PRIMARY_COLOR__`)
  - Environment variables or `.env` patterns used at build time
  - Hard-coded URLs, colors, feature flags that should be configurable
  - Branding: app name, tagline, support email, website, privacy/terms URLs
  - Colors: primary, secondary, accent, etc.

  Build a config schema array. Each field:
  ```json
  {
    "key": "apiBaseUrl",
    "label": "API Base URL",
    "type": "text",
    "placeholder": "https://api.example.com/",
    "required": true,
    "group": "API Config",
    "placeholder_token": "__API_BASE_URL__"
  }
  ```
  Supported types: `text`, `color`, `boolean`, `number`. Group related fields (e.g., "API Config", "Branding", "Colors", "Features").

**Show the user what you detected** — name, package, platform, template dir, and the full config schema. Ask them to confirm or adjust before creating.

**Create it:**
```bash
curl -s -X POST https://apk.g3h.cloud/api/v1/app-types \
  -H "X-API-Key: $SKILL_KEY" \
  -H "Content-Type: application/json" \
  -d '{ "name": "...", "description": "...", "platform": "android", "configSchema": [...], "templateDir": "...", "defaultPackage": "..." }'
```

Save the returned `id` as the `appTypeId`.

### Step 4: Determine the build config

Read the project source to extract current values for submission.

**Required:**
- `appName` — User-facing app name
- `packageId` — Android package ID
- `appTypeId` — From step 2 or 3

**Optional:**
- `versionName` — Semver string (from build.gradle, app.json, package.json)
- `versionCode` — Integer version code
- `keystoreId` — check available keystores: `curl -s https://apk.g3h.cloud/api/v1/keystores -H "X-API-Key: $SKILL_KEY"`. Ask the user which to use if there are multiple.

**`configValues` object** — Fill in values from the source that match the app type's config schema. Only include fields you have real values for.

**Ask the user what icon/logo to use.** Check available assets:
```bash
curl -s https://apk.g3h.cloud/api/v1/assets -H "X-API-Key: $SKILL_KEY"
```
Show the list and ask which one to use (if any). The chosen filename goes in `iconPath` (e.g., `"1709234567-abc123.png"`).

If unsure about any config value, **ask the user** rather than guessing.

### Step 5: Confirm with the user

Print a summary:
```
Submitting build:
  App:       Rovers Route
  Package:   com.roversroute.app
  Version:   1.2.0
  App Type:  Rovers Route (ID: 4)
  Keystore:  Production (ID: 1)
  Icon:      1709234567-abc123.png
  Config:    apiBaseUrl=https://api.roversroute.com/, primaryColor=#4CAF50
```

Ask the user to confirm before proceeding.

### Step 6: Submit the build

```bash
curl -s -X POST https://apk.g3h.cloud/api/v1/builds \
  -H "X-API-Key: $SKILL_KEY" \
  -H "Content-Type: application/json" \
  -d '$JSON_BODY'
```

Response: `{"buildId":"...","status":"queued"}`. Save the `buildId`.

If you get a 409, a build is already running — tell the user.

### Step 7: Poll for completion

Poll every 10 seconds:
```bash
curl -s https://apk.g3h.cloud/api/v1/builds/$BUILD_ID \
  -H "X-API-Key: $SKILL_KEY"
```

Print each status change (e.g., `queued -> preparing -> building -> signing -> completed`).

Stop when `completed` or `failed`. On failure, show the `statusMessage`.

### Step 8: Download the APK

On `completed`:
```bash
curl -s -L -o "$APP_NAME-$VERSION.apk" \
  https://apk.g3h.cloud/api/v1/builds/$BUILD_ID/download \
  -H "X-API-Key: $SKILL_KEY"
```

Save to the project root. Tell the user the file path and size.

## Argument handling

If the user passes arguments (e.g., `/build-apk 2.1.0`), treat the first argument as the `versionName` override.
