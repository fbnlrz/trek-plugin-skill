# trek-plugin-skill

An **agent skill** that teaches AI coding agents (Claude Code and other
SKILL.md-compatible agents) how to build, test, and publish plugins for
[TREK](https://github.com/mauriceboe/TREK) — the self-hosted travel planner —
and its community registry
[TREK-Plugins](https://github.com/mauriceboe/TREK-Plugins).

## What the agent learns

- The plugin model: `integration` / `page` / `widget`, the isolated
  child-process runtime, and the sandboxed iframe UI.
- `trek-plugin.json`: every manifest field, the full permission catalog, and
  the `http:outbound:<host>` vs `egress[]` trap.
- Server code with `definePlugin`: routes, cron jobs, the `ctx` object
  (`db`, `trips`, `users`, `ws`, `config`, `log`) and its error codes.
- The client `postMessage` bridge (`trek:ready`, `trek:context`,
  `trek:invoke`, …) and the per-plugin CSP.
- Local development and testing: `trek-plugin dev`, `dev-fixtures.json`,
  `createMockHost`.
- The whole `trek-plugin` CLI (`create`, `dev`, `validate`, `pack`, `entry`,
  `release`, `preflight`, `submit`, `publish`, `keygen`/`sign`).
- Publishing: GitHub releases, the registry entry schema, **every CI gate**
  of the TREK-Plugins repo (entry + README quality gates), signing (TOFU),
  and the update flow.

## Layout

```
skills/trek-plugin-dev/
├── SKILL.md                    # entry point: workflow, rules, decision tables
└── references/
    ├── manifest.md             # trek-plugin.json + permissions + egress
    ├── server-api.md           # definePlugin, ctx, routes, jobs
    ├── client-bridge.md        # iframe sandbox + postMessage protocol
    ├── testing.md              # dev server + createMockHost
    ├── cli.md                  # all trek-plugin CLI commands
    └── publishing.md           # releases, registry entry, CI gates, signing
```

## Install

### Local — CLI, Desktop, or IDE

The interactive `/plugin` command works in the local Claude Code CLI, the
Desktop app, and the IDE extensions:

```
/plugin marketplace add fbnlrz/trek-plugin-skill
/plugin install trek-plugin-dev@trek-plugin-skill
```

### Claude Code on the web (claude.ai/code)

`/plugin` is **not available in web sessions** (it opens an interactive picker).
On the web you enable the skill through repo config instead — pick one:

**Option A — declare the marketplace in your repo** (recommended; auto-loads for
every web session on that repo). Commit `.claude/settings.json` to the **repo you
run web sessions on** (e.g. your `trek-plugin-<id>` repo):

```json
{
  "extraKnownMarketplaces": {
    "trek-plugin-skill": {
      "source": { "source": "github", "repo": "fbnlrz/trek-plugin-skill" }
    }
  },
  "enabledPlugins": ["trek-plugin-dev@trek-plugin-skill"]
}
```

The `enabledPlugins` entry is `<plugin>@<marketplace-key>`, so it must match the
key you used under `extraKnownMarketplaces`.

**Option B — vendor the skill into your repo** (simplest, no marketplace).
Copy the skill folder into the repo you work on; web sessions auto-load
`.claude/skills/**/SKILL.md`:

```bash
cp -r skills/trek-plugin-dev /path/to/your-repo/.claude/skills/
```

Web caveats: the session's network access must be **Trusted** or **Full** so it
can reach `github.com`; the marketplace must be on the repo's **default branch**
(it is — `main`); **user-scoped** plugins (`~/.claude/settings.json`) do *not*
carry into web sessions, so declare them in the **repo's** `.claude/settings.json`;
and start a **new** web session after committing config (resuming reuses cached
config). Docs:
[Claude Code on the web](https://code.claude.com/docs/en/claude-code-on-the-web) ·
[Discover plugins](https://code.claude.com/docs/en/discover-plugins).

### Plain skill (any Claude Code)

Or just copy the folder into a project or your user skills directory:

```bash
cp -r skills/trek-plugin-dev /path/to/project/.claude/skills/   # project
cp -r skills/trek-plugin-dev ~/.claude/skills/                  # user (local only)
```

However you install it, the skill triggers automatically when a task involves
TREK plugins, `trek-plugin-sdk`, `trek-plugin.json`, or the TREK-Plugins
registry — or invoke it explicitly with `/trek-plugin-dev`.

## Updating

This plugin is intentionally **unversioned** (its `plugin.json` has no `version`
field), so Claude Code resolves its version from the git commit — **every new
session installs the latest `main`**. You don't bump anything to get updates.

### On the web (claude.ai/code)

- **Just start a new cloud session.** A new session re-clones the repo and
  re-installs plugins from the marketplace at startup, picking up the latest
  commit. **Resuming** an existing session does *not* refresh — it keeps the
  plugins from that session's original start. There is no mid-session refresh
  (`/plugin` and `/reload-plugins` are interactive, so unavailable on the web).
- If you **vendored** the skill into `.claude/skills/` (Option B), update by
  replacing the files, committing to the default branch, and starting a new
  session — the fresh clone carries the new `SKILL.md`.
- **Want stability instead of latest?** Pin the marketplace source to a tag or
  branch in your `.claude/settings.json`, and bump the `ref` when you want the
  update:

  ```json
  {
    "extraKnownMarketplaces": {
      "trek-plugin-skill": {
        "source": { "source": "github", "repo": "fbnlrz/trek-plugin-skill", "ref": "v1.2.0" }
      }
    },
    "enabledPlugins": ["trek-plugin-dev@trek-plugin-skill"]
  }
  ```

### Local (CLI / Desktop / IDE)

```
/plugin marketplace update trek-plugin-skill    # refresh the marketplace
/plugin update trek-plugin-dev@trek-plugin-skill
```

> **Maintainer note:** because there is no `version` field, pushing to `main`
> ships to everyone on their next session. If you fork this and prefer pinned
> semver releases, add a `version` to `.claude-plugin/plugin.json` and **bump it
> on every release** — Claude Code caches a plugin whose version string didn't
> change, so an un-bumped version silently withholds updates.

## Sources

Built from the primary sources (July 2026): the
[TREK](https://github.com/mauriceboe/TREK) repo (`plugin-sdk/`, server plugin
runtime), the [TREK wiki](https://github.com/mauriceboe/TREK/wiki)
(Plugin-Development, Plugin-Permissions, Plugin-Publishing, Plugins), and the
[TREK-Plugins](https://github.com/mauriceboe/TREK-Plugins) registry (schemas,
CI scripts, koffi example). Community plugins are third-party software — see
the registry's security notes.

## Feedback & corrections

This skill is documentation verified against TREK's source — but TREK evolves.
When an agent using the skill hits a claim that contradicts the real TREK source
or a running instance (or a gap that cost time), it will hand you a **ready-made
feedback block, already filled in**. Just
[open an issue](https://github.com/fbnlrz/trek-plugin-skill/issues/new/choose),
pick **📋 Paste an agent-generated report**, paste, and submit — that's the whole
job. Manual **Skill discrepancy** and **Missing guidance** forms exist too.

One thing we ask you to keep honest: the block's **Evidence** line — read in the
source, seen on a real instance, seen in `trek-plugin dev`, seen in a custom
harness (no real CSP/sandbox), or merely inferred. An inference isn't a confirmed
discrepancy; several reported "bugs" have turned out to be test-method artifacts.

## License

MIT (this skill). TREK and TREK-Plugins are licensed by their own authors.
