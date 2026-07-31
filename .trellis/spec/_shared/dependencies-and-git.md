# Dependencies And Git

## When This Applies

Adding or upgrading a dependency, touching any lockfile, and every commit.

## The Local Pattern

### Exact versions only

`scripts/check-pinned-deps.mjs` scans every `package.json` in the repo and
fails unless each entry of `dependencies`, `devDependencies`, and
`optionalDependencies` is a literal semver. Two exemptions are hard-coded:

- internal workspace deps (`@earendil-works/pi-*`)
- non-registry specifiers (`workspace:`, `file:`, `link:`, `git+`, `github:`,
  `https:`, ...)

So `"esbuild": "0.28.1"` is valid; `"esbuild": "^0.28.1"` fails the check.

### Install without lifecycle scripts

```bash
npm install --ignore-scripts          # local hydrate/update
npm ci --ignore-scripts               # clean / CI-style
npm install --package-lock-only --ignore-scripts   # metadata-only lockfile refresh
```

CI does the same (`.github/workflows/ci.yml` → `npm ci --ignore-scripts`). Do
not run lifecycle scripts unless the user asks.

### Two generated lockfiles for coding-agent

| Artifact | Generator | Guard |
|---|---|---|
| `packages/coding-agent/npm-shrinkwrap.json` | `node scripts/generate-coding-agent-shrinkwrap.mjs` | `--check` mode inside `npm run check` |
| `packages/coding-agent/install-lock/` | `node scripts/generate-coding-agent-install-lock.mjs` | `--check` mode inside `npm run check` |

Both scripts carry an explicit allowlist of packages permitted to ship install
scripts, with a written reason per entry — see `allowedInstallScriptPackages`
in `scripts/generate-coding-agent-install-lock.mjs:17`:

```js
["@google/genai@1.52.0", "preinstall is a no-op in the published package"],
["protobufjs@7.6.5", "postinstall only warns about protobufjs version scheme mismatches"],
```

A new dependency with lifecycle scripts requires review and an explicit
allowlist entry. Never add one silently. The shrinkwrap generator also fails
when an allowlisted package disappears, so stale entries must be removed.

### Lockfile commits are blocked by default

`.husky/pre-commit` runs `scripts/check-lockfile-commit.mjs` first. It compares
`HEAD:package-lock.json` against the staged version and aborts unless
`PI_ALLOW_LOCKFILE_CHANGE` is `1` / `true` / `yes`. One diff shape is exempt:
`hasOnlyWorkspacePackageChanges` (`check-lockfile-commit.mjs:54`, applied at
`:92`) lets through a diff that only restates workspace package metadata, which
is what a version bump produces. Set the env var only when the lockfile change
is genuinely intended for that commit.

### Multi-session git discipline

Several agent sessions may share this working directory, each editing different
files. Consequences:

- Stage explicit paths, never a directory sweep:
  `git add packages/ai/src/models-store.ts packages/ai/test/models-runtime.test.ts`
- Run `git status` before committing and confirm only your files are staged
- `packages/ai/src/models.generated.ts` may always be included alongside your files
- Commit only when the user asks

Forbidden outright — these destroy other sessions' work or bypass the gate:
`git reset --hard`, `git checkout .`, `git clean -fd`, `git stash`,
`git add -A`, `git add .`, `git commit --no-verify`.

On rebase conflicts: resolve only files you modified; if a conflict lands in a
file you did not touch, abort and ask. Never force push.

### Commit message format

```
{feat,fix,docs}[(ai,tui,agent,coding-agent)]: <message>
```

Informative and concise, no emojis. To auto-close issues, repeat the keyword
per issue — `closes #1, closes #2`. A shared keyword (`closes #1, #2`) only
closes the first.

## Reference Files

- `scripts/check-pinned-deps.mjs`, `scripts/check-lockfile-commit.mjs`
- `scripts/generate-coding-agent-shrinkwrap.mjs`,
  `scripts/generate-coding-agent-install-lock.mjs`
- `.husky/pre-commit`, `.npmrc`, root `package.json` (`overrides`)
- `AGENTS.md` — "Dependency and Install Security", "Git"

## Anti-Patterns

| Anti-pattern | Consequence |
|---|---|
| Range specifier (`^`, `~`, `*`) on an external dep | `check:pinned-deps` hard fail |
| `git add -A` / `git add .` | Stages another session's in-flight work |
| `PI_ALLOW_LOCKFILE_CHANGE=1` set out of habit | Smuggles unreviewed lockfile diffs into unrelated commits |
| Adding a lifecycle-script dep without an allowlist entry + reason | `check:shrinkwrap` / `check:install-lock` hard fail, and a real supply-chain risk |
| Hand-editing `npm-shrinkwrap.json` or `install-lock/` | Regenerate with the owning script instead |
