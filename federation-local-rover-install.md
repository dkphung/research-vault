---
tags: [graphql]
date: 2024-12-22
status: complete
---

# Federation Local Rover Installation via federation/package.json

> **SUPERSEDED (2024-12-08)**: This approach has been replaced with the simpler `pnpm dlx` method.
> See [pnpm-dlx-rover-approach.md](./pnpm-dlx-rover-approach.md) for the current recommendation.

**Date:** 2024-12-05
**Status:** Superseded
**Author:** Claude + Human

## Problem Statement

Currently, the federation setup requires developers to:
1. Install Rover CLI globally (`~/.rover/bin/rover`) via curl
2. Have Node.js and pnpm installed to run the compose script
3. Manage Rover version updates manually

This creates friction and pollutes the global scope. We want:
- Rover available when running `pnpm compose`
- Installed locally to the project (not global)
- No changes to subgraph's root `package.json` dependencies
- Seamless installation without user interaction

## Solution: federation/package.json with @apollo/rover

Add a separate `package.json` inside the gitignored `federation/` folder with `@apollo/rover` as a dependency. The compose script automatically creates this and runs `pnpm install` on first use.

## How It Works

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           SUBGRAPH REPO                                      │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   my-subgraph/                                                              │
│   ├── package.json              # Main project (UNTOUCHED dependencies)     │
│   │   └── scripts.compose: "pnpm dlx tsx scripts/compose-federation..."     │
│   │                                                                         │
│   ├── scripts/                                                              │
│   │   └── compose-federation-schema.ts   # Compose script                   │
│   │                                                                         │
│   └── federation/               # GITIGNORED - created on --fetch           │
│       ├── package.json          # Contains @apollo/rover dependency         │
│       ├── pnpm-lock.yaml        # Lock file for federation deps             │
│       ├── node_modules/                                                     │
│       │   └── .bin/                                                         │
│       │       └── rover         # <-- Rover binary lives here!              │
│       ├── router.yaml                                                       │
│       ├── supergraph.yaml                                                   │
│       ├── supergraph.graphql                                                │
│       └── subgraphs/                                                        │
│           └── *.schema.graphql                                              │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Flow

1. Developer runs: `pnpm compose --fetch`
2. Script fetches router.yaml, supergraph.yaml, subgraphs/ from federation-router
3. Script checks if `federation/node_modules/.bin/rover` exists
4. If not, creates `federation/package.json` with `@apollo/rover`
5. Script runs: `cd federation && pnpm install`
6. Rover is now at `federation/node_modules/.bin/rover`
7. Script uses that rover to compose supergraph
8. Docker router starts

On subsequent runs:
- `federation/node_modules` already exists
- Skip install, just compose (fast!)

## Implementation Details

### Change 1: `templates/subgraph/scripts/compose-federation-schema.ts`

Add the `ensureRover()` function:

```typescript
// Configuration
const ROVER_VERSION = "0.37.0";

// Paths
const FEDERATION_DIR = resolve(ROOT_DIR, "federation");
const FEDERATION_PKG_JSON = resolve(FEDERATION_DIR, "package.json");
const ROVER_BIN = resolve(FEDERATION_DIR, "node_modules", ".bin", "rover");

// Package.json content for federation folder
const FEDERATION_PACKAGE_JSON = {
  name: "federation-tools",
  version: "1.0.0",
  private: true,
  description: "Local federation tooling - this folder is gitignored",
  dependencies: {
    "@apollo/rover": ROVER_VERSION,
  },
};

function ensureRover(): void {
  // Check if rover binary exists
  if (existsSync(ROVER_BIN)) {
    return;
  }

  console.log("Setting up Rover CLI...");

  // Ensure federation directory exists
  mkdirSync(FEDERATION_DIR, { recursive: true });

  // Create or update package.json
  const currentPkg = existsSync(FEDERATION_PKG_JSON)
    ? JSON.parse(readFileSync(FEDERATION_PKG_JSON, "utf-8"))
    : {};

  const needsUpdate =
    !currentPkg.dependencies ||
    currentPkg.dependencies["@apollo/rover"] !== ROVER_VERSION;

  if (needsUpdate) {
    writeFileSync(
      FEDERATION_PKG_JSON,
      JSON.stringify(FEDERATION_PACKAGE_JSON, null, 2) + "\n"
    );
  }

  // Install dependencies
  console.log("Installing @apollo/rover...");
  execSync("pnpm install --silent", {
    cwd: FEDERATION_DIR,
    stdio: "pipe",
  });

  console.log(`✓ Rover ${ROVER_VERSION} installed`);
}
```

