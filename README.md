# renovate

Self-hosted Renovate for stack4ops repositories.

| File | Role |
| --- | --- |
| `config.json` | Global config: which repositories the bot runs on, tool installation. |
| `renovate.json` | This repository's own repo config, so its pinned digests get updated. |
| `default.json` | Shared preset. Repositories opt in with `"extends": ["local>stack4ops/renovate"]`. |
| `.github/workflows/renovate.yml` | Runs the bot (two nightly windows, or manually with dry run). |
| `.github/workflows/nix-hash-fixup.yml` | Reusable workflow that fixes Nix hashes on Renovate branches. |
| `.github/workflows/container-smoke-test.yml` | One-off manual probe of the Nix container image. |

The global config is **not** called `renovate.json`, and that is deliberate.
This repository is itself tracked now, and Renovate reads a root
`renovate.json` as the repository's own config -- where global-only options
like `autodiscover` and `repositories` are a validation error. Hence
`config.json` for the global half and `renovate.json` for the repo half.

## Tracked repositories and why the order matters

| Repository | What Renovate updates there | Hash fixup | What gates the merge |
| --- | --- | --- | --- |
| `stack4ops/renovate` | pinned action digests and the Nix container digest | n/a | nothing -- read by hand |
| `stack4ops/pkgs` | the annotated `version` in each package, plus `flake.lock` | yes | its own `flake check` job |
| `stack4ops/images` | `flake.lock` only (nixpkgs and the `stack4ops-pkgs` input) | no | argunix |

The dependency runs one way: `images` builds from the overlay in `pkgs`, and
that dependency lives entirely in `images`' lock file, because the flake input
tracks a branch rather than a tag. There is therefore no version string in
`images` for Renovate to bump -- only `lockFileMaintenance` moves it.

So the two repositories are processed in separate windows:

```
02:17  stack4ops/renovate  action and container digests (no dependents)
       stack4ops/pkgs      version bump -> hash fixup -> flake check
                           -> platform automerge lands it on main
04:17  stack4ops/images    lockFileMaintenance moves stack4ops-pkgs to that main
                           -> argunix builds and publishes the new image
```

Two hours is enough for the package build and the automerge. If the first half
has not landed by 04:17 nothing breaks -- the images run finds the old main and
picks the bump up the next night. `images` overrides the preset's weekly
`lockFileMaintenance` schedule to nightly for exactly this reason; `pkgs`,
whose only input is a nixpkgs release branch, keeps the weekly default.

Worth knowing when a bump feels slow: the preset soaks annotated patch and
minor updates for `minimumReleaseAge: 3 days` before opening the PR at all.
A zot release reaches the registry on the fourth night, not the first.

## Where the required check comes from

Automerge is only safe behind a required status, and the two repositories get
theirs from different places.

`stack4ops/pkgs` gates on its own `.github/workflows/ci.yml`, whose job is
named **flake check**. argunix does not watch that repository: argunix exists
to build and publish artefacts, nothing there carries `meta.image-format`, and
the check that decides a merge belongs where the merge decision is made.

`stack4ops/images` gates on argunix, which evaluates and builds it and reports
back to the forge. Confirm you have actually seen an argunix status on a GitHub
pull request before relying on it -- without a required check GitHub merges a
Renovate PR the moment it opens.

Either way, branch protection on `main` must name the check **and** have
"Allow auto-merge" enabled.

## Adding a repository

1. Add it to `repositories` in `config.json`, in dependency order.
2. In the repository, add `renovate.json` with `{"extends": ["local>stack4ops/renovate"]}`.
3. Annotate versions Renovate should track (see below).
4. **Only if the repository contains fixed-output hashes** (`src.hash`,
   `vendorHash`, `cargoHash`, `npmDepsHash`): add a `pull_request` workflow
   calling `nix-hash-fixup.yml`. An images repository does not need one.
5. If it consumes another stack4ops flake, give it a nightly
   `lockFileMaintenance` schedule and a bot window after the repository it
   depends on.
6. Give it a required check -- argunix if the repository publishes artefacts,
   otherwise a `nix flake check` workflow of its own -- and enable branch
   protection on the default branch naming that check, plus "Allow auto-merge".

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

Without the comment the line is invisible to Renovate. That is the usual
reason a dependency silently stops being updated.

## Secrets

- `RENOVATE_TOKEN` here: a PAT with access to every tracked repository.
- In each repository that uses `nix-hash-fixup.yml`: a token with
  `contents:write` on that repository, passed as the `token` secret.

The reusable workflow is only callable from other repositories if this
repository is public, or if both are private in the same organization with
workflow access granted.

## The Nix container image

`nix-hash-fixup` runs in
`registry.opencode.de/oci-community/images/applicative-systems/nix/nix`,
digest-pinned, instead of installing Nix through
`cachix/install-nix-action`. The chain now depends on no third-party action
inside that job.

The image reference is written out literally in `container.image` rather than
passed in as an input, because Renovate's `github-actions` manager reads that
field as a docker dependency and keeps the digest current. An expression there
would be an opaque string and the pin would rot. `:main` is a moving tag kept
only so Renovate knows which stream to follow; the digest is what pins.

`actions/checkout` is gone from that job. GitHub mounts its own glibc-linked
node into container jobs to execute JS actions, and this image has no dynamic
loader at either `/lib64/ld-linux-x86-64.so.2` or
`/lib/ld-linux-x86-64.so.2` -- measured, not assumed -- so every JS action
fails before its first line. Since the job used no other action, a `git init`
plus `git fetch` with the token replaces checkout outright. Not a workaround,
just fewer moving parts.

What the image provides, under `/root/.nix-profile/bin`: bash, git, nix 2.34,
tar, gzip. No jq, which nothing in this job needs. `NIX_CONFIG` sets
`extra-experimental-features` rather than `experimental-features`, so it
appends `nix-command flakes` instead of replacing whatever the image's own
`nix.conf` enables.

The first step re-checks that nix and git are reachable and falls back to the
profile path or to `nixpkgs#git`. Both branches are dormant today; they are
there because `:main` is a moving tag and a rebuilt image could drop a tool or
land it off PATH.

### One thing still unverified

Run `container-smoke-test.yml` once from the Actions tab. A local `docker pull`
cannot tell you whether GitHub's runners can pull the image **without**
credentials -- a cached opencode login would hide that. If they cannot, the job
fails at the pull step and needs a `credentials:` block with a read-only pull
secret. The same run also shows whether the nix profile is on PATH inside a
container job, which decides whether the fallback above is dormant or load-bearing.
