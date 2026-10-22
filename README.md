# git.pp-desk — a read-only window onto a git.pp project

**The Desk shows you what a git.pp project's record currently says — the derived state, the conflicts and who won them, and how stale each witness is — and it can prove, on demand, that everything it shows can be destroyed and rebuilt identically from the record alone.**

**Live:** https://gitpp-desk.superinstance.dev (watching `poc-quilt-1`)

**For:** anyone running or evaluating a git.pp project who wants the state without folding the record themselves. **Not for:** writing anything. The Desk has no write path, no admin key, and no push route — that absence is deliberate, and §2 explains why it's the point.

## 1. Thirty seconds

```bash
curl -s "https://gitpp-desk.superinstance.dev/api/desk?project=poc-quilt-1"
```

Real output, fetched 2026-10-21, trimmed to the load-bearing fields (the full response also carries the 8 records, the application order, and the refresh manifest):

```json
{
  "project": "poc-quilt-1", "ready": true,
  "state": { "config.timeout": 45, "env.oracle_disk_pct": 93,
             "env.oracle_load1_x100": 17, "feature.cache": "on",
             "feature.search": "on", "status": "open" },
  "conflicts": [ { "key": "config.timeout", "winner": "override-1",
                   "losers": ["task-a", "task-b", "task-d"] } ],
  "witnesses": [ { "id": "witness-1", "recordsBehind": 6 },
                 { "id": "witness-2", "recordsBehind": 3 } ],
  "digest": "010e942c049515c29650c7901b8bd34fdf25d16a076e2ba781d1535a3cb95823",
  "root":   "2b79c3d2ea178105859c00a06f441a26bc2c26e6d321e9749410fba7ef038048",
  "refreshStats": { "ticks": 9999, "skippedTicks": 9997, "lastApiCalls": 1 }
}
```

