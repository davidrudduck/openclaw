# Fork Changes: davidrudduck/openclaw

Tracking all changes in our fork that are not yet in upstream `openclaw/openclaw:main`.
Use this to identify when upstream absorbs our fixes so we can drop the fork delta.

**Last synced with upstream:** 2026-02-15 (upstream commit `cc15b8c6a`)

---

## Open Upstream PRs

| PR                                                        | Title                                                           | Branch                                 | Status | Importance                                                                                                        |
| --------------------------------------------------------- | --------------------------------------------------------------- | -------------------------------------- | ------ | ----------------------------------------------------------------------------------------------------------------- |
| [#15824](https://github.com/openclaw/openclaw/pull/15824) | normalize hook addresses to canonical `provider:kind:id` format | `fix/channel-id-hook-normalization-v2` | OPEN   | HIGH — enables consistent keying for all downstream hook consumers (mem0, universal-message-logger, brain-worker) |
| [#15825](https://github.com/openclaw/openclaw/pull/15825) | wire message_sent hook into reply dispatcher for all channels   | `fix/wire-message-sent-hook-pr-v2`     | OPEN   | HIGH — required for universal-message-logger to capture outbound messages; currently only inbound is logged       |
| [#14320](https://github.com/openclaw/openclaw/pull/14320) | context decay — graduated context window management             | `feat/context-decay`                   | OPEN   | MEDIUM — reduces token cost for long conversations; experimental, all knobs disabled by default                   |
| [#13183](https://github.com/openclaw/openclaw/pull/13183) | use execFileSync to prevent shell injection                     | `fix/execsync-to-execfilesync`         | OPEN   | HIGH — security: prevents shell injection in daemon startup                                                       |
| [#12172](https://github.com/openclaw/openclaw/pull/12172) | harden resolveUserPath and compact against undefined            | `fix/trim-bug-remaining-guards`        | OPEN   | MEDIUM — prevents crash on undefined workspace paths                                                              |
| [#11866](https://github.com/openclaw/openclaw/pull/11866) | guard .trim() on undefined in subagent spawn                    | `fix/subagent-trim-crash`              | OPEN   | MEDIUM — prevents crash during subagent system prompt building                                                    |

### Merged PRs

| PR                                                        | Title                                            | Merged |
| --------------------------------------------------------- | ------------------------------------------------ | ------ |
| [#13185](https://github.com/openclaw/openclaw/pull/13185) | sanitize error responses to prevent info leakage | Yes    |
| [#13184](https://github.com/openclaw/openclaw/pull/13184) | default standalone servers to loopback bind      | Yes    |

### Closed / Superseded PRs

| PR                                                         | Superseded by | Reason                                                                                      |
| ---------------------------------------------------------- | ------------- | ------------------------------------------------------------------------------------------- |
| [#12573](https://github.com/openclaw/openclaw/pull/12573)  | #15824        | Replaced by v2 with `provider:kind:id` naming aligned to upstream vocabulary                |
| [#11867](https://github.com/openclaw/openclaw/pull/11867)  | #15825        | Replaced by v2 — clean rebase, upstream absorbed markComplete/registerDispatcher removals   |
| [#16178](https://github.com/openclaw/openclaw/pull/16178)  | —             | Closed; empty guild channels fix — may resubmit if bug persists                             |
| [#12251](https://github.com/openclaw/openclaw/pull/12251)  | —             | Closed without merge; timing-safe comparison — absorbed into fork security audit quick wins |
| [#12371](https://github.com/openclaw/openclaw/pull/12371)  | #13185        | Branch deleted during fork maintenance                                                      |
| [#12370](https://github.com/openclaw/openclaw/pull/12370)  | #13184        | Branch deleted during fork maintenance                                                      |
| [#12253](https://github.com/openclaw/openclaw/pull/12253)  | #13183        | Branch deleted during fork maintenance                                                      |
| [#12207](https://github.com/openclaw/openclaw/pull/12207)  | #12251        | Duplicate; same branch, earlier version                                                     |
| [#11823](https://github.com/openclaw/openclaw/pull/11823)  | #11867        | Replaced by centralised approach                                                            |
| [Fork #8](https://github.com/davidrudduck/openclaw/pull/8) | #14320        | Closed — upstream PR is canonical                                                           |
| [Fork #9](https://github.com/davidrudduck/openclaw/pull/9) | #14320        | Closed — upstream PR is canonical                                                           |

---

## Change Groups

### 1. Hook Address Canonicalization (Feature) — HIGH PRIORITY

**PR:** [#15824](https://github.com/openclaw/openclaw/pull/15824)
**Branch:** `fix/channel-id-hook-normalization-v2` (1 squashed commit, rebased on upstream/main)
**Commit:** `209d51d0f`

Normalizes all hook address fields (`from`, `to`, `originatingTo`, `conversationId`) to canonical `provider:kind:id` format (e.g. `discord:channel:123`, `telegram:user:999`). Extends upstream's existing `${provider}:${kind}:${finalId}` pattern from `resolveGroupSessionKey`.

**Why this matters:**

- **universal-message-logger** writes `channel_id` to postgres — without canonicalization, the same channel appears as `channel:123`, `discord:channel:123`, or bare `123` depending on provider/chatType
- **mem0-extension** keys memories by `user_id` (identity-resolved) — canonical addresses enable future cross-channel memory dedup
- **conversation-brain-worker** currently hedges with `AND (channel_id = $2 OR channel_id = 'channel:' || $2)` — canonical format eliminates this hack
- **Phase 1 & 2 SQL migrations** (`/data/monday/sql/`) already normalize existing data TO this format — this PR ensures new data arrives canonical at write time

**Files:**

- `src/auto-reply/reply/dispatch-from-config.ts` — `toCanonicalAddress()`, `parseRoutingAddress()`, `chatTypeToKind()`
- `src/auto-reply/reply/dispatch-from-config.test.ts` — 29 new tests

---

### 2. message_sent Hook (Feature) — HIGH PRIORITY

**PR:** [#15825](https://github.com/openclaw/openclaw/pull/15825)
**Branch:** `fix/wire-message-sent-hook-pr-v2` (1 squashed commit, rebased on upstream/main)
**Commit:** `eb2ceadfe`

Centralises `message_sent` hook firing in the reply dispatcher. Hooks fire after each successful payload delivery with channel/account/conversation context.

**Why this matters:**

- **universal-message-logger** currently only logs INBOUND messages (via `message:received` event bus) — this hook enables logging OUTBOUND messages too, completing the conversation record
- Downstream consumers get consistent `channelId`/`accountId`/`conversationId` context for routing and analytics
- Fire-and-forget pattern ensures hook errors never break message delivery

**Files:**

- `src/auto-reply/reply/reply-dispatcher.ts` — `hookContext` option, `message_sent` firing with `hasHooks` guard
- `src/auto-reply/reply/reply-dispatcher.test.ts` — 5 new tests (NEW FILE)
- `src/infra/outbound/deliver.ts` — `conversationId: to` added to hook context
- `src/infra/outbound/deliver.test.ts` — 2 new tests
- `src/agents/system-prompt.ts` — guard `file.path ?? "(unnamed)"` (minor defensive fix)

---

### 3. .trim() Crash Guards (Bug Fix) — MEDIUM PRIORITY

**PRs:** [#11866](https://github.com/openclaw/openclaw/pull/11866), [#12172](https://github.com/openclaw/openclaw/pull/12172)

Guards against `undefined.trim()` crashes in subagent system prompt building and workspace resolution.

**Why this matters:**

- Production crashes when subagents spawn with undefined workspace paths
- Root cause: `resolveUserPath(input)` calls `input.trim()` without null guard

**Files:**

- `src/agents/system-prompt.ts` — `(file.path ?? "").trim()`
- `src/agents/subagent-announce.ts`, `src/agents/subagent-registry.ts`
- `src/agents/pi-embedded-runner/compact.ts` — uses `resolveRunWorkspaceDir`
- `src/agents/pi-embedded-runner/run/attempt.ts`
- `src/utils.ts` — `resolveUserPath` input guard
- `src/utils.test.ts`

**Upstream status:** Not yet merged. Check if upstream adds null guards to these paths.

---

### 4. Security Hardening (Bug Fixes) — HIGH PRIORITY

**PRs:** [#13183](https://github.com/openclaw/openclaw/pull/13183) (open), [#13185](https://github.com/openclaw/openclaw/pull/13185) (merged), [#13184](https://github.com/openclaw/openclaw/pull/13184) (merged)

| Fix                | File                         | Description                             | Status           |
| ------------------ | ---------------------------- | --------------------------------------- | ---------------- |
| Shell injection    | `src/daemon/program-args.ts` | `execFileSync` instead of `execSync`    | PR #13183 OPEN   |
| Error sanitization | `src/gateway/*.ts` + others  | Prevent info leakage in error responses | PR #13185 MERGED |
| Loopback bind      | `src/gateway/server-http.ts` | Default standalone servers to 127.0.0.1 | PR #13184 MERGED |

---

### 5. Security Audit Quick Wins (Bug Fixes) — HIGH PRIORITY

**Branch:** `fix/security-audit-quick-wins` (fork-only, not yet submitted upstream)

Five low-effort, high-impact hardening fixes identified during a full security audit.

| Fix | File                                    | Description                                                                                 |
| --- | --------------------------------------- | ------------------------------------------------------------------------------------------- |
| M1  | `src/config/includes.ts`                | Block `__proto__`, `prototype`, `constructor` in `deepMerge` to prevent prototype pollution |
| M2  | `src/security/secret-equal.ts`          | SHA-256 hash before `timingSafeEqual` to eliminate length-leak timing side-channel          |
| M4  | `src/markdown/frontmatter.ts`           | Explicit `schema: "core"` on YAML.parse to prevent YAML 1.1 type coercion                   |
| H3  | `src/gateway/hooks-mapping.ts`          | Block `__proto__`, `prototype`, `constructor` in hook template `getByPath` traversal        |
| L3  | `src/discord/monitor/exec-approvals.ts` | Escape backticks in exec-approval command previews to prevent Discord markdown injection    |

**Upstream status:** Not yet submitted upstream. Subsumes the closed #12251 (timing-safe comparison).

---

### 6. Context Decay — Graduated Context Window Management (Feature) — MEDIUM PRIORITY

**PR:** [#14320](https://github.com/openclaw/openclaw/pull/14320)
**Branch:** `feat/context-decay`

Experimental proactive context window management. Graduated decay tiers: keep -> swap-to-file -> LLM-summarize -> group-summarize -> strip. All knobs disabled by default.

**Why this matters:**

- Reduces token cost for long DM conversations (our primary use case)
- File swap tier is zero-LLM-cost and lossless — good for tool results that bloat context
- 106 tests across 7 files

**Config knobs:** `stripThinkingAfterTurns`, `summarizeToolResultsAfterTurns`, `stripToolResultsAfterTurns`, `maxContextMessages`, `summarizeWindowAfterTurns`, `summarizeWindowSize`, `swapToolResultsAfterTurns`, `swapMinChars`

**Upstream status:** Not yet merged. Track via PR #14320.

---

### 7. Empty Guild Channels Allowlist Bug (Bug Fix) — LOW PRIORITY

**PR:** [#16178](https://github.com/openclaw/openclaw/pull/16178) (CLOSED)
**Branch:** `fix/empty-guild-channels-allowlist-clean`

Fixes silent message drop for Discord guilds without explicit channel configs. `{ ...undefined, ...undefined }` produces `{}`, which is incorrectly treated as "channels configured but none match".

**File:** `src/discord/monitor/allow-list.ts` — add `Object.keys(channels).length === 0` guard

**Upstream status:** Closed. Bug introduced by upstream `c7ea47e88`. May resubmit if issue persists after upstream changes.

---

### 8. Housekeeping

| Item              | Description                             |
| ----------------- | --------------------------------------- |
| `.gitignore`      | Added `.omc/` (fork-specific tooling)   |
| `CLAUDE.md`       | Fork-specific instructions (gitignored) |
| `FORK-CHANGES.md` | This file (gitignored)                  |

---

## Dependency Graph

```
#15824 (canonical addresses) ← #15825 (message_sent hook) uses canonical conversationId
                              ← universal-message-logger benefits from both
                              ← brain-worker hedge removal depends on #15824 + Phase 2 migration

#14320 (context decay)       — independent, no dependencies on other PRs

#13183 (execFileSync)        — independent security fix
#12172 + #11866 (trim guards) — independent crash fixes
```

## Quick Check: Is Our Change Still Needed?

Run after each upstream sync:

```bash
# 1. List all our non-merge commits not in upstream
git log --oneline --no-merges upstream/main...HEAD --right-only

# 2. Check canonical address normalization
grep -rn 'toCanonicalAddress\|parseRoutingAddress' src/auto-reply/reply/dispatch-from-config.ts

# 3. Check message_sent hook
grep -rn 'message_sent' src/auto-reply/reply/reply-dispatcher.ts src/infra/outbound/deliver.ts

# 4. Check security fixes
grep -n 'execFileSync' src/daemon/program-args.ts

# 5. Check open PR status
gh pr list --repo openclaw/openclaw --author davidrudduck --state open
```

## Total Fork Delta

- **6 open upstream PRs** + 2 merged + fork-only security audit quick wins
- **8 change groups**: hook canonicalization, message_sent hook, .trim() guards, security hardening, security audit quick wins, context decay, empty guild channels, housekeeping
- Priority: 4 HIGH, 2 MEDIUM, 1 LOW, 1 housekeeping
