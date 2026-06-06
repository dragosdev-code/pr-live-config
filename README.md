<div align="center">

  <h1>pr-live-config</h1>

  <p><strong>Live parser patterns for <a href="https://github.com/dragosdev-code/pullwatch">Pullwatch</a>.</strong></p>

  <p>
    <a href="https://github.com/dragosdev-code/pullwatch">Pullwatch repository</a>
    ·
    <a href="https://dragosdev-code.github.io/pullwatch/">Documentation</a>
    ·
    <a href="https://dragosdev-code.github.io/pullwatch/architecture/remote-configuration/">Remote configuration guide</a>
  </p>

</div>

## What this is

[Pullwatch](https://github.com/dragosdev-code/pullwatch) is a Chrome extension that keeps your GitHub pull request inbox in the toolbar. It reads the same signed-in HTML pages your browser would show on github.com, then parses that markup into structured PR rows.

GitHub changes its DOM more often than a Chrome Web Store review cycle allows. This repository holds the regex and selector patterns that power that parser. Pullwatch downloads `patterns.json` from here at runtime, so a broken selector can be patched with a commit instead of waiting for a new extension release.

The extension always ships a bundled copy of these patterns as a fallback. This repo is the live channel on top of that floor: validated, versioned, and safe to roll forward, never backward.

## How Pullwatch uses it

Production extensions fetch this URL (defined as `REMOTE_PATTERNS_URL` in Pullwatch):

[https://raw.githubusercontent.com/dragosdev-code/pr-live-config/main/patterns.json](https://raw.githubusercontent.com/dragosdev-code/pr-live-config/main/patterns.json)

**When fetches happen**

| Trigger | Behavior |
| ------- | -------- |
| Service worker startup | Fetches immediately after loading cached or bundled patterns (no TTL gate). |
| GitHub PR list refresh | `GitHubService` calls `refreshIfStale()` at the start of each list fetch. If the last remote attempt was within six hours (`PATTERN_REFRESH_TTL_MS`), the call returns without downloading again. |

PR lists refresh about every three minutes by default, but the remote pattern file is polled at most every six hours during normal operation, plus once on each service worker wake.

**What happens to the response**

The update path is deliberately cautious:

1. **Schema validation** (Valibot, same rules as [`pattern-registry-schema.ts`](https://github.com/dragosdev-code/pullwatch/blob/main/extension/common/pattern-registry-schema.ts))
2. **Version gate**: remote `version` must be strictly greater than the version already loaded
3. **Upgrade ceiling**: remote `version` cannot jump more than 1000 ahead of the local version (`REMOTE_PATTERNS_MAX_VERSION_DELTA`)
4. **`minExtensionVersion` gate**: skipped when the running extension build is too old
5. **Compilation guard**: every regex is compiled inside `safeCompile`; a single bad pattern rejects the whole update

If any step fails, the previous patterns stay in place. Users are never left worse off than before the fetch.

For the full pipeline (cache, bundled defaults, storage envelope, and edge cases), see [Remote Configuration](https://dragosdev-code.github.io/pullwatch/architecture/remote-configuration/) in the Pullwatch docs.

## Repository layout

| File / branch | URL | Role |
| ------------- | --- | ---- |
| `patterns.json` on **`main`** | [raw `main` file](https://raw.githubusercontent.com/dragosdev-code/pr-live-config/main/patterns.json) | Production. Every installed Pullwatch extension reads this. |
| `patterns.json` on **`staging`** | [raw `staging` file](https://raw.githubusercontent.com/dragosdev-code/pr-live-config/staging/patterns.json) | Pre-production. Smoke tests run against this before promotion. |
| [`patterns.json`](patterns.json) | (local) | The config file: `version`, `minExtensionVersion`, optional `updatedAt`, and the `patterns` object (selectors and regexes for page recognition, PR rows, links, authors, timestamps, `newExperience`, and related fields). |

Pullwatch never fetches `staging` in production. Until you merge into `main`, live users keep the previous production config.

## Making a change

This repo is maintained alongside Pullwatch. The intended workflow matches [canary/DOM_CHANGE_RUNBOOK.md § Step 7](https://github.com/dragosdev-code/pullwatch/blob/main/canary/DOM_CHANGE_RUNBOOK.md#step-7--deploy-to-patternsjson-hot-fix-for-live-users):

1. **Check out `staging` and edit `patterns.json`.**
   - Bump the integer `version` by 1 for every change. Without a higher version, the extension ignores the update.
   - Set `updatedAt` to the current ISO timestamp (recommended; the field is optional in the schema).
   - Adjust `minExtensionVersion` only when the new patterns require a newer extension build.

2. **Validate offline from the Pullwatch repo** (optional but fast, no network):
   - `npm test` runs `pattern-registry-schema.test.ts`, which mirrors runtime `validateRemoteConfig()`.

3. **Commit and push to `staging`.** The smoke tests fetch the hosted file from GitHub, not your working tree.

4. **Run the remote schema smoke test against staging** (from the Pullwatch repo):
   - `npm run test:remote-patterns:staging` fetches the staging URL, validates the schema, compiles every regex (Acts 1 to 3), and runs **Act 4**: hosted `patterns` must match bundled [`default-patterns.ts`](https://github.com/dragosdev-code/pullwatch/blob/main/extension/common/default-patterns.ts) in the Pullwatch checkout you are testing from. Keep that checkout current if Act 4 should reflect your intended baseline.
   - Or trigger the [**Remote Patterns Schema Smoke**](https://github.com/dragosdev-code/pullwatch/actions/workflows/remote-patterns-smoke.yml) workflow with **Use staging URL** checked.

5. **Merge `staging` into `main`.** The raw `main` file is live as soon as the push lands.

6. **Confirm production** (from the Pullwatch repo):
   - `npm run test:remote-patterns` validates the `main` URL (Acts 1 to 3 only; production may run ahead of the last store build without matching bundled defaults).
   - `npm run test:remote-patterns:production:parity` is the opt-in variant that also runs Act 4 against `main`, useful before an extension release when you want `main` and bundled defaults in sync.

7. **Run the canary** (`npm run canary:test` or the hourly [**Parser Canary Tests**](https://github.com/dragosdev-code/pullwatch/actions/workflows/canary-parser-test.yml) workflow) after a pattern change. Smoke tests prove the JSON is structurally sound; the canary proves the regexes still match live GitHub HTML.

**Remote-only hotfix vs extension release**

| What changed | Pullwatch repo | This repo |
| ------------ | -------------- | --------- |
| Regex fix shipped only via remote (no store release) | No `BUNDLED_PATTERNS_REGISTRY_VERSION` bump | `patterns.json` `version` +1 on `main` |
| Regex fix in bundled `default-patterns.ts` (new extension build) | `BUNDLED_PATTERNS_REGISTRY_VERSION` +1 | Same `version` on `main` (keep in sync) |

See the [When to bump which version](https://dragosdev-code.github.io/pullwatch/architecture/remote-configuration/#when-to-bump-which-version) table for examples. After a DOM fix, the runbook also recommends committing the matching `default-patterns.ts` change in Pullwatch so the next store build and Act 4 parity stay aligned.

Scheduled CI in Pullwatch validates production `patterns.json` every six hours (same interval as `PATTERN_REFRESH_TTL_MS`, but independent of it).

## Config fields (short reference)

| Field | Required | Meaning |
| ----- | -------- | ------- |
| `version` | yes | Monotonic config revision (integer ≥ 1). Must increase for any pattern change to apply. |
| `minExtensionVersion` | yes | Minimum Pullwatch semver that may use this config (`isVersionAtLeast` against `chrome.runtime.getManifest().version`). Use when new patterns depend on newer extension code; otherwise a low bound such as `1.0.0` is fine. |
| `updatedAt` | no | ISO timestamp for humans and audit. |
| `patterns` | yes | The full pattern registry (same shape as [`default-patterns.ts`](https://github.com/dragosdev-code/pullwatch/blob/main/extension/common/default-patterns.ts)). |

Schema definition and runtime validation: [`extension/common/pattern-registry-schema.ts`](https://github.com/dragosdev-code/pullwatch/blob/main/extension/common/pattern-registry-schema.ts).

Bundled floor version (persisted on first install when storage is empty): [`BUNDLED_PATTERNS_REGISTRY_VERSION`](https://github.com/dragosdev-code/pullwatch/blob/main/extension/common/constants.ts) in Pullwatch.

## Further reading

These pages in the [Pullwatch documentation](https://dragosdev-code.github.io/pullwatch/) go deeper than this README:

- [Remote Configuration](https://dragosdev-code.github.io/pullwatch/architecture/remote-configuration/): safety rails, cadence, storage, and edge cases
- [Parser Waterfall](https://dragosdev-code.github.io/pullwatch/architecture/parser-waterfall/): how compiled patterns are consumed stage by stage
- [Canary Monitor](https://dragosdev-code.github.io/pullwatch/architecture/canary-monitor/): how DOM drift is detected before users report empty lists
- [Architecture Overview](https://dragosdev-code.github.io/pullwatch/architecture/overview/): where remote config fits in the wider extension design
- [DOM change runbook](https://github.com/dragosdev-code/pullwatch/blob/main/canary/DOM_CHANGE_RUNBOOK.md): step-by-step triage and promotion checklist

**Pullwatch project links**

- Source: [github.com/dragosdev-code/pullwatch](https://github.com/dragosdev-code/pullwatch)
- Issues and feedback: [github.com/dragosdev-code/pullwatch/issues](https://github.com/dragosdev-code/pullwatch/issues)
- Privacy policy: [dragosdev-code.github.io/pullwatch/privacy/](https://dragosdev-code.github.io/pullwatch/privacy/)

## License

Same terms as [Pullwatch](https://github.com/dragosdev-code/pullwatch/blob/main/LICENSE) (MIT). This config repository is part of that project’s operational tooling.
