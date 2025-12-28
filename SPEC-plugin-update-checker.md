# Specification: OpenCode Plugin Update Checker

## Overview

A plugin + CLI tool for OpenCode that checks for updates to pinned plugins and notifies users when newer versions are available.

## Problem

OpenCode's plugin version resolution works as follows:
- **Unpinned plugins** (`"my-plugin"` or `"my-plugin@latest"`): Fetches from npm on every startup
- **Pinned plugins** (`"my-plugin@1.2.3"`): Cached forever, never checks for updates

Users who pin for stability have no visibility into available updates.

## Solution

A plugin that:
1. Checks npm registry for newer versions of pinned plugins on startup
2. Shows a TUI toast notification when updates are available
3. Provides a tool to update the config file with new versions

A CLI that:
1. Checks for updates and outputs to terminal
2. Can apply updates to config file

---

## Technical Context

### OpenCode Plugin API

**Location:** `packages/plugin/src/index.ts`

```typescript
export type PluginInput = {
  client: ReturnType<typeof createOpencodeClient>
  project: Project
  directory: string
  worktree: string
  $: BunShell
}

export type Plugin = (input: PluginInput) => Promise<Hooks>
```

**Relevant Hooks:**

```typescript
export interface Hooks {
  config?: (input: Config) => Promise<void>  // Called on startup
  tool?: { [key: string]: ToolDefinition }   // Register custom tools
}
```

### Toast Notifications

```typescript
await client.tui.showToast({
  body: {
    title: "Plugin Updates Available",
    message: "2 plugins have updates available",
    variant: "info",  // "info" | "success" | "warning" | "error"
    duration: 5000    // milliseconds
  }
})
```

### OpenCode Config Locations

- Global: `~/.config/opencode/opencode.json` or `opencode.jsonc`
- Project: `.opencode/opencode.json` or `opencode.jsonc`

Config structure:

```json
{
  "plugin": [
    "opencode-beads@0.3.2",
    "some-other-plugin@1.0.0",
    "unpinned-plugin"
  ]
}
```

### npm Registry API

```
GET https://registry.npmjs.org/{package-name}/latest
Response: { "version": "1.2.3", ... }

GET https://registry.npmjs.org/{package-name}
Response: { "dist-tags": { "latest": "1.2.3" }, "versions": { ... } }
```

---

## Implementation

### Project Structure

```
opencode-plugin-updates/
├── package.json
├── tsconfig.json
├── src/
│   ├── index.ts           # Plugin entry point
│   ├── cli.ts             # CLI entry point
│   ├── core/
│   │   ├── check.ts       # Version checking logic
│   │   ├── config.ts      # Config file read/write
│   │   └── registry.ts    # npm registry client
│   └── types.ts
```

### Core Logic (`src/core/`)

**`registry.ts`** - npm registry client:

```typescript
interface PackageInfo {
  name: string
  latest: string
}

async function getLatestVersion(packageName: string): Promise<string>
async function getPackageInfo(packageName: string): Promise<PackageInfo>
```

**`config.ts`** - Config file operations:

```typescript
interface PluginEntry {
  raw: string           // "opencode-beads@0.3.2"
  name: string          // "opencode-beads"
  version: string | null // "0.3.2" or null if unpinned
  isPinned: boolean
}

interface ConfigFile {
  path: string
  plugins: PluginEntry[]
}

function parsePluginString(plugin: string): PluginEntry
function findConfigFiles(): Promise<ConfigFile[]>  // global + project
function updatePluginVersion(configPath: string, pluginName: string, newVersion: string): Promise<void>
```

**`check.ts`** - Update checking:

```typescript
interface UpdateInfo {
  name: string
  currentVersion: string
  latestVersion: string
  configPath: string      // which config file it's in
}

async function checkForUpdates(configs: ConfigFile[]): Promise<UpdateInfo[]>
```

### Plugin (`src/index.ts`)

