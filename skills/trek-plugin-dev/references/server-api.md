# Server API — `definePlugin` and the `ctx` object

The server entry is `server/index.js` at the plugin root: **built, plain
CommonJS** (`package.json` has `"type": "commonjs"`). It exports the object
returned by `definePlugin(...)`. The host loads it in an isolated child
process; everything reaches TREK through the `ctx` argument over RPC.

```js
const { definePlugin } = require('trek-plugin-sdk')   // injected at runtime — devDependency only!

module.exports = definePlugin({
  // Runs once on activation. NO user context: ctx.trips.* is refused here.
  async onLoad(ctx) {
    await ctx.db.migrate('001_init',
      'CREATE TABLE IF NOT EXISTS cache (k TEXT PRIMARY KEY, v TEXT)')
    ctx.log.info('loaded')
  },

  // Runs once on deactivation/stop — flush or release resources.
  async onUnload(ctx) { ctx.log.info('unloading') },

  // HTTP routes, mounted at /api/plugins/<id><path>.
  routes: [
    { method: 'GET', path: '/status', auth: true,
      async handler(req, ctx) {
        const rows = await ctx.db.query('SELECT COUNT(*) AS n FROM cache')
        return {
          status: 200,
          headers: { 'content-type': 'application/json' },
          body: JSON.stringify({ n: rows[0].n, user: req.user?.username }),
        }
      } },
  ],

  // Cron jobs — TREK owns the schedule. NO user context.
  jobs: [
    { id: 'refresh', schedule: '*/15 * * * *', async handler(ctx) { /* … */ } },
  ],
})
```

The routes and job ids on the **loaded definition are authoritative** (a
route's array index is its internal id); the `routes` block in the manifest is
never consumed.

## Type surface (from `trek-plugin-sdk`)

```ts
export const PLUGIN_API_VERSION = 1;

export interface PluginDefinition {
  onLoad?(ctx: PluginContext): Promise<void> | void;
  onUnload?(ctx: PluginContext): Promise<void> | void;
  routes?: PluginRoute[];
  jobs?: PluginJob[];
  hooks?: { photoProvider?: PhotoProvider; calendarSource?: CalendarSource }; // reserved — host does not call these yet
}

export interface PluginRoute {
  method: 'GET' | 'POST' | 'PUT' | 'DELETE' | 'PATCH';
  path: string;
  auth?: boolean;               // default true; false for OAuth callbacks/webhooks
  handler(req: PluginRequest, ctx: PluginContext): Promise<PluginResponse>;
}

export interface PluginRequest {
  method: string;
  path: string;
  query: Record<string, unknown>;
  body: unknown;
  user: { id: number; username: string; isAdmin: boolean } | null;
}
export interface PluginResponse {
  status: number;
  headers?: Record<string, string>;
  body?: unknown;
}

export interface PluginJob {
  id: string;
  schedule: string;             // cron expression; the host owns the schedule
  handler(ctx: PluginContext): Promise<void>;
}

export interface PluginContext {
  readonly id: string;                                   // your plugin id
  readonly config: Readonly<Record<string, unknown>>;    // resolved settings (secrets decrypted)
  db: {
    query<T = unknown>(sql: string, ...args: unknown[]): Promise<T[]>;
    exec(sql: string, ...args: unknown[]): Promise<{ changes: number }>;
    migrate(id: string, sql: string): Promise<{ applied: boolean }>;
  };
  trips: {
    getById(tripId: number, asUserId: number): Promise<unknown>;
    getPlaces(tripId: number, asUserId: number): Promise<unknown[]>;
    getReservations(tripId: number, asUserId: number): Promise<unknown[]>;
  };
  users: { getById(id: number): Promise<unknown> };
  ws: {
    broadcastToTrip(tripId: number, event: string, data: Record<string, unknown>): Promise<void>;
    broadcastToUser(userId: number, event: string, data: Record<string, unknown>): Promise<void>;
  };
  log: {
    info(msg: string, meta?: Record<string, unknown>): void;
    warn(msg: string, meta?: Record<string, unknown>): void;
    error(msg: string, meta?: Record<string, unknown>): void;
  };
}
```

## `ctx` semantics and required permissions

| Area | Behavior | Requires |
|---|---|---|
| `ctx.db` | Your **own** SQLite file (never `trek.db`). `migrate(id, sql)` runs a keyed, idempotent migration once per id — use for `CREATE TABLE`. `ATTACH`/`DETACH`/`VACUUM`/`PRAGMA` refused. | `db:own` |
| `ctx.trips` | Read-only; **route handlers only**. The host binds the acting user from the authenticated request and membership-checks every read. The `asUserId` parameter exists for source compatibility but is **ignored by the host** — you cannot impersonate. From `onLoad`/`jobs` (no user): `RESOURCE_FORBIDDEN`. | `db:read:trips` |
| `ctx.users.getById` | Public profile only: `id, username, display_name, avatar`. Never hashes/tokens/secrets. | `db:read:users` |
| `ctx.ws.broadcastToTrip` | Real-time event to a trip's members; event name forced to `plugin:<id>:<event>` (cannot forge core events). | `ws:broadcast:trip` |
| `ctx.ws.broadcastToUser` | Same, to one user's connections. | `ws:broadcast:user` |
| `ctx.config` | Frozen object of resolved settings; `secret: true` values arrive decrypted (server-side only). | — |
| `ctx.log` | Goes to the plugin's error log in the admin panel. | — |
| `ctx.id` | Your plugin id. | — |

**Error codes** (thrown across RPC): `PERMISSION_DENIED` (manifest didn't
grant the capability), `UNKNOWN_METHOD` (host doesn't expose the method),
`RESOURCE_FORBIDDEN` (membership check failed / no user context).

## Routes

- Mounted at **`/api/plugins/<id><path>`**. In the dev server they appear
  under `/api/<path>`.
- `auth: true` (default): `req.user` is the logged-in user; unauthenticated
  calls get 401. `auth: false` for OAuth callbacks / webhooks.
- The proxy forwards only `{ method, path, query, body, user }` — plugin code
  never sees raw headers or the session cookie.
- Return shape: `{ status, headers?, body? }`. Serialize JSON yourself and set
  `content-type` explicitly.

## Outbound HTTP

Use the global `fetch` (Node >= 18). Every request passes the runtime egress
guard: only hosts granted as `http:outbound:<host>` (exact or `*.suffix`
wildcard) are reachable; private/loopback/link-local/metadata addresses are
always refused (SSRF backstop). See the egress trap in
[manifest.md](manifest.md).

## What plugin code can NOT do

No filesystem writes, no reading TREK's files or env secrets, no child
processes, no worker threads, no native addons. Filesystem reads are scoped to
the plugin's own code (Node `--permission` model). A crash/hang/OOM kills only
the plugin's process; TREK restarts or disables it.

## Reference implementation

`plugin-sdk/examples/koffi/server/index.js` (TREK repo): a single
membership-checked `GET /state` route computing days-until-trip — note it does
**not** use `definePlugin` (it exports a plain object, which is equivalent
since `definePlugin` is an identity function for typing).
