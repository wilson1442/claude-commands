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
- `buildFormat` — `"aab"` (Android App Bundle, required for new Google Play submissions) or `"apk"` (default). **For Play Store submissions, always use `"aab"`.**

**`configValues` object** — Fill in values from the source that match the app type's config schema. Only include fields you have real values for.

#### Icon / Image handling

You have three options for providing an app icon:

1. **Use an existing asset** — Check available assets:
   ```bash
   curl -s https://apk.g3h.cloud/api/v1/assets -H "X-API-Key: $SKILL_KEY"
   ```
   Set the chosen filename as `iconPath` (e.g., `"1709234567-abc123.png"`).

2. **Upload an image file** — If the user has a local image file to use as the icon, upload it first:
   ```bash
   curl -s -X POST https://apk.g3h.cloud/api/v1/assets/upload \
     -H "X-API-Key: $SKILL_KEY" \
     -F "file=@/path/to/icon.png"
   ```
   Use the returned `filename` as `iconPath`.

3. **Provide an image URL** — If the user provides a URL to an image, either:
   - Download it to an asset first:
     ```bash
     curl -s -X POST https://apk.g3h.cloud/api/v1/assets/from-url \
       -H "X-API-Key: $SKILL_KEY" \
       -H "Content-Type: application/json" \
       -d '{"url": "https://example.com/logo.png"}'
     ```
     Use the returned `filename` as `iconPath`.
   - Or pass `iconUrl` directly in the build submission (it will be downloaded automatically).

**Ask the user what icon/logo to use** if none is obvious from the project.

If unsure about any config value, **ask the user** rather than guessing.

### Step 5: Confirm with the user

Print a summary:
```
Submitting build:
  App:       Rovers Route
  Package:   com.roversroute.app
  Version:   1.2.0
  Format:    AAB (Play Store) / APK
  App Type:  Rovers Route (ID: 4)
  Keystore:  Production (ID: 1)
  Icon:      1709234567-abc123.png
  Source:    server template / uploaded project
  Config:    apiBaseUrl=https://api.roversroute.com/, primaryColor=#4CAF50
```

Ask the user to confirm before proceeding.

### Step 6: Submit the build

The build can be submitted in two ways:

#### Option A: JSON submission (using server-side template)

Standard submission using a template already on the build server:
```bash
curl -s -X POST https://apk.g3h.cloud/api/v1/builds \
  -H "X-API-Key: $SKILL_KEY" \
  -H "Content-Type: application/json" \
  -d '$JSON_BODY'
```

The JSON body supports these fields:
- `appTypeId` (required), `appName` (required), `packageId` (required)
- `versionName`, `versionCode`, `keystoreId`, `templateVersionId`
- `buildFormat` — `"aab"` or `"apk"` (default `"apk"`)
- `configValues` (object of key/value pairs matching the config schema)
- `iconPath` (filename of an existing asset)
- `iconUrl` (URL to download an image — will be fetched and used as the icon automatically)

#### Option B: Multipart submission (uploading project source)

If the user wants to build from a local project directory (e.g., their `frontend/android` folder), zip it up and upload it:
```bash
# Zip the Android project
cd /path/to/project
zip -r /tmp/android-project.zip frontend/android/

# Or just the android directory itself
cd /path/to/project/frontend/android
zip -r /tmp/android-project.zip .

# Submit with the project archive
curl -s -X POST https://apk.g3h.cloud/api/v1/builds \
  -H "X-API-Key: $SKILL_KEY" \
  -F "appTypeId=1" \
  -F "appName=My App" \
  -F "packageId=com.example.app" \
  -F "versionName=1.0.0" \
  -F "buildFormat=aab" \
  -F "keystoreId=1" \
  -F 'configValues={"apiBaseUrl":"https://api.example.com"}' \
  -F "projectArchive=@/tmp/android-project.zip"
```

You can also upload an icon file in the same request:
```bash
curl -s -X POST https://apk.g3h.cloud/api/v1/builds \
  -H "X-API-Key: $SKILL_KEY" \
  -F "appTypeId=1" \
  -F "appName=My App" \
  -F "packageId=com.example.app" \
  -F "iconFile=@/path/to/icon.png" \
  -F "projectArchive=@/tmp/android-project.zip"
```

**Notes on project archives:**
- The archive can contain the project at the root, or inside a single top-level directory
- If the archive contains a `frontend/android/` subdirectory, it will be detected and used automatically
- The project must contain `build.gradle` or `settings.gradle` (native Android project)
- Archives can be `.zip`, `.tar`, `.tar.gz`, or `.tgz`
- Max upload size: 200MB

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

### Step 8: Download the output

On `completed`:
```bash
# Extension is .aab for AAB builds, .apk for APK builds
curl -s -L -o "$APP_NAME-$VERSION.$EXT" \
  https://apk.g3h.cloud/api/v1/builds/$BUILD_ID/download \
  -H "X-API-Key: $SKILL_KEY"
```

Save to the project root. Tell the user the file path and size.

For Play Store submissions (AAB), remind the user to upload the `.aab` file to Google Play Console and enroll in Play App Signing.

## Argument handling

If the user passes arguments (e.g., `/build-apk 2.1.0`), treat the first argument as the `versionName` override.