```typescript
import { Plugin, tool } from "@opencode-ai/plugin"
import { findConfigFiles, updatePluginVersion } from "./core/config"
import { checkForUpdates } from "./core/check"

export const PluginUpdates: Plugin = async (input) => {
  const { client } = input

  return {
    async config() {
      // Run update check on startup
      const configs = await findConfigFiles()
      const updates = await checkForUpdates(configs)

      if (updates.length > 0) {
        await client.tui.showToast({
          body: {
            title: "Plugin Updates Available",
            message: `${updates.length} plugin(s) can be updated`,
            variant: "info",
            duration: 5000
          }
        })
      }
    },

    tool: {
      "plugins_check_updates": tool({
        description: "Check for available plugin updates",
        args: {},
        async execute() {
          const configs = await findConfigFiles()
          const updates = await checkForUpdates(configs)

          if (updates.length === 0) {
            return "All plugins are up to date"
          }

          return updates.map(u =>
            `${u.name}: ${u.currentVersion} → ${u.latestVersion}`
          ).join("\n")
        }
      }),

      "plugins_update": tool({
        description: "Update a plugin to its latest version",
        args: {
          name: tool.schema.string().describe("Plugin name to update"),
        },
        async execute({ name }) {
          const configs = await findConfigFiles()
          const updates = await checkForUpdates(configs)
          const update = updates.find(u => u.name === name)

          if (!update) {
            return `No update available for ${name}`
          }

          await updatePluginVersion(update.configPath, name, update.latestVersion)
          return `Updated ${name} to ${update.latestVersion}. Restart OpenCode to apply.`
        }
      }),

      "plugins_update_all": tool({
        description: "Update all plugins to their latest versions",
        args: {},
        async execute() {
          const configs = await findConfigFiles()
          const updates = await checkForUpdates(configs)

          if (updates.length === 0) {
            return "All plugins are up to date"
          }

          for (const update of updates) {
            await updatePluginVersion(update.configPath, update.name, update.latestVersion)
          }

          return `Updated ${updates.length} plugin(s). Restart OpenCode to apply.`
        }
      })
    }
  }
}
```

### CLI (`src/cli.ts`)

```typescript
#!/usr/bin/env bun
import { findConfigFiles, updatePluginVersion } from "./core/config"
import { checkForUpdates } from "./core/check"

const args = process.argv.slice(2)
const command = args[0]

async function main() {
  const configs = await findConfigFiles()
  const updates = await checkForUpdates(configs)

  switch (command) {
    case "check":
    case undefined:
      if (updates.length === 0) {
        console.log("✓ All plugins are up to date")
        return
      }
      console.log("Updates available:\n")
      for (const u of updates) {
        console.log(`  ${u.name}: ${u.currentVersion} → ${u.latestVersion}`)
        console.log(`    Config: ${u.configPath}`)
      }
      break

    case "update":
      const pluginName = args[1]
      if (pluginName) {
        // Update specific plugin
        const update = updates.find(u => u.name === pluginName)
        if (!update) {
          console.log(`No update available for ${pluginName}`)
          return
        }
        await updatePluginVersion(update.configPath, pluginName, update.latestVersion)
        console.log(`Updated ${pluginName} to ${update.latestVersion}`)
      } else {
        // Update all
        if (updates.length === 0) {
          console.log("All plugins are up to date")
          return
        }
        for (const update of updates) {
          await updatePluginVersion(update.configPath, update.name, update.latestVersion)
          console.log(`Updated ${update.name} to ${update.latestVersion}`)
        }
      }
      console.log("\nRestart OpenCode to apply changes.")
      break

    case "help":
    default:
      console.log(`
opencode-plugin-updates - Check and update OpenCode plugins

Commands:
  check           Check for available updates (default)
  update [name]   Update all plugins, or a specific plugin
  help            Show this help message
`)
  }
}

main().catch(console.error)
```

### package.json

```json
{
  "name": "opencode-plugin-updates",
  "version": "0.1.0",
  "type": "module",
  "main": "dist/index.js",
  "bin": {
    "opencode-plugin-updates": "dist/cli.js"
  },
  "exports": {
    ".": "./dist/index.js"
  },
  "scripts": {
    "build": "bun build src/index.ts --outdir dist && bun build src/cli.ts --outdir dist",
    "dev": "bun run src/index.ts"
  },
  "dependencies": {
    "@opencode-ai/plugin": "latest"
  },
  "devDependencies": {
    "typescript": "^5.0.0"
  },
  "peerDependencies": {
    "opencode-ai": "*"
  }
}
```

---

## Edge Cases to Handle

1. **Network failures** - Gracefully handle registry fetch failures, don't block startup
2. **Private registries** - Respect `.npmrc` settings (Bun handles this automatically)
3. **Scoped packages** - Handle `@scope/package-name@1.0.0` correctly
4. **JSONC comments** - Config files may have comments, preserve them on write
5. **Multiple configs** - Same plugin pinned differently in global vs project config
6. **Unpinned plugins** - Skip these (they already get latest)
7. **Plugin checking itself** - Don't get into infinite loops

---

## User Experience

**On startup (if updates available):**

```
┌─────────────────────────────────────┐
│ Plugin Updates Available            │
│ 2 plugin(s) can be updated          │
└─────────────────────────────────────┘
```

**Tool usage in chat:**

```
User: Check for plugin updates
Assistant: I found 2 plugins with available updates:

  opencode-beads: 0.3.2 → 0.4.0
  some-plugin: 1.0.0 → 1.2.0

Would you like me to update them?

User: Yes, update all

Assistant: Updated 2 plugins. Restart OpenCode to apply changes.
```

**CLI usage:**

```bash
# Check for updates
opencode-plugin-updates check

# Update all plugins
opencode-plugin-updates update

# Update specific plugin
opencode-plugin-updates update opencode-beads
```
