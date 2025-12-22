# Bun-Compiled Compose Federation CLI

**Date:** 2024-12-05
**Status:** Research
**Author:** Claude + Human

## Problem Statement

Currently, the federation setup requires developers to:
1. Install Rover CLI globally (`~/.rover/bin/rover`) via curl
2. Have Node.js and pnpm installed to run the compose script
3. Use `tsx` to execute TypeScript

This creates friction and pollutes the global scope. We want:
- Rover available when running `pnpm compose`
- Installed locally to the project (not global)
- No changes to subgraph's root `package.json` dependencies
- Seamless installation without user interaction

## Solution: Bun-Compiled Standalone Binary

Use Bun's `--compile` feature to create a self-contained executable that:
1. Includes the Bun runtime (no Node.js needed)
2. Downloads Rover to `federation/bin/` on first run
3. Handles all compose operations

### How Bun Compile Works

```bash
bun build --compile ./src/compose-federation.ts --outfile compose-federation
```

This creates a single binary (~60MB) containing:
- Bun runtime
- All TypeScript code
- All bundled dependencies

**No Node.js, pnpm, or tsx needed to run it.**

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           BUILD TIME (CI)                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   src/compose-federation.ts                                                 │
│            │                                                                │
│            │  bun build --compile                                           │
│            ▼                                                                │
│   ┌─────────────────────────────────────────────────────────────┐          │
│   │  compose-federation-darwin-arm64  (macOS Apple Silicon)     │          │
│   │  compose-federation-darwin-x64    (macOS Intel)             │ ──────┐  │
│   │  compose-federation-linux-x64     (Linux for CI)            │       │  │
│   └─────────────────────────────────────────────────────────────┘       │  │
│                                                                         │  │
│                                            Uploaded to GitHub Releases ◄┘  │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│                        RUNTIME (developer's machine)                         │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   Developer runs: curl ... | bash  (init-federation.sh)                     │
│            │                                                                │
│            │  Detects platform (darwin-arm64, etc.)                         │
│            │  Downloads correct binary from GitHub Releases                 │
│            ▼                                                                │
│   subgraph-repo/                                                            │
│   ├── scripts/                                                              │
│   │   └── compose-federation   ◄── Downloaded binary (~60MB)                │
│   ├── package.json             ◄── Updated: "compose": "./scripts/..."      │
│   └── federation/              ◄── Created on first `pnpm compose`          │
│       ├── bin/rover            ◄── Downloaded automatically                 │
│       ├── router.yaml                                                       │
│       ├── supergraph.yaml                                                   │
│       ├── supergraph.graphql                                                │
│       └── subgraphs/*.graphql                                               │
│                                                                             │
│   Developer runs: pnpm compose                                              │
│            │                                                                │
│            │  Executes ./scripts/compose-federation                         │
│            │  (No Node.js, pnpm, or bun needed - it's self-contained!)      │
│            ▼                                                                │
│   Federation composed, router started at http://localhost:4000              │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Repository Structure After Implementation

```
federation-router/
├── .github/workflows/
│   ├── build-compose-cli.yml              # NEW: Builds the binaries
│   ├── sync-to-deployments-staging.yml
│   └── sync-to-deployments-production.yml
│
├── src/                                   # NEW: Source for compiled CLI
│   └── compose-federation.ts              # The main script (Bun)
│
├── templates/subgraph/                    # SIMPLIFIED
│   └── .github/workflows/
│       ├── compose-federation-schema.yml  # Updated to use binary
│       └── promote-to-production.yml
│   # NOTE: scripts/ folder removed - binary downloaded instead
│
├── subgraphs/
├── router.yaml
├── supergraph.yaml
└── ...

federation-scripts/
├── init-federation.sh                     # Updated to download binary
└── README.md
```

## Implementation Details

### File 1: `src/compose-federation.ts`

The main TypeScript source that gets compiled:

```typescript
#!/usr/bin/env bun
/**
 * Compose Federation CLI
 *
 * A standalone CLI for composing Apollo Federation supergraphs.
 * Compiled with `bun build --compile` into a self-contained executable.
 *
 * Usage:
 *   ./compose-federation              # compose + start router
 *   ./compose-federation --fetch      # fetch config first
 *   ./compose-federation --skip-docker
 *   ./compose-federation --watch
 */

import { existsSync, mkdirSync, readdirSync, statSync, watch } from "node:fs";
import { tmpdir } from "node:os";
import { dirname, join, relative, resolve } from "node:path";
import { $ } from "bun";

// Configuration
const ROVER_VERSION = "0.37.0";
const ROUTER_IMAGE = "ghcr.io/apollographql/router:v2.9.0";
const ROUTER_PORT = 4000;
const CONTAINER_NAME = "federation-router";

const FEDERATION_REPO = "git@github.com:risk-and-safety/federation-router.git";
const FEDERATION_BRANCH = "trunk";
const FETCH_PATHS = ["router.yaml", "supergraph.yaml", "subgraphs"];

// Paths
const ROOT_DIR = process.cwd();
const SRC_DIR = resolve(ROOT_DIR, "src");
const GRAPHQL_DIR = resolve(ROOT_DIR, "graphql");
const FEDERATION_DIR = resolve(ROOT_DIR, "federation");
const SUBGRAPHS_DIR = resolve(FEDERATION_DIR, "subgraphs");
const ROVER_BIN = resolve(FEDERATION_DIR, "bin", "rover");
const SUPERGRAPH_CONFIG = resolve(FEDERATION_DIR, "supergraph.yaml");
const SUPERGRAPH_OUTPUT = resolve(FEDERATION_DIR, "supergraph.graphql");

// Rover Installation
function getRoverDownloadUrl(): string {
  const platform = process.platform;
  const arch = process.arch;

  let target: string;
  if (platform === "darwin") {
    target = arch === "arm64" ? "aarch64-apple-darwin" : "x86_64-apple-darwin";
  } else if (platform === "linux") {
    target = arch === "arm64" ? "aarch64-unknown-linux-gnu" : "x86_64-unknown-linux-gnu";
  } else {
    throw new Error(`Unsupported platform: ${platform}-${arch}`);
  }

  return `https://github.com/apollographql/rover/releases/download/v${ROVER_VERSION}/rover-v${ROVER_VERSION}-${target}.tar.gz`;
}

async function ensureRover(): Promise<void> {
  if (existsSync(ROVER_BIN)) return;

  console.log("Installing Rover CLI...");
  const binDir = dirname(ROVER_BIN);
  mkdirSync(binDir, { recursive: true });

  const url = getRoverDownloadUrl();
  await $`curl -sSL ${url} | tar -xz -C ${binDir} --strip-components=1 dist/rover`.quiet();
  await $`chmod +x ${ROVER_BIN}`;

  console.log(`✓ Rover ${ROVER_VERSION} installed`);
}

// ... rest of implementation (fetch, compose, docker, watch stages)
```

### File 2: `.github/workflows/build-compose-cli.yml`

```yaml
name: Build Compose CLI

on:
  push:
    tags: ["v*"]
  workflow_dispatch:
    inputs:
      version:
        description: "Version tag (e.g., v1.0.0)"
        required: true

jobs:
  build:
    strategy:
      matrix:
        include:
          - os: macos-14          # Apple Silicon runner
            target: darwin-arm64
            artifact: compose-federation-darwin-arm64
          - os: macos-13          # Intel runner
            target: darwin-x64
            artifact: compose-federation-darwin-x64
          - os: ubuntu-latest
            target: linux-x64
            artifact: compose-federation-linux-x64

    runs-on: ${{ matrix.os }}

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Bun
        uses: oven-sh/setup-bun@v2
        with:
          bun-version: latest

      - name: Build binary
        run: |
          bun build --compile \
            --minify \
            --sourcemap \
            ./src/compose-federation.ts \
            --outfile ${{ matrix.artifact }}

      - name: Test binary
        run: ./${{ matrix.artifact }} --help

      - name: Upload artifact
        uses: actions/upload-artifact@v4
        with:
          name: ${{ matrix.artifact }}
          path: ${{ matrix.artifact }}

  release:
    needs: build
    runs-on: ubuntu-latest
    permissions:
      contents: write

    steps:
      - name: Download all artifacts
        uses: actions/download-artifact@v4
        with:
          path: binaries
          merge-multiple: true

      - name: Create Release
        uses: softprops/action-gh-release@v2
        with:
          tag_name: ${{ github.event.inputs.version || github.ref_name }}
          files: binaries/*
          generate_release_notes: true
```

### File 3: Updated `init-federation.sh`

```bash
#!/bin/bash
set -e

REPO="risk-and-safety/federation-router"
BRANCH="trunk"

echo "🚀 Initializing federation..."

# Prerequisites
if ! command -v docker &> /dev/null; then
  echo "❌ Docker is required"
  exit 1
fi

if [ ! -f "package.json" ]; then
  echo "❌ No package.json found"
  exit 1
fi

# Detect platform
OS=$(uname -s | tr '[:upper:]' '[:lower:]')
ARCH=$(uname -m)
case "$ARCH" in
  x86_64)  ARCH="x64" ;;
  aarch64|arm64) ARCH="arm64" ;;
esac

BINARY_NAME="compose-federation-${OS}-${ARCH}"
DOWNLOAD_URL="https://github.com/${REPO}/releases/latest/download/${BINARY_NAME}"

# Download CLI binary
echo "Downloading compose-federation CLI..."
mkdir -p scripts
curl -sSL --fail "$DOWNLOAD_URL" -o scripts/compose-federation
chmod +x scripts/compose-federation
echo "✓ Downloaded scripts/compose-federation"

# Fetch workflow templates
TEMP_DIR=$(mktemp -d)
trap "rm -rf $TEMP_DIR" EXIT

git clone --depth 1 --branch "$BRANCH" --filter=blob:none --sparse \
  "git@github.com:${REPO}.git" "$TEMP_DIR" 2>/dev/null
git -C "$TEMP_DIR" sparse-checkout set --no-cone "templates/subgraph" 2>/dev/null

mkdir -p .github/workflows
cp "$TEMP_DIR/templates/subgraph/.github/workflows/compose-federation-schema.yml" .github/workflows/
echo "✓ Copied workflow"

# Update .gitignore
grep -q "^federation/$" .gitignore 2>/dev/null || echo "federation/" >> .gitignore
grep -q "^scripts/compose-federation$" .gitignore 2>/dev/null || echo "scripts/compose-federation" >> .gitignore

# Update package.json
node -e "
const fs = require('fs');
const pkg = JSON.parse(fs.readFileSync('package.json', 'utf-8'));
pkg.scripts = { ...pkg.scripts, compose: './scripts/compose-federation' };
fs.writeFileSync('package.json', JSON.stringify(pkg, null, 2) + '\n');
"
echo "✓ Updated package.json"

# Initial fetch and compose
./scripts/compose-federation --fetch

echo ""
echo "✨ Federation initialized!"
echo "  pnpm compose         - Rebuild and reload"
echo "  pnpm compose --fetch - Fetch latest config"
echo "  pnpm compose --watch - Watch mode"
```

### File 4: Updated CI Workflow

```yaml
name: Federation Schema Composition

on:
  push:
    branches: [main, trunk]
    paths:
      - "src/**/*.graphql"
      - "graphql/**/*.graphql"
  workflow_dispatch:

jobs:
  compose-and-publish:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - uses: actions/checkout@v4
        with:
          repository: risk-and-safety/federation-router
          path: federation-router
          token: ${{ secrets.RSS_SVC_GITHUB_TOKEN }}
          sparse-checkout: |
            supergraph.yaml
            subgraphs

      - name: Download compose-federation CLI
        run: |
          mkdir -p scripts
          curl -sSL --fail \
            "https://github.com/risk-and-safety/federation-router/releases/latest/download/compose-federation-linux-x64" \
            -o scripts/compose-federation
          chmod +x scripts/compose-federation

      - name: Prepare and compose
        run: |
          mkdir -p federation/subgraphs
          cp federation-router/supergraph.yaml federation/
          cp federation-router/subgraphs/*.graphql federation/subgraphs/ 2>/dev/null || true
          ./scripts/compose-federation --skip-docker

      - name: Publish schema
        working-directory: federation-router
        run: |
          REPO_NAME=$(node -p "require('../package.json').name.replace(/^@[^/]+\//, '')")
          cp ../federation/subgraphs/$REPO_NAME.schema.graphql subgraphs/
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git add subgraphs/$REPO_NAME.schema.graphql
          git diff --staged --quiet || git commit -m "chore: update $REPO_NAME subgraph"
          git push
```

## User Experience

### First Time Setup

```bash
$ curl -sSL .../init-federation.sh | bash

🚀 Initializing federation...

Downloading compose-federation CLI...
✓ Downloaded scripts/compose-federation

Fetching from federation-router...
  ✓ router.yaml
  ✓ supergraph.yaml
  ✓ subgraphs

Installing Rover CLI...              # Automatic!
✓ Rover 0.37.0 installed

Found 3 GraphQL file(s)
✓ Combined into my-subgraph.schema.graphql
✓ Composed supergraph.graphql

Starting federation-router...
✓ Router started at http://localhost:4000

✨ Federation initialized!
```

### Daily Development

```bash
$ pnpm compose --watch

👀 Watching for .graphql changes...

📝 Detected change: user.graphql
🔄 Recomposing...
✓ Ready
```

## Subgraph Structure After Setup

```
my-subgraph-repo/
├── package.json                    # "compose": "./scripts/compose-federation"
├── .gitignore                      # federation/, scripts/compose-federation
├── scripts/
│   └── compose-federation          # Binary (~60MB) - GITIGNORED
├── .github/workflows/
│   └── compose-federation-schema.yml
└── federation/                     # GITIGNORED
    ├── bin/rover                   # Downloaded (~50MB)
    ├── router.yaml
    ├── supergraph.yaml
    ├── supergraph.graphql
    └── subgraphs/*.graphql
```

## Comparison

| Aspect | Before | After (Bun Compiled) |
|--------|--------|---------------------|
| Developer dependencies | Node.js, pnpm, tsx | Docker only |
| Rover installation | Global (~/.rover) | Local (federation/bin/) |
| Compose script | TypeScript (needs tsx) | Standalone binary |
| CI Rover install | curl + cache | Download binary |
| Subgraph pkg.json deps | None added | None added |
| Offline after setup | No | Yes |

## Alternatives Considered

### Option 1: `pnpm dlx @apollo/rover`
- Cached in pnpm store, not truly project-local
- Requires pnpm installed

### Option 2: Download binary to `federation/bin/`
- More complex platform detection
- Need to handle permissions manually

### Option 3: `federation/package.json` with rover
- Adds node_modules to federation/
- Still requires pnpm install

## Risks and Mitigations

| Risk | Mitigation |
|------|------------|
| Binary size (~60MB) | Acceptable; smaller than node_modules |
| Platform coverage | Build for darwin-arm64, darwin-x64, linux-x64 |
| Version updates | Tag releases, CI auto-builds |
| Bun runtime bugs | Bun is stable; fallback to tsx if needed |

## Next Steps

1. [ ] Create `src/compose-federation.ts` with full implementation
2. [ ] Add `.github/workflows/build-compose-cli.yml`
3. [ ] Update `templates/subgraph/.github/workflows/compose-federation-schema.yml`
4. [ ] Update `federation-scripts/init-federation.sh`
5. [ ] Create initial release with binaries
6. [ ] Test on fresh subgraph repo
7. [ ] Update README.md documentation
