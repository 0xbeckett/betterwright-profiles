# Named, lockable browser profiles in one BetterWright home

## The limitation

All BetterWright state lives under `$BETTERWRIGHT_HOME`, and a home has exactly
**one** persistent browser profile at `browser/profile`. Per
`docs/attach-mode.md`, only one process may own it at a time; a second
concurrent worker receives an isolated ephemeral profile from `browser/runtime`
rather than corrupting the persistent one.

That is the right safety behaviour, but it makes **concurrency and logged-in
state mutually exclusive**. A second concurrent run gets an ephemeral profile —
a signed-out browser. So any host that wants to run browser work in parallel
must serialize every run behind one global lock, even when the runs touch
completely unrelated sites and unrelated accounts.

## Why it matters

The framing is *concurrency vs. logged-in state*. Today you can have one or the
other, never both. A host running, say, a "social" worker that holds the
X/LinkedIn logins and a "review" worker that reads and screenshots pages cannot
run them at the same time without one of them being silently signed out.

## The feature

Named profiles inside one home. A caller picks a profile by name; each name is
an independent persistent profile with its own lock. Different names run
concurrently, each fully logged in. The same name still serializes with the
existing ephemeral fallback.

```js
new BetterWright({ profile: "social" }); // holds the X / LinkedIn logins
new BetterWright({ profile: "review" }); // reads and screenshots, concurrently
```

- **Library:** new `profile?: string` option on the `BetterWright`
  constructor, alongside `home`.
- **CLI:** `--profile <name>` on `run` / `repl` / `agent`.
- **MCP:** `BETTERWRIGHT_PROFILE` environment variable (matching how
  `BETTERWRIGHT_TIMEZONE` / `BETTERWRIGHT_LOCALE` are already read).

## Default behaviour is unchanged

**Omitting `profile` changes nothing.** The default profile directory stays
`browser/profile`, byte-for-byte, with the same lock at
`browser/profile.betterwright-lock`. There is **no migration, move, or copy** of
any existing profile — an install with cookies in `browser/profile/` keeps using
exactly that directory. This is covered by tests
(`profileDirFor keeps the default profile path byte-for-byte`,
`the default profile uses the historical lock location`).

## How it works

- `src/profile-name.ts` resolves a name to a directory: no name →
  `browser/profile` (unchanged); a name → `browser/profiles/<name>`.
- The profile **lock is already derived from the profile directory**
  (`profileLockDirFor(profileDir)` → `<profileDir>.betterwright-lock`), so
  scoping the directory scopes the lock for free. `acquireProfileLock` and its
  ephemeral-runtime fallback are untouched — same-profile contention behaves
  exactly as before.
- Only the profile directory and its lock are scoped. The **vault, `artifacts/`,
  the CloakBrowser binary cache, and `browser/runtime/` stay shared**, so a
  credential saved once is reachable from any profile.

### Scope decisions called out in the brief

- **Vault is not scoped.** `createLocalCredentialVault({ home })` still keys off
  the home only, so a credential saved from one profile is usable from every
  profile. Verified in
  `BetterWright scopes only the profile directory; vault, artifacts, runtime stay shared`.
- **Daemon socket stays per-home.** The session daemon holds one browser (hence
  one profile). Rather than derive a per-profile socket path — which risks the
  104/108-byte `sun_path` limit that `docs/architecture.md` documents, and which
  the brief explicitly wants avoided — the profile rides in the daemon's
  **config signature** (`normalizeDaemonConfig`). A daemon serves one profile at
  a time; a client that requests a different profile than the running daemon
  holds sees a signature mismatch and falls back to a private in-process browser
  **scoped to the profile it asked for**. So cross-profile concurrency always
  works; only daemon-side session persistence is limited to one profile per home
  at a time. This keeps the socket path length invariant and never runs a client
  on the wrong profile.
- **Name validation is a strict allowlist**, not a blocklist: letters, digits,
  `.`, `-`, `_`, must start with a letter or digit, max 64 chars. That rejects
  path separators, `..`, absolute paths, leading dots/dashes, whitespace, and
  NUL by construction; Windows reserved device names (`CON`, `NUL`, `COM1`…) are
  rejected on every platform for portability. An invalid name throws a
  `TypeError` at construction. A name can never escape `browser/profiles/`.
- **Case sensitivity is explicit:** names are used verbatim as a directory
  segment, so they are as case-sensitive as the filesystem — two profiles on a
  case-sensitive volume, one on a case-insensitive one. BetterWright does not
  fold case itself (that would rename directories out from under an install).

## No changes to security-sensitive behaviour

No changes to network policy, download policy, or credential handling. The guard
proxy, the network floor, redaction, and the vault API are untouched. The diff
adds one small pure module, threads one option through the client / CLI / MCP /
daemon-config, and updates docs and tests.

## Tests

- New `tests/node/profiles.test.mjs`: name validation (traversal, separators,
  absolute paths, reserved names, NUL, over-long, non-string), default path
  byte-for-byte, named-path layout, **concurrent different profiles** (both hold
  independent locks, neither falls back), **concurrent same profile** (second
  gets the ephemeral fallback), and the shared-vs-scoped split at the client
  layer.
- One additive line in `tests/types/package.test.ts` exercises the new
  `profile` field against the published declarations.
- The existing suite passes unmodified (`npm run release:check`: lint,
  typecheck, `check:build`, unit tests, `test:types`, `check:package`).

## Docs

- `docs/architecture.md` — State-on-disk tree now shows `browser/profiles/`,
  plus a "Named profiles" subsection covering the lock model, the shared-vs-
  scoped split, and the deliberate per-home daemon-socket decision.
- `docs/attach-mode.md` — a "Named profiles for concurrency" subsection.
- `README.md` and `CHANGELOG.md` updated.
