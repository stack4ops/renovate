# renovate

Self-hosted Renovate for stack4ops repositories.

| File | Role |
| --- | --- |
| `renovate.json` | Global config: which repositories the bot runs on, tool installation. |
| `default.json` | Shared preset. Repositories opt in with `"extends": ["local>stack4ops/renovate"]`. |
| `.github/workflows/renovate.yml` | Runs the bot (nightly, or manually with dry run). |
| `.github/workflows/nix-hash-fixup.yml` | Reusable workflow that fixes Nix hashes on Renovate branches. |

## Adding a repository

1. Add it to `repositories` in `renovate.json`.
2. In the repository, add `renovate.json` with `{"extends": ["local>stack4ops/renovate"]}`.
3. Annotate versions Renovate should track (see below).
4. If a bump changes Nix hashes, call `nix-hash-fixup.yml` from a `pull_request` workflow.
5. Branch protection on the default branch: require the CI status, and enable
   "Allow auto-merge" in the repository settings. Without a required check,
   GitHub merges immediately.

## Annotating versions

The preset tracks any version with a `# renovate:` comment on the line above:

```nix
# renovate: datasource=github-releases depName=project-zot/zot
version = "2.1.17";
```

```yaml
# renovate: datasource=github-releases depName=project-zot/zot
image: zot:2.1.17
```

A leading `v` in upstream tags is stripped by default. Override with
`extractVersion=<regex>` in the comment if a project tags differently.

## Secrets

- `RENOVATE_TOKEN` here: a PAT with access to every tracked repository.
- In each repository that uses `nix-hash-fixup.yml`: a token with
  `contents:write` on that repository, passed as the `token` secret.

The reusable workflow is only callable from other repositories if this
repository is public, or if both are private in the same organization with
workflow access granted.