Update the `composeSchemas()` function to use the local rover:

```typescript
function composeSchemas(): void {
  if (!existsSync(FEDERATION_DIR)) {
    console.error("Error: federation/ directory not found. Run with --fetch first.");
    process.exit(1);
  }

  // Ensure rover is installed
  ensureRover();

  // ... rest of compose logic ...

  // Use local rover binary
  execSync(`${ROVER_BIN} supergraph compose --config ${SUPERGRAPH_CONFIG} > ${SUPERGRAPH_OUTPUT}`, {
    stdio: "inherit",
    cwd: FEDERATION_DIR,
    env: { ...process.env, APOLLO_ELV2_LICENSE: "accept" },
  });
}
```

Update `fetchFederationFiles()` to preserve node_modules:

```typescript
function fetchFederationFiles(): void {
  const tempDir = join(tmpdir(), `federation-${Date.now()}`);

  try {
    // ... fetch logic ...

    // Preserve node_modules if it exists (don't re-download rover)
    const nodeModulesBackup = join(tmpdir(), `federation-nm-${Date.now()}`);
    const hasNodeModules = existsSync(resolve(FEDERATION_DIR, "node_modules"));

    if (hasNodeModules) {
      cpSync(
        resolve(FEDERATION_DIR, "node_modules"),
        nodeModulesBackup,
        { recursive: true }
      );
    }

    // Also preserve package.json and lock file
    const pkgBackup = existsSync(FEDERATION_PKG_JSON)
      ? readFileSync(FEDERATION_PKG_JSON, "utf-8")
      : null;
    const lockBackup = existsSync(resolve(FEDERATION_DIR, "pnpm-lock.yaml"))
      ? readFileSync(resolve(FEDERATION_DIR, "pnpm-lock.yaml"), "utf-8")
      : null;

    // Clean and repopulate federation dir
    rmSync(FEDERATION_DIR, { recursive: true, force: true });
    mkdirSync(FEDERATION_DIR, { recursive: true });

    // Copy fetched files
    for (const path of FETCH_PATHS) {
      cpSync(join(tempDir, path), join(FEDERATION_DIR, path), { recursive: true });
    }

    // Restore node_modules, package.json, lock file
    if (hasNodeModules) {
      cpSync(nodeModulesBackup, resolve(FEDERATION_DIR, "node_modules"), { recursive: true });
      rmSync(nodeModulesBackup, { recursive: true, force: true });
    }
    if (pkgBackup) writeFileSync(FEDERATION_PKG_JSON, pkgBackup);
    if (lockBackup) writeFileSync(resolve(FEDERATION_DIR, "pnpm-lock.yaml"), lockBackup);
  } finally {
    rmSync(tempDir, { recursive: true, force: true });
  }
}
```

### Change 2: `federation-scripts/init-federation.sh`

Remove the global Rover installation section:

