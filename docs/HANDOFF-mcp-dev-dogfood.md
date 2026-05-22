# Handoff: dogfood your MCP via project-scoped `.mcp.json`

**Audience:** Claude Code sessions working on an MCP server project alongside the human author.
**Goal:** Give the in-progress MCP server to *yourself* (the Claude Code session that is building it) so you can call its tools as part of the code-test loop — without waiting for the user to wire it up in their global config.

---

## The idea in one paragraph

When you build an MCP server, the natural feedback loop is: write code → run server → call a tool → see what it returns → iterate. Without dogfooding, that last step is missing in a Claude Code session — you write the code, you write a unit test of helpers, and you ask the user to verify the real tool behavior. With a project-scoped `.mcp.json` at the repo root, Claude Code can launch the server you just built and call its tools directly. The loop closes. You don't get hot-reload, and you can't always exercise the real external dependency (a Discord guild, a Slack workspace, an internal API), but you *can* exercise schemas, tool-list output, error paths, allow-list logic, sanitizers, formatters, and anything else that doesn't require live external state.

## Why this matters more for MCP projects than for normal apps

A normal app you can run with `node`, `python`, `pytest`, `npm test`, `curl`. You see output directly. An MCP server speaks JSON-RPC over stdio. Stand-alone invocation is awkward — you have to hand-craft `{"jsonrpc":"2.0","method":"tools/list",...}` and parse a streaming response. With `.mcp.json` registered, you just say "call the `foo_tool` with these args" and the harness handles the protocol. That's the difference between an interactive test cycle and a manual integration test.

## The pattern

### 1. Project-scoped `.mcp.json`

Put a file named exactly `.mcp.json` at the repo root. Claude Code auto-discovers it when started from inside the project, prompts the user for trust on first use, then makes the configured servers available to you.

Minimal shape:

```json
{
  "mcpServers": {
    "<short-server-name>-dev": {
      "command": "node",
      "args": ["build/index.js"],
      "env": {
        "REQUIRED_SECRET":   "${REQUIRED_SECRET}",
        "ALLOW_LIST_VAR":    "${ALLOW_LIST_VAR}"
      }
    }
  }
}
```

Notes:

- **Name the entry with a `-dev` suffix** (or similar) so it doesn't collide with the user's production-config entry of the same MCP elsewhere. They might run both side-by-side.
- **`args` is relative to the project root**, where `.mcp.json` lives. So `build/index.js`, `dist/main.js`, or `python -m my_mcp` all work without absolute paths.
- **`command` should match what `npm run start` or the README says to run.** Skip the `npm` wrapper if you can — `node build/index.js` is more direct and avoids extra process layers.
- **Use the build output, not the dev/ts-node entry.** You want to test what the build produces. If the build is broken, the MCP won't start, and that's a useful signal.

### 2. The single non-negotiable rule about secrets

**Never hardcode a token, API key, or any secret inside `.mcp.json`.** That file is checked into the repo. The only way to expose a secret through it is `${VAR}` interpolation that pulls from the user's shell environment at session-start time.

If your MCP needs `FOO_API_KEY`, write `"FOO_API_KEY": "${FOO_API_KEY}"` and tell the user (in the README, or in conversation) to export it before launching Claude Code. If they haven't, your server will start without it and either fail loudly or refuse all tool calls (which is itself a useful default behavior — see fail-closed below).

Non-secret env vars (allow-lists, feature flags, log levels) can use `${VAR}` for the same reason — different developers may want different values without editing the committed file.

If you ever need a developer-local override that *isn't* shareable, the right move is to add a `.mcp.local.json` pattern (Claude Code reads it if present and merges over `.mcp.json`) and gitignore that file. But default to keeping everything in the committed file with env-var interpolation.

### 3. The loop, with eyes open about limits

The development loop becomes:

1. Edit source.
2. `npm run build` (or your equivalent — `tsc`, `pyinstaller`, etc.).
3. In the Claude Code session, `/mcp` to disconnect/reconnect the dev server so it picks up the new build.
4. Call a tool. Observe the response.
5. Go to step 1.

What you **do not get**:

- **Hot reload.** Step 3 is mandatory after every build. If you forget, you'll see the old behavior and waste time wondering why your fix didn't take. Get in the habit of running build + reconnect together.
- **Real external state.** For an MCP that talks to Discord, GitHub, Slack, an internal API — the local server will try to connect using whatever credentials you set. If the user is happy to give you sandbox/test credentials, great, you get real round-trips. If not, you're stuck verifying everything that *doesn't* require the live system: schemas, error paths, allow-list rejection, sanitizer escape, formatter output, ZodError shapes.
- **Coverage for tool calls you can't safely make.** A destructive tool (delete, send, write) you probably don't want to fire against the real backing system. Stub it or rely on the fact that you've removed it (see "Read-only fork" patterns in any of the MCPs in this set).

