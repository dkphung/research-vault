# Rover Installation via pnpm dlx (Recommended Approach)

**Date:** 2024-12-08
**Status:** Recommended Implementation
**Supersedes:** federation-local-rover-install.md

## The Approach

Use `pnpm dlx @apollo/rover@VERSION` to invoke Rover directly instead of managing a local package.json and node_modules.

## Why This is Better

**Simplicity**: The federation/package.json approach has ~80 lines of complexity:
- `ensureRover()` function to create package.json and run pnpm install
- Backup/restore logic for node_modules during fetch operations
- Package.json and lock file management

All of this can be replaced with a single line: `pnpm dlx @apollo/rover@${ROVER_VERSION}`

**Cleaner structure**: No generated package.json, no node_modules folder in federation/, no lock file to manage.

**Same functionality**: Version is still pinned, still cached by pnpm, still works offline after first run.

## Why federation/package.json Was Wrong

The original research claimed pnpm dlx was "not truly project-local" - but both approaches use pnpm's global cache. The node_modules folder is just hardlinks to that cache. They're functionally identical in terms of "locality" - one just adds 80 lines of unnecessary file management.

## Implementation

In `compose-federation-schema.ts`:
- Remove the `ensureRover()` function
- Remove the backup/restore logic in `fetchFederationFiles()`
- Change the rover invocation from `${ROVER_BIN}` to `pnpm dlx @apollo/rover@${ROVER_VERSION}`

Net result: ~90 fewer lines of code.

## Prerequisites (Unchanged)

Same as before: pnpm, git, docker, Node.js/Bun