```bash
#!/bin/bash
set -e

REPO="git@github.com:risk-and-safety/federation-router.git"
BRANCH="trunk"
TEMPLATES_PATH="templates/subgraph"

echo "🚀 Initializing federation..."

# Prerequisites check (NO MORE ROVER CHECK!)
if ! command -v node &> /dev/null; then
  echo "❌ Node.js is required"
  exit 1
fi

if ! command -v pnpm &> /dev/null; then
  echo "❌ pnpm is required. Install with: npm install -g pnpm"
  exit 1
fi

if ! command -v docker &> /dev/null; then
  echo "❌ Docker is required"
  exit 1
fi

if [ ! -f "package.json" ]; then
  echo "❌ No package.json found"
  exit 1
fi

# REMOVED: Rover installation - compose script handles this now!

# Fetch templates
TEMP_DIR=$(mktemp -d)
trap "rm -rf $TEMP_DIR" EXIT

echo "Fetching templates from federation-router..."
git clone --depth 1 --branch "$BRANCH" --filter=blob:none --sparse "$REPO" "$TEMP_DIR" 2>/dev/null
git -C "$TEMP_DIR" sparse-checkout set --no-cone "$TEMPLATES_PATH" 2>/dev/null

# Copy files
mkdir -p scripts
cp "$TEMP_DIR/$TEMPLATES_PATH/scripts"/compose-federation-schema.ts scripts/
echo "✓ Copied scripts/compose-federation-schema.ts"

mkdir -p .github/workflows
cp "$TEMP_DIR/$TEMPLATES_PATH/.github/workflows"/compose-federation-schema.yml .github/workflows/
echo "✓ Copied .github/workflows/compose-federation-schema.yml"

# Update .gitignore
grep -q "^federation/$" .gitignore 2>/dev/null || echo "federation/" >> .gitignore
echo "✓ Updated .gitignore"

# Update package.json
node -e "
const fs = require('fs');
const pkg = JSON.parse(fs.readFileSync('package.json', 'utf-8'));
pkg.scripts = { ...pkg.scripts, compose: 'pnpm dlx tsx scripts/compose-federation-schema.ts' };
fs.writeFileSync('package.json', JSON.stringify(pkg, null, 2) + '\n');
console.log('✓ Updated package.json scripts');
"

# Run initial fetch and compose (this installs Rover automatically)
echo ""
echo "Fetching federation config and installing Rover..."
pnpm compose --fetch

echo ""
echo "✨ Federation initialized!"
echo "  pnpm compose         - Rebuild and reload"
echo "  pnpm compose --fetch - Fetch latest config"
echo "  pnpm compose --watch - Watch mode"
```

### Change 3: `templates/subgraph/.github/workflows/compose-federation-schema.yml`

Remove Rover installation steps:

```yaml
name: Federation Schema Composition

on:
  push:
    branches: [main, trunk]
    paths:
      - 'src/**/*.graphql'
      - 'graphql/**/*.graphql'
  workflow_dispatch:

jobs:
  compose-and-publish:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repo
        uses: actions/checkout@v4

      - name: Checkout federation-router repo
        uses: actions/checkout@v4
        with:
          repository: risk-and-safety/federation-router
          path: federation-router
          token: ${{ secrets.RSS_SVC_GITHUB_TOKEN }}
          sparse-checkout: |
            supergraph.yaml
            subgraphs

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Setup pnpm
        uses: pnpm/action-setup@v2
        with:
          version: 9

      # REMOVED: Cache Rover CLI
      # REMOVED: Install Rover CLI
      # REMOVED: Add Rover to PATH
      # The compose script handles all of this now!

      - name: Prepare federation config
        run: |
          mkdir -p federation/subgraphs
          cp federation-router/supergraph.yaml federation/
          cp federation-router/subgraphs/*.graphql federation/subgraphs/ 2>/dev/null || true

      - name: Compose schema
        run: pnpm dlx tsx scripts/compose-federation-schema.ts --skip-docker

      - name: Copy schema to federation-router
        run: |
          REPO_NAME=$(node -p "const p = require('./package.json'); p.name.startsWith('@') ? p.name.split('/')[1] : p.name")
          cp federation/subgraphs/$REPO_NAME.schema.graphql federation-router/subgraphs/

      - name: Commit and push to federation-router
        working-directory: federation-router
        run: |
          REPO_NAME=$(node -p "const p = require('../package.json'); p.name.startsWith('@') ? p.name.split('/')[1] : p.name")
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git add subgraphs/$REPO_NAME.schema.graphql
          git diff --staged --quiet || git commit -m "chore: update $REPO_NAME subgraph schema"
          git push
```

## Subgraph Structure After Setup

