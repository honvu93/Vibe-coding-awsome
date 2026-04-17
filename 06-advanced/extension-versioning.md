# Extension and Build Versioning

## The Problem

When your project produces multiple build artifacts from a single codebase — browser
extensions, mobile apps, desktop apps, or any multi-target build system — you need:

- Clear versioning so you know exactly what is deployed where
- Snapshot directories that allow immediate rollback to a previous version
- A reliable way to distinguish build targets in source code without spaghetti
  conditionals

## Multi-Target Builds from One Codebase

The core pattern: one source tree, multiple build configurations, multiple output
artifacts. The same logic applies to:

- Browser extension: Community edition vs Pro edition
- Mobile app: Free vs Premium app store builds
- Desktop app: Open source vs Enterprise
- CLI tool: Self-hosted vs SaaS-connected
- Any web app with meaningfully different feature sets per deployment target

## Build Target Differentiation

Inject the build target as an environment variable at build time. Every major build
tool supports this pattern.

```javascript
// vite.config.js
export default defineConfig({
  define: {
    'import.meta.env.VITE_BUILD_TARGET': JSON.stringify(
      process.env.BUILD_TARGET || 'community'
    )
  }
})
```

```javascript
// webpack.config.js
new webpack.DefinePlugin({
  BUILD_TARGET: JSON.stringify(process.env.BUILD_TARGET || 'community')
})
```

In source code, encapsulate the check in a single utility file. Never scatter raw
environment variable reads throughout the codebase:

```javascript
// src/lib/build-target.js
export const BUILD_TARGET = import.meta.env.VITE_BUILD_TARGET
export const IS_PRO = BUILD_TARGET === 'pro'
export const IS_COMMUNITY = BUILD_TARGET === 'community'
```

Feature-gating then reads clearly:

```javascript
import { IS_PRO } from '../lib/build-target.js'

function getAvailableExportFormats() {
  if (IS_PRO) return ['mp4', 'webm', 'mov', 'gif']
  return ['mp4']
}
```

This approach allows the build tool's dead-code elimination to strip target-specific
code from the output that does not apply to that build.

## Version Numbering

Pick one scheme and never deviate. Common options:

| Scheme | Example progression | Best for |
|---|---|---|
| Major.Minor | 3.0, 3.01, 3.1, 3.11, 4.0 | Desktop/extension apps with infrequent major bumps |
| SemVer | 1.0.0, 1.0.1, 1.1.0, 2.0.0 | Libraries, APIs, anything with a public contract |
| CalVer | 2026.04, 2026.04.1 | Services that ship continuously |

Rules that apply regardless of scheme:

1. Every build that ships to users **requires a version bump** before the build runs.
2. The version must be **identical across all files that declare it**: `package.json`,
   build config, manifest files (`manifest.json` for extensions, `Info.plist` for iOS,
   `build.gradle` for Android).
3. The version bump commit happens **before** the build, not after.

Version mismatch between declaring files is one of the most common causes of "I thought
I deployed the new version" incidents.

## Snapshot Directories

`dist/` is ephemeral — it gets overwritten on every build. Snapshot directories are
permanent per-version archives:

```
project/
├── dist/                           # Current working build (overwritten each time)
├── builds/
│   ├── community-v3.21/           # Old snapshot — kept for rollback
│   ├── community-v3.22/           # Previous release
│   ├── community-v3.23/           # Current release
│   ├── pro-v3.21/
│   ├── pro-v3.22/
│   └── pro-v3.23/
```

Why snapshots matter:

- **Rollback**: If v3.23 has a critical bug, load v3.22 immediately while the fix is
  prepared. No rebuild required, no waiting for CI.
- **Comparison**: Diff two snapshot directories to understand exactly what changed
  between releases.
- **Distribution traceability**: Each snapshot is a complete, ready-to-deploy artifact
  with its version embedded in the directory name.

For browser extensions specifically, the snapshot directory is what you zip and submit
to the Chrome Web Store or Firefox Add-ons. Keeping it means you can always re-submit
a previous version.

## Build Scripts

```bash
# Build both targets
npm run build:community   # dist/ + builds/community-v{version}/
npm run build:pro         # dist/ + builds/pro-v{version}/
```

Implementation in `package.json`:

```json
{
  "scripts": {
    "build:community": "BUILD_TARGET=community vite build && node scripts/snapshot.js community",
    "build:pro": "BUILD_TARGET=pro vite build && node scripts/snapshot.js pro"
  }
}
```

```javascript
// scripts/snapshot.js
import { cpSync, mkdirSync } from 'node:fs'
import { createRequire } from 'node:module'

const require = createRequire(import.meta.url)
const { version } = require('../package.json')
const [,, target] = process.argv

const dest = `builds/${target}-v${version}`
mkdirSync(dest, { recursive: true })
cpSync('dist', dest, { recursive: true })
console.log(`Snapshot written to ${dest}`)
```

For simpler projects, a shell one-liner in `package.json` is fine:

```json
"build:community": "BUILD_TARGET=community vite build && cp -r dist builds/community-v$(node -p \"require('./package.json').version\")"
```

## The Build Checklist

Before every build that will be distributed:

1. Bump version in all declaring files (`package.json`, build config, manifest)
2. Run the full test suite — do not build from a failing state
3. Run linting and type checks
4. Build all targets
5. Spot-check each build output (open in browser, run the CLI, install the extension)
6. Commit with version in the message: `chore: bump to v3.23`
7. Tag the commit: `git tag v3.23`
8. Keep snapshot directories — do not delete old builds until you have confirmed the
   new release is stable

## Anti-Patterns

**Building without bumping the version.** You cannot distinguish which build is deployed
where. Support tickets become unsolvable.

**Version mismatch across declaring files.** The OS, browser, or app store reads one
file; your analytics reads another. You ship v3.23 with a manifest that says v3.22.

**Only keeping `dist/`.** One bad deploy and you have no rollback path that does not
involve a full rebuild. Full rebuilds can fail.

**Raw environment variable reads scattered throughout source code.** When you need to
find all the places a feature is gated, you grep for a string instead of reading one
file. Extract to a central `build-target.js` utility.

**Committing `dist/` but not versioned snapshots.** `dist/` is always the latest build
for exactly one version. It tells you nothing about what was deployed last week.

**No build target differentiation in code.** Feature differences implemented as
runtime configuration that can be toggled by end users, rather than stripped at build
time. This is a security and licensing risk for anything gated as a paid feature.

## Storage Considerations

Snapshot directories accumulate. For projects with large build outputs (Electron apps,
React Native bundles), storage cost is real. Policies to consider:

- Keep the last N releases per target (e.g., last 5)
- Keep all releases from the current major version, prune previous majors
- Store snapshots in an artifact registry (S3, GitHub Releases, Artifactory) instead
  of in the repository

For browser extensions, the full zip is typically under a few MB. Keeping all versions
in the repository is practical and the traceability is worth the storage.

## When This Applies

- Browser extensions with multiple editions or permission levels
- Electron desktop apps (free vs pro, or different OS targets with different features)
- Mobile apps with different app store builds (free, paid, enterprise MDM)
- Any project where two outputs from the same source code have materially different
  features and need independent versioning and rollback capability

## When This Is Overkill

- Single-target web applications — use CI/CD artifact storage instead
- Open source libraries — npm/PyPI/crates.io versioning handles this better
- Internal tools with one deployment target and a short deployment history
- Prototype/throw-away projects where rollback is not a requirement