What just happened: you read a *projection*. Somewhere else, eight git.pp records (tasks, witnesses, one captain's override) were folded into a single state — `config.timeout` is 45 because `override-1` outranked three rival tasks, whose losing values (30, 60, 99) are preserved, not overwritten. The Desk holds a copy of that derivation so you don't have to redo it; the two hashes at the bottom are how it proves the copy is honest (§3). The page at `/` is the same projection as a story: watch the fold replay, then press the burn button.

## 2. Five minutes — the model

Four concepts, in the order they compose:

- **The record** — the append-only set of git.pp records a project consists of. It lives in the git.pp Worker (`https://gitpp.superinstance.dev`), not here. It is the only truth in the system.
- **The fold** — the deterministic rule (spec: git.pp-ng `DESIGN.md` §2) that derives one state from the record set: class precedence (override > task > witness), then exact maximality, then cleanest ancestry chain, then smallest id. Arrival order is never an input.
- **The projection** — what the Desk stores: the folded state, the conflicts, per-witness `recordsBehind`, the five plain-language digest lines, and two pins (next concept). Held in a Cloudflare Durable Object (Agents SDK), one instance per project, refreshed on a 60-second cadence.
- **The pins** — a sha256 **digest** over the canonical state JSON (pins *what was derived*) and the checkpoint-v2 **closure root** over the record set (pins *what it was derived from*). §3 is about why you need both.

What the Desk does with them:

- **`refresh()`** — each tick first reads the source Worker's `/api/repos` manifest (a per-repo `updatedAt` sheet). If none of the project's repos moved since the last full read, the tick ends: 1 call, no re-derivation. Otherwise it fetches `/api/state` + `/api/records` and re-derives. Both source endpoints are public and credential-free.
- **`why(key)`** — replays the fold rule over the held records for one key and returns the winner, the preserved losers, the ancestry counts, and a plain-language explanation. Live: `curl -s "https://gitpp-desk.superinstance.dev/api/desk?project=poc-quilt-1&action=why&key=config.timeout"` → *"override-1 wins config.timeout=45 on class precedence… Its stated reason: 'captain: 45 is the safe value'."*
- **`draftOverride(key, id, reason)`** — drafts a captain's ruling as a ready-to-push override record (parents = the current frontier, effect = the key's standing value) and hands the record back to you. **It never submits it.** There is no admin key in this repo or Worker, no storage binding to the record, no push path — the Desk returns JSON; the captain's own tooling (`pp.py`, in git.pp-ng) decides and pushes. The draft response says so in its own `note` field.
- **`burn()` / `rebuild()` / `destroyInstance()`** — escalating forms of the same drill: wipe the stored projection (or destroy the entire Durable Object — every table, the alarm, all storage) and let it rebuild from the record alone. §3 gives the measured results.
- **The page at `/`** — the chair: claim, a browser-side replay of the fold (a client port of the same rule, digested on-page and compared against the Desk's live digest), the burn drill centre-stage, a shuffle-and-re-fold control, then the depth — conflicts with a *why?* per key, the witness table, the record in application order, the draft form. Live over WebSocket; every `setState` broadcasts.

The same surface over HTTP, for scripts: `/api/desk?project=<name>&action=<status|refresh|why|draft|burn|rebuild|destroy>` (`key`, `id`, `reason` as further query params; unknown action → 400; a failed upstream read → 502, and the previous projection stays served rather than a half-derived one).

### Honest limits, early

- **Read-only, structurally.** Not a setting: there is no credential or binding in this Worker with which a write could be made. If you need to change state, you write a record through the runner, and the Desk notices within a tick.
- **It polls; it does not subscribe.** Refresh is a 60 s cadence plus the manifest skip. Consuming push events instead is designed, not built.
- **The cadence lives in `desk.json`, in this repo** — a spike simplification, stated plainly. The full design declares cadences as records in the record itself (like every other default in the cascade); the shape here is the same — an alarm the Desk owns, intent written somewhere it doesn't — but the declaration has not moved yet.
- **Re-reads are project-grained.** `/api/records` has no per-repo filter, so a one-repo change re-reads the project's records. The skip is what saves the quiet minutes, not the busy ones.
- **Warm-state wart, kept visible:** Durable Object code deploys do not migrate stored state; an instance warmed under v1 once threw on v2's new fields until a burn reset the shape. All v2 fields are now read through defaults. The playtest caught it; the note stays so the next port expects it.

## 3. The deep layer

### The projection is disposable — and that's a test, not a slogan

Most dashboards quietly become a second source of truth: their stored state drifts from the system's, and nobody can say which is right. The Desk is built on the opposite bet — *an always-on projection is beautiful exactly insofar as it is disposable*. So the deciding test is a drill, run live:

`destroyInstance()` destroys the whole Durable Object (`this.destroy()`). A fresh instance answers under the same name, empty. Its `onStart` re-registers the 60 s cadence — the one dependency the heal leans on, now explicit, because `destroy()` takes the schedules with everything else — and with no writes from anyone, only its own ticks, it rebuilds the projection from the record alone. Measured on the disposable project `demo-race-1` (never on `poc-quilt-1`), 2026-10-14: the rebuilt projection landed on the pre-destroy digest `eaf5bce8…874777` and root `0be8be5a…46afe72` — **identical**. Nothing strands, because there is nothing here to strand: anything another agent must see, or the captain has decided, belongs in the record as a record. On any disagreement between this page and the record, the record wins and the Desk rebuilds. That paragraph is the design; `src/index.ts` is mostly it, enforced.

### Why a state digest is not enough — the closure root

The digest pins the derived state. It cannot pin the *derivation*. Two different record sets can fold to the same state today — a closure quietly swapped for another that agrees on every standing key — and diverge tomorrow, when a new record's ancestry reaches into the swapped part. The closure root closes that hole: checkpoint v2's construction (spec: git.pp-ng `docs/CHECKPOINT-V2.md`), ported in `src/digest.ts` — a whole-record leaf per record (id, parents-as-set, kind, effects in the float64 domain, reason), hashed over the frontier records' ancestor-closure union. Any held record added, removed, or **re-versioned** moves it, including changes the fold itself ignores.

The port is not trusted on its own say-so. It reproduces the conformance-pinned root at `poc-quilt-1@task-c` — `60effe23970d…fc91d` over a 5-record closure — from the live record set; the Python referee computes the same whole-set root the Desk serves (`2b79c3d2…f038048`); and the playtest carries a third, from-the-doc implementation that must agree before anything else is believed (§3, verification).

### The refusal is the architecture

`draftOverride` returning a record instead of applying it looks like a missing feature. It is the security model. The Desk is reachable by anyone with the URL; it holds no credential, so there is nothing to steal from it, no binding through which a forged record could be pushed, and no code path to audit for write-safety — the class of bug that the git.pp Worker itself had to have removed (an unauthenticated write route, found and deleted in its security hardening) cannot exist here, because the verb does not. Drafts that matter become records through the captain's tooling, under the captain's key, in the open.

### Measured, not described

Refresh economics on `poc-scale-1` (62 records), from the Desk's own `apiCalls` instrumentation, 2026-10-14:

| tick | v1 | v2 |
|---|---|---|
| quiet | 2 calls, 2 server derivations, every minute | **1 call** (manifest), 0 derivations, ~0.3 s |
| changed | 2 calls | 3 calls (manifest + the same two reads) |

The live counters at the top of this README agree with the shape: 9,997 of 9,999 ticks on `poc-quilt-1` were skips. And the cadence is real, not decorative — a cold `poc-scale-1` Desk (`ready:false`) populated itself with no further request ~80 s later, landing on digest `f1e78b16…4380d` from its own scheduled tick.

### Verify it yourself

```
node test/playtest.mjs   # live, against the deployed Desk
```

The playtest's first check computes the state digest **in the test script, by a second implementation of the canonical form, over bytes fetched directly from the git.pp Worker — never through the Desk** — and only then compares. It also checks `why()` against the standing winner, the draft's parents against the frontier, burn/rebuild identity (digest *and* root), the page's structure, and the manifest-skip equality (a skipped tick's digest+root equal a forced full re-read's). Measured totals, 2026-10-14: **16/16 on `poc-quilt-1`, 16/16 on `poc-scale-1`, 19/19 on `demo-race-1`** with the destroy drill enabled (`DESTROY_DRILL=1`). A second implementation proving it means the same thing is the family's whole verification contract — the fold has three (TypeScript, the Python referee, a BEAM port) held to one conformance corpus in git.pp-ng.

## Run your own

```
npm install
npm run dev      # local
npm run deploy   # wrangler deploy (needs a Cloudflare account with
                 # Workers + Durable Objects; DESK binding and the
                 # SQLite migration are declared in wrangler.toml)
```

Point `desk.json`'s `source` at any git.pp-compatible record worker; the Desk reads only its two public endpoints.

## Siblings — one tool, one repo

- **[git.pp-ng](https://github.com/patchwork-quilt/git.pp-ng)** — the spec, the fold engine, the runner (`pp.py`), and the conformance corpus this Desk's root port is pinned against. Start here for the system itself.
- **[git.pp-grid](https://github.com/patchwork-quilt/git.pp-grid)** — the shared-state surface as a spreadsheet-plain page: custody rotas, freezes, who's on call. The Desk is for one project's truth; the Grid is for a team's week.
- **[gitpp-action](https://github.com/patchwork-quilt/gitpp-action)** — the fold in your own CI: derived state as a job summary / advisory signal on the repos you already have, no installation.
- **[gitpp-app](https://github.com/patchwork-quilt/gitpp-app)** — the GitHub App: the same derivation posted as a real Check on pull requests, installed per-repo.
- **[gitpp-beam](https://github.com/patchwork-quilt/gitpp-beam)** — the fold's third implementation, in Elixir/BEAM, from the spec alone. Read it to see the semantics survive a hostile port.
- **[outbound-scanner](https://github.com/patchwork-quilt/outbound-scanner)** — unrelated job, same shelf: a pre-send fence that stops agent harnesses narrating session secrets into chat.

## License

MIT © 2026 SuperInstance.