```
my-subgraph/
├── package.json                          # UNCHANGED (no new deps)
│   └── scripts.compose: "pnpm dlx tsx scripts/compose-federation-schema.ts"
│
├── scripts/
│   └── compose-federation-schema.ts      # The compose script
│
├── .github/workflows/
│   └── compose-federation-schema.yml     # CI workflow
│
├── .gitignore                            # Contains: federation/
│
├── src/
│   └── graphql/*.graphql                 # Your schemas
│
└── federation/                           # GITIGNORED
    ├── package.json                      # {"dependencies": {"@apollo/rover": "0.37.0"}}
    ├── pnpm-lock.yaml
    ├── node_modules/
    │   ├── @apollo/rover/
    │   └── .bin/
    │       └── rover                     # Rover binary (~50MB)
    ├── router.yaml
    ├── supergraph.yaml
    ├── supergraph.graphql
    └── subgraphs/*.schema.graphql
```

## User Experience

### First Time Setup

```bash
$ curl -sSL .../init-federation.sh | bash

🚀 Initializing federation...

Fetching templates from federation-router...
✓ Copied scripts/compose-federation-schema.ts
✓ Copied .github/workflows/compose-federation-schema.yml
✓ Updated .gitignore
✓ Updated package.json scripts

Fetching federation config and installing Rover...
Fetching from federation-router...
  ✓ router.yaml
  ✓ supergraph.yaml
  ✓ subgraphs

Setting up Rover CLI...
Installing @apollo/rover...
✓ Rover 0.37.0 installed

Found 3 GraphQL file(s)
✓ Combined into my-subgraph.schema.graphql

Composing supergraph...
✓ Composed supergraph.graphql

Starting federation-router...
✓ Router started at http://localhost:4000

✨ Federation initialized!
```

### Subsequent Runs (Fast)

```bash
$ pnpm compose

Found 3 GraphQL file(s)
✓ Combined into my-subgraph.schema.graphql

Composing supergraph...
✓ Composed supergraph.graphql

Restarting federation-router...
✓ Router restarted at http://localhost:4000
```

## Comparison with Other Approaches

| Aspect | Global Install (current) | federation/package.json | Bun Compiled |
|--------|-------------------------|------------------------|--------------|
| Rover location | `~/.rover/bin/` | `federation/node_modules/.bin/` | `federation/bin/` |
| Pollutes global | Yes | No | No |
| User dependencies | Node, pnpm, curl | Node, pnpm | None |
| Main pkg.json changes | None | None | None |
| Version pinning | No (uses latest) | Yes (in federation/package.json) | Yes (in binary) |
| Offline after setup | No | Yes | Yes |
| CI complexity | Cache rover, install, add to PATH | Just run compose | Download binary |
| Maintenance | N/A | Update ROVER_VERSION in .ts | Build & release |

## Advantages of This Approach

1. **Simple** - No build pipeline, just update the TypeScript template
2. **Version pinned** - Rover version controlled in the script
3. **Local to project** - No global pollution
4. **Leverages existing tools** - Uses pnpm which subgraphs already have
5. **Cached** - pnpm caches packages, fast reinstalls
6. **CI simplified** - No more Rover install steps in workflow

## Disadvantages

1. **Requires Node.js + pnpm** - Unlike Bun compiled approach
2. **node_modules in federation/** - Adds ~100MB to federation/ folder
3. **First run slower** - Downloads Rover package (~50MB)

## Files Changed Summary

| File | Change |
|------|--------|
| `templates/subgraph/scripts/compose-federation-schema.ts` | Add `ensureRover()`, update paths, preserve node_modules on fetch |
| `federation-scripts/init-federation.sh` | Remove Rover installation section |
| `templates/subgraph/.github/workflows/compose-federation-schema.yml` | Remove Rover cache/install/PATH steps |

## Next Steps

1. [x] Update `templates/subgraph/scripts/compose-federation-schema.ts`
2. [x] Update `federation-scripts/init-federation.sh`
3. [x] Update `templates/subgraph/.github/workflows/compose-federation-schema.yml`
4. [ ] Test on fresh subgraph repo
5. [ ] Update documentation