What you **do** get:

- A real `tools/list` round-trip that confirms the server starts and advertises what you expect.
- A real call site for any tool that fails early — argument validation, allow-list rejection, missing-token paths. These run end-to-end without touching the external system.
- A natural way to verify changes to response shape: just call the tool and read the JSON.
- Pressure on yourself to keep the server startable and self-contained. If it depends on five env vars that nobody documented, you'll find out the first time you reconnect.

### 4. The Claude Code session's playbook

When you (the next session, or another instance of yourself) opens a project with this pattern in place, you should:

1. **Notice the `.mcp.json` and read it.** It tells you which env vars the dev server needs.
2. **Check whether the env vars are set.** Either ask the user, or just attempt a tool call and see what happens — most MCPs in this style fail loudly when their config is missing.
3. **Make your code change.**
4. **Build.** If the build fails, that's the test result; no need to reconnect.
5. **Reconnect the MCP** (instruct the user to `/mcp` and reconnect the `*-dev` entry — Claude Code's mcp slash command can also list and reconnect).
6. **Call the tools you changed**, with arguments that exercise the change. For a parser change, call a tool that uses the parser. For a new error message, call a tool with the failing input.
7. **Read the response.** This is the equivalent of looking at a test output.
8. **If the change is to behavior the user expects to verify themselves** (e.g. real Discord messages render correctly), ship the change to a branch and call it out — you can demonstrate the code path but not the user-facing experience.

### 5. What to put in the project README about this

Two or three lines is enough. Don't pre-document the whole pattern in every project's README — link to your team's canonical version. Something like:

```markdown
### Self-test loop

This project has a `.mcp.json` at the root. Export the env vars listed
inside (the values prefixed with `$`), then run Claude Code from this
directory. The dev MCP entry will be available; reconnect it after each
`npm run build` to pick up changes.
```

That's it. The pattern is generic; the README only needs to point at it and say "yes, this project uses it."

### 6. Anti-patterns to avoid

- **Committing `.mcp.json` with a real token in it.** This will happen at least once across an org doing this. Add a pre-commit hook or a CI check that fails if any value in `.mcp.json` doesn't match `^\$\{[A-Z_]+\}$` or one of a small list of safe literals.
- **Pointing `args` at `src/index.ts` via ts-node.** Tempting because no build step, but you lose the signal that the build is broken, and the dev runtime drifts from the shipped runtime. Build first, run the build output.
- **Including the dev MCP entry in the production `.mcp.json` example in the README.** The dev entry should not show up in any user-facing setup instructions — it's for the dev loop only. Production users get the deployment instructions (see this project's README for the corporate-deployment shape).
- **Treating dogfooding as a substitute for tests.** It is not. A `.mcp.json`-driven manual call cycle is great during development; a unit test of pure functions (sanitizers, parsers, formatters) is what protects you from regressions across PRs and CI. Do both.
- **Leaving a `*-dev` entry registered after the work is done.** Fine while the project is active, but when you ship, document for the user that they can delete the dev entry from their session list — it's not meant for production.

### 7. Quick safety checklist before committing a `.mcp.json`

- [ ] No literal tokens, keys, or passwords anywhere in the file.
- [ ] Every secret is `${VAR}` interpolation.
- [ ] `args` points at built artifacts, not source.
- [ ] Server name is suffixed (e.g. `-dev`) to avoid collision with prod entries.
- [ ] README links to this pattern doc instead of duplicating it.
- [ ] `.gitignore` includes any local-override file you create (e.g. `.mcp.local.json`).

---

## Worked example: this repository (`mcp-discord-readonly`)

The `.mcp.json` at the root looks like:

```json
{
  "mcpServers": {
    "discord-readonly-dev": {
      "command": "node",
      "args": ["build/index.js"],
      "env": {
        "DISCORD_TOKEN": "${DISCORD_TOKEN}",
        "DISCORD_ALLOWED_GUILDS": "${DISCORD_ALLOWED_GUILDS}",
        "DISCORD_ALLOWED_CHANNELS": "${DISCORD_ALLOWED_CHANNELS}"
      }
    }
  }
}
```

The dev loop, end to end:

```bash
export DISCORD_TOKEN="…"                            # from Discord Developer Portal
export DISCORD_ALLOWED_GUILDS="123456789012345678"  # a sandbox guild
npm run build
# in Claude Code, /mcp -> reconnect discord-readonly-dev
# then call discord_read_messages, discord_get_server_info, etc.
```

For a session that can't get real Discord credentials, the still-useful cases are: `tools/list` (verifies tool surface didn't regress), calls with missing tokens (verifies fail-closed path), calls with channel IDs that aren't allow-listed (verifies allow-list enforcement), and calls with malformed args (verifies Zod error shape).
