# rust_simple

Minimal single-crate Rust project template with strict lints, `tracing`,
`thiserror`/`anyhow`, `serde`, a `Justfile`, `bacon` config, and CI.

## Quickstart

```bash
just run      # cargo run
just test     # cargo test --all-targets --all-features
just lint     # cargo clippy ... -D warnings
just fmt      # cargo fmt
bacon         # background compile/test loop
```

See [`AGENTS.md`](AGENTS.md) for code style, lint policy, and conventions.

## CI

`.github/workflows/ci.yml` runs `lint` and `test` on push and PR against
`main`/`master`.

## Dependabot

Configured in `.github/dependabot.yml` to update `cargo` and `github-actions`
daily, aggressively: direct and transitive deps, including major bumps,
grouped into one PR per ecosystem.

The `merge` job in `ci.yml` auto-approves and enables auto-merge (squash) for
Dependabot PRs once `lint` and `test` pass.

### Required one-time GitHub setup

These repo settings are required. All three are scriptable via `gh`; UI paths
shown for reference. Run from inside the repo (replace `OWNER/REPO` if needed).

1. **Allow auto-merge** (UI: Settings > General > Pull Requests).

   ```bash
   gh repo edit --enable-auto-merge
   ```

2. **Allow GitHub Actions to create and approve pull requests** (UI:
   Settings > Actions > General > Workflow permissions). No `gh` flag exists;
   use the REST API:

   ```bash
   gh api -X PUT repos/{owner}/{repo}/actions/permissions/workflow \
     -F default_workflow_permissions=read \
     -F can_approve_pull_request_reviews=true
   ```

3. **Branch protection requiring `Lint` and `Test`** on `main` (UI:
   Settings > Branches). Without this, `gh pr merge --auto` merges
   immediately and CI is bypassed.

   ```bash
   gh api -X PUT repos/{owner}/{repo}/branches/main/protection \
     -H "Accept: application/vnd.github+json" \
     --input - <<'JSON'
   {
     "required_status_checks": {
       "strict": true,
       "checks": [{ "context": "Lint" }, { "context": "Test" }]
     },
     "enforce_admins": false,
     "required_pull_request_reviews": null,
     "restrictions": null
   }
   JSON
   ```

### GitHub token

The merge job uses the built-in `secrets.GITHUB_TOKEN`; no PAT needed.

Dependabot-triggered workflows get a read-only `GITHUB_TOKEN` by default.
The `merge` job elevates this via an explicit job-level `permissions:` block
(`contents: write`, `pull-requests: write`), which Dependabot runs respect on
`pull_request` events. The `lint` and `test` jobs need no elevation.

Caveat: merges performed with `GITHUB_TOKEN` do not trigger downstream
workflows (e.g. a release workflow listening on `push`). If you later need
that, swap in a PAT or GitHub App token via repo secrets.

### Tradeoffs

- Grouped wildcard PRs mean one broken transitive bump fails the whole PR;
  you lose per-dep granularity. Acceptable for an aggressive update policy.
- Auto-merging major bumps without human review relies entirely on CI as the
  safety net. The strict clippy + test config is the backstop.

## License

MIT, see [`LICENSE`](LICENSE).
