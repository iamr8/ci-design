# CI/CD Blueprint: release branches, hotfixes, manual releases, GitHub Actions

A design for the CI and release pipeline of a small public repository on
GitHub Actions. It does not depend on a language, a framework or a package
type. Each repository fills in the placeholders in section 2. The rest stays
the same.

Two project profiles are covered: **app** (installable software that users
download) and **web** (a web app or API that you deploy). Section 6.4 lists
where they differ. Everything else is the same for both.

The design assumes a public repository. There, rulesets, required checks,
CodeQL, environments and artifact attestations are free. The design uses all
of them.

## TL;DR

| Environment | Runs | How it is deployed | Promote to the next environment |
|---|---|---|---|
| Canary (isolated, one per PR) | Each open PR into `<DEV>`, `release/**` or `hotfix/**`, from this repository (not sync PRs) | PR preview on each push (section 14): `app` gives a download, `web` deploys to its own `<PREVIEW_ENV>`. Fork PRs run CI only, with no preview. | Merge the PR into `<DEV>`. |
| Dev | `<DEV>`, the unstable main line: every merged feature | Dev build to `<DEV_ENV>`. `<DEV_BUILD_MODE>`: manual (`workflow_dispatch`) or automatic on each merge (section 5.6). | Promote to release (the cut): it creates `release/X.Y.0` and the Release PR. |
| Staging (RC) | The Release PR head (`release/**`): the next version, which replaces production | Pre-release to `<STAGING_ENV>`. `<RELEASE_RC_MODE>`: manual or automatic on each push (section 5.5). Stacked PRs merged into the branch update it. | Merge the Release PR into `<MAIN>`. |
| Hotfix | The Hotfix PR head (`hotfix/**`): production plus its fixes | Pre-release to `<HOTFIX_ENV>`. `<HOTFIX_RC_MODE>`: manual or automatic on each push (section 5.5). Fix PRs merged into the branch update it. | Merge the Hotfix PR into `<MAIN>`: it patches production. |
| Prod | `<MAIN>`, the stable main line: the current release | Deploy to `<PROD_ENV>`, then tag and release (section 8.8). | - |

The environments are for `web`. For `app`, each build is a download instead:
a dev build as a workflow artifact, a pre-release or a release (section 6.4).

- Feature PRs go into `<DEV>`. Promote to release cuts `release/X.Y.0`; fixes
  for it go into that branch, and its Release PR merges into `<MAIN>`.
- After every merge into `<MAIN>`, the bot opens a Hotfix PR for the shipped
  version. Fixes for production go there. An idle Hotfix PR blocks nothing.
- Every merge into `<MAIN>` comes back to `<DEV>` as a squashed sync PR.
- A milestone per version lists what shipped in it.

## Contents

1. [How to use this document](#1-how-to-use-this-document)
2. [Placeholders](#2-placeholders)
3. [Goals and invariants](#3-goals-and-invariants)
4. [The big picture](#4-the-big-picture)
5. [Branch and version model](#5-branch-and-version-model)
6. [Repository setup: rulesets, environments, release bot, settings](#6-repository-setup-rulesets-environments-release-bot-settings)
7. [Workflow catalog](#7-workflow-catalog)
8. [Workflows in detail](#8-workflows-in-detail)
9. [Security model](#9-security-model)
10. [Reliability model](#10-reliability-model)
11. [Shared steps and their contracts](#11-shared-steps-and-their-contracts)
12. [Testing the CI itself](#12-testing-the-ci-itself)
13. [Release notes](#13-release-notes)
14. [PR preview builds](#14-pr-preview-builds)
15. [Setup checklist for a new repository](#15-setup-checklist-for-a-new-repository)
16. [Stack examples](#16-stack-examples)
17. [Threats and controls](#17-threats-and-controls)
18. [Release branches and hotfixes](#18-release-branches-and-hotfixes)
19. [Sync into DEV and into an open release branch](#19-sync-into-dev-and-into-an-open-release-branch)
20. [Milestones](#20-milestones)
21. [Policy checks in the gate](#21-policy-checks-in-the-gate)
22. [Edge cases](#22-edge-cases)
23. [Design decisions](#23-design-decisions)
24. [Appendix: YAML skeletons](#24-appendix-yaml-skeletons)

## 1. How to use this document

- Sections 3 to 14 and 18 to 21 describe the design. Section 5 is the model in
  one place; read it first.
- Pick a profile first: app or web (section 6.4).
- Section 2 lists what you replace for each repository. Section 16 shows
  values for common stacks.
- Section 6 and section 15 are the setup steps.
- Section 17 maps each threat to the control that stops it, and section 22
  lists the edge cases. Read both before you remove any part of the design.
- Section 24 has short YAML skeletons for the patterns that are easy to get
  wrong.

## 2. Placeholders

| Placeholder | Meaning |
|---|---|
| `<READ_VERSION_CMD>` | Prints the version from `VERSION`. |
| `<BASE_VERSION>` | The release version, `X.Y.Z`. |
| `<BUILD>` | Build number: `github.run_number` of the workflow that builds (`build-develop.yml` for dev builds, `pre-release.yml` for pre-releases). |
| `<DEV_VERSION>` | Version of a dev build, set inside the artifact, for example `X.Y.0-dev.<BUILD>`. Must sort below its own release (section 5.7). A dev build gets no `v*` tag. |
| `<RC_TAG>` | Tag of a pre-release: SemVer `vX.Y.Z-rc.<BUILD>`, PEP 440 `vX.Y.ZrcN` (`N` = `<BUILD>`) (section 5.7). |
| `<REL_TAG>` | Tag of a release, `vX.Y.Z`. |
| `<TOOLCHAIN_SETUP>` | Action that installs the language toolchain. |
| `<TEST_CMD>` | Runs all unit tests. |
| `<GUARD_CMD>` | Project guard: checks project rules that no generic linter knows. |
| `<PACKAGE_CMD>` | Builds the release artifact for a given version. |
| `<ARTIFACT>` | Name prefix of release files, for example `myapp-<version>`. |
| `<PROFILE>` | `app` (installable software users download) or `web` (a web app or API you deploy). Section 6.4. |
| `<DEPLOY_CMD>` | Web only. Deploys a built artifact to an environment. |
| `<PREVIEW_ENV>` | Web only. The environment that PR previews (canary) deploy to: small, one isolated deploy per PR (section 14). |
| `<DEV_ENV>` / `<STAGING_ENV>` / `<HOTFIX_ENV>` / `<PROD_ENV>` | Web only. GitHub environments that dev builds, release branch pre-releases, hotfix branch pre-releases and releases deploy to, for example `dev` / `staging` / `hotfix` / `production` (section 6.4). |
| `<LOCKED_INSTALL>` | The ecosystem's install command that uses only the lockfile and runs no install scripts. |
| `<TOOLS_DIR>` | Folder with the CI-only tools (linters) and their lockfile. |
| `<LINT_CMD>` | Security-focused linter for the language. |
| `<SAST_RULES>` | The repository's own blocking Semgrep rules. |
| `<CODEQL_LANG>` | CodeQL language id of the project. `codeql.yml` always adds `actions` (section 8.3). |
| `<DEV>` / `<MAIN>` | The unstable main line and the stable main line, for example `develop` / `main`. |
| `<DEV_BUILD_MODE>` | How dev builds start: `manual` or `automatic` on each merge into `<DEV>` (section 5.6). |
| `<RELEASE_RC_MODE>` / `<HOTFIX_RC_MODE>` | How pre-releases start on `release/**` / `hotfix/**`: `manual` (default) or `automatic` on each push (section 5.5). |
| `<BOT_APP>` | The GitHub App that makes all release writes (section 6.3). |
| `<CARD_HEADING>` | The first line of the PR preview card, used to find the old card (section 8.2). |
| `<ECOSYSTEM>` | Dependabot package ecosystem of a lockfile (section 24.7). |

Rule: CI logic lives in workflow YAML. Logic runs in bash `run:` steps (CLI
tools such as git, gh, curl and jq) or in `actions/github-script` steps. CI
has no code in a language runtime (Python, Node modules, C# and so on). So the
pipeline does not depend on the project's stack. A step that more than one
workflow needs is a composite action under `.github/actions/` or a reusable
workflow. The project's own commands (`<TEST_CMD>`, `<GUARD_CMD>`,
`<PACKAGE_CMD>`) stay in the project's toolchain.

## 3. Goals and invariants

### Goals

1. A broken or unsafe change never reaches a user.
2. No untrusted code ever runs next to a secret or a write token.
3. A version ships once, with notes. To recover from a failed release run,
   run it again.
4. A human decides when to publish. A merge into `<DEV>` alone publishes
   nothing.
5. Every PR can be tried before merge.
6. A user can verify where a release file came from.
7. A fix for production can ship at any time, without the unreleased work on
   `<DEV>`, and it always comes back to `<DEV>`.
8. Milestone `X.Y.Z` lists exactly the PRs (and their issues) that merged into
   `<MAIN>` as `X.Y.Z`, with no human cleanup.

### Invariants

Each rule has one place that enforces it.

| # | Invariant | Enforced by |
|---|---|---|
| I1 | `VERSION` on the branch being built or merged is the only source of the version. No workflow asks for one. | `next-version check --branch`; `release.yml` reads the file. |
| I2 | A build is higher than every release. Pre-release tags do not count. | `next-version check` against all tags. |
| I3 | A version ships once. | Tag ruleset; the `version` job refuses an existing tag. |
| I4 | Builds run only from a manual dispatch (Build develop, Create pre-release), an automatic build where its mode says so (`<DEV_BUILD_MODE>`, `<RELEASE_RC_MODE>`, `<HOTFIX_RC_MODE>`), or a merge into `<MAIN>`. | Triggers. |
| I5 | A dispatch on the wrong branch does nothing. | Job-level `if: github.ref == 'refs/heads/<branch>'` on every job that writes. |
| I6 | Protected branches take PRs only. Only the release bot writes to them directly: the bump commit on `<DEV>`, and its own commits on `release/**` and `hotfix/**`. The repository owner can bypass every rule, for repairs and E3. | Branch rulesets with `<BOT_APP>` and repository admins as the only bypass actors. |
| I7 | A merge needs a green `gate` check. A merge into `<DEV>`, `release/**` or `hotfix/**` also needs clean code scanning; `<MAIN>` takes only Release and Hotfix PRs, whose lines were scanned in their own PRs. | Branch rulesets. |
| I8 | Every build runs the full checks, on the exact commit it builds. | `checks.yml` is a reusable workflow; builds call it. |
| I9 | Release and Hotfix PRs merge into `<MAIN>` with a merge commit. `<DEV>` gets their content by squashed sync PRs, not their history. | `<MAIN>` ruleset: merge commit only; `dev` ruleset: squash only. |
| I10 | Only the release bot creates, moves or deletes release tags. | Tag ruleset on `v*`. |
| I11 | No job that runs PR code holds a write token or a secret. | Top-level `permissions: contents: read`; `persist-credentials: false`; secrets only in environments. |
| I12 | A job with a write token on a PR event runs no file from the PR. It checks out nothing, or only the workflow's own commit (the default branch). | `canary` and `pr-guard.yml` designs (sections 9.3, 21.1). |
| I13 | Publish credentials exist only in jobs that run on `<DEV>` or `<MAIN>`. On `release/**` and `hotfix/**`, only the bot key and the staging and hotfix deploy credentials exist. | Environments with deployment branch policies (sections 6.2, 6.4). |
| I14 | The tag and the GitHub release are created last, after any deploy. | `publish` job after `build` (and after `deploy`). |
| I15 | Side effects (milestones, Hotfix PR, sync PRs) are their own jobs. Their failure never undoes a release. | Separate jobs in `release.yml` (section 8.8). |
| I16 | Every release has notes. | GitHub generated notes from the previous release tag, grouped by `.github/release.yml` (section 13). |
| I17 | Every action is pinned to a full commit SHA. Every CI tool is pinned by a lockfile or an exact version. | Repository setting "require SHA pinning"; zizmor `adhoc-packages` audit; Dependabot. |
| I18 | Every release file has a build provenance attestation. GitHub shows its SHA-256 digest. | `build.yml`. |
| I19 | The artifact holds only what the build declares to ship. A new dev file cannot leak into it. | `<PACKAGE_CMD>`. |
| I21 | Workflows themselves pass a linter and a security audit. | `workflows` job (actionlint + zizmor). |
| I22 | A build never changes the working tree. The version goes into the artifact only. | `<PACKAGE_CMD>`. |
| I23 | One thing ships at a time: while a Hotfix PR has work (section 18.3), no release is cut or merged. An idle Hotfix PR blocks nothing. | P5; Promote to release. |
| I24 | Every merge into `<MAIN>` comes back to `<DEV>`. | `sync-develop` (section 19.2). |
| I25 | A closed milestone is final. | `milestone-close` never reopens or reassigns one (section 20). |

## 4. The big picture

```mermaid
flowchart LR
  subgraph PR["Pull requests"]
    direction TB
    ci["CI: checks.yml, package, preview, gate"]
    cql["CodeQL"]
  end

  subgraph DEV["DEV branch"]
    direction TB
    devb["Build develop<br/>(manual or automatic)"]
    cr["Promote to release<br/>(manual: the cut)"]
  end

  subgraph RB["release/X.Y.0 and hotfix/X.Y.Z"]
    direction TB
    pre["Create pre-release<br/>(manual or automatic)"]
  end

  subgraph MAIN["MAIN branch"]
    direction TB
    rel["Release: checks, build, deploy, publish"]
    fol["At the merge: milestone, next Hotfix PR, sync PRs"]
  end

  PR -->|"merge (gate green)"| DEV
  PR -->|"stacked or fix PR"| RB
  cr -->|"cut + Release PR"| RB
  RB -->|"Release PR or Hotfix PR"| MAIN
  devb -->|"artifact or DEV_ENV"| OUT[("GitHub Releases<br/>or environments")]
  pre -->|"RC_TAG, STAGING_ENV or HOTFIX_ENV"| OUT
  rel -->|"REL_TAG or PROD_ENV"| OUT
  fol -->|"sync PR (squash)"| DEV
  fol -->|"next Hotfix PR"| RB

  style ci fill:#1976d2,stroke:#0d47a1,stroke-width:2px,color:#fff
  style cql fill:#1976d2,stroke:#0d47a1,stroke-width:2px,color:#fff
  style devb fill:#f57c00,stroke:#e65100,stroke-width:2px,color:#fff
  style cr fill:#f57c00,stroke:#e65100,stroke-width:2px,color:#fff
  style pre fill:#f57c00,stroke:#e65100,stroke-width:2px,color:#fff
  style rel fill:#d32f2f,stroke:#b71c1c,stroke-width:2px,color:#fff
  style fol fill:#388e3c,stroke:#1b5e20,stroke-width:2px,color:#fff
  style OUT fill:#7b1fa2,stroke:#4a148c,stroke-width:2px,color:#fff
```

Colors: blue runs PR code with no secrets. Green is bot work on merged code.
Orange is a manual run, or a dev build, that uses an environment. Red is the
stable release.

Every release creates a tag and a GitHub release with its notes. How the
build reaches its users depends on `<PROFILE>` (section 6.4). An app attaches
its files to the GitHub release. A web project deploys to an environment.

## 5. Branch and version model

### 5.1 Branches

- `<DEV>` is the default branch and the unstable main line: every merged
  feature, not yet tested as a release. Feature PRs target it. Dev builds come
  from it.
- Promote to release cuts `release/X.Y.0` from `<DEV>`. From then on the
  branch stays separate from `<DEV>`. Stacked fix PRs target it, and its
  Release PR merges into `<MAIN>`.
- After each merge into `<MAIN>`, the bot creates `hotfix/X.Y.Z` from
  `<MAIN>`. Fix PRs target it, and its Hotfix PR merges into `<MAIN>`.
- `<MAIN>` is the stable main line. It holds only released code. It gets a
  commit only when a Release PR or a Hotfix PR merges.
- `sync/**/*` branches carry the bot's sync PRs (section 19).
- Feature branches can have any name. GitHub deletes them on merge.

### 5.2 The model in four rules

Section 5.8 shows the model as a diagram, with real values.

Four rules carry the whole model:

1. **At the cut, the paths split.** `release/X.Y.0` keeps `X.Y.0`. In the same
   run, the bot moves `<DEV>` to `X.(Y+1).0`. From then on, each branch's
   `VERSION` names the version its PRs ship in.
2. **Every merge into `<MAIN>` comes back.** After each Release PR or Hotfix
   PR merge, the bot opens a sync PR into `<DEV>`. It carries that branch's
   own commits without `VERSION`, and it merges by squash. A hotfix that
   ships during a release also goes into the release branch (19.5).
3. **A branch into `<MAIN>` merges only when complete.** It has no open
   stacked PRs, and a Hotfix PR needs at least one fix (section 21). A
   Release PR that misses a shipped hotfix conflicts on `VERSION`, so GitHub
   blocks it until the sync PR merges (section 19.5).
4. **One thing ships at a time.** While a Hotfix PR has work (section 18.3),
   no release can be cut or merged. An idle Hotfix PR blocks nothing. The bot
   closes it only when the next release ships; it never closes a Hotfix PR
   that has work (section 18.2).

### 5.3 The version file

**Format.**

- One file, `VERSION`, at the repository root.
- Content: exactly `X.Y.Z` and one line feed. No `v`, no words, no spaces, no
  byte-order mark, no carriage return.
- `.gitattributes`: `VERSION text eol=lf`.
- The guard rejects anything else.

**Other manifests read it.**

A manifest that needs the version reads `VERSION` at build time and holds no
version of its own.

| Stack | How the build reads `VERSION` |
|---|---|
| .NET / ASP.NET Core | `Directory.Build.props` sets `VersionPrefix` from the file (MSBuild `File::ReadAllText`, trimmed). `dotnet publish` and `dotnet pack` use it. |
| Node | The package step writes the version into the packed `package.json`; the committed one stays `0.0.0`. |
| Python | Dynamic version from a file (for example `setuptools` `version = {file = "VERSION"}`). |
| Go | `-ldflags "-X main.version=$(cat VERSION)"`. |

**Who writes it.**

| Writer | When | Branch | Value |
|---|---|---|---|
| The bot | At the cut | `<DEV>` | `X.(Y+1).0` |
| The bot | When it creates a hotfix branch | `hotfix/X.Y.(Z+1)` | `X.Y.(Z+1)` |
| A trusted human, in a version-jump PR | Major or other jump | `<DEV>` | any version the `<DEV>` rule in section 5.4 accepts |

Nobody else writes it. The release branch never changes it. It keeps the value
`<DEV>` had at the cut.

### 5.4 Version rules by branch kind

`next-version check --branch <ref>` applies these rules.

| Branch | Rule |
|---|---|
| `<DEV>` | `Z` is 0. Higher than every release tag. Higher than every open release branch's version. |
| `release/X.Y.0` | `VERSION` equals the branch name. `Z` is 0. Higher than every release tag. |
| `hotfix/X.Y.Z` | `VERSION` equals the branch name. `Z` above 0. Higher than every release in line `X.Y`. Line `X.Y` is the line of `VERSION` at the head of `<MAIN>` (the latest merged line, shipped or not; E43). |
| `<MAIN>` (in `release.yml`) | The rule of the merged branch kind, read from the version (`Z` = 0: release; `Z` above 0: hotfix). The tag must not exist. |

Pre-release tags never count as releases.

General rules:

1. Feature and fix PRs never change the version.
2. A major jump, or any other jump, is its own PR into `<DEV>` that changes
   only `VERSION`. A trusted human decides it.
3. A build must be higher than the last release. A version tried as a dev
   build or a pre-release can still ship as a release.
4. A release `X.Y.Z` sorts above all of its own dev builds and pre-releases
   (section 5.7).
5. The build passes the version to the artifact (build property, package
   metadata). It never writes it back to the working tree.

### 5.5 Build kinds

| Build | From | What it is | Tag (SemVer) | Workflow |
|---|---|---|---|---|
| Dev build | `<DEV>` | Raw: the merged PRs added up. Can have many bugs. | none; the artifact carries `X.Y.0-dev.<BUILD>` | `build-develop.yml`, `<DEV_BUILD_MODE>` |
| Pre-release (release candidate) | `release/X.Y.0`, `hotfix/X.Y.Z` | Staging or hotfix: the version as it will ship, for a last test. | `vX.Y.Z-rc.<BUILD>` | `pre-release.yml`, `<RELEASE_RC_MODE>` / `<HOTFIX_RC_MODE>` |
| Release | `<MAIN>` | Stable. | `vX.Y.Z` | `release.yml` |

All three call one reusable `build.yml` (section 8.7).

What each build kind delivers per profile, and the `web` environments:
section 6.4.

The Release PR and the Hotfix PR are the release candidates. The head of each
is exactly what will ship. So testers, `<STAGING_ENV>` and `<HOTFIX_ENV>`
should run it. `pre-release.yml` builds that head. Two options
decide when, one per branch kind:

| Option | Branches | `manual` (default) | `automatic` |
|---|---|---|---|
| `<RELEASE_RC_MODE>` | `release/**` | A build only when someone runs Create pre-release. | Also a build on each push: each merged stacked PR and each bot commit. |
| `<HOTFIX_RC_MODE>` | `hotfix/**` | Same. | Same, for each merged fix PR and the bot's version commit. |

- Pick `automatic` when testers must follow every merge. Keep `manual` when
  each build or deploy costs too much.
- Manual dispatch stays in both modes. With `automatic`, it is how you rerun a
  build.
- Each mode is one entry in the `push.branches` list of `pre-release.yml`.
  So the choice is visible in the file and needs no runtime switch.
- With `<HOTFIX_RC_MODE>` = `automatic`, the Hotfix env runs the new release
  as soon as `next-hotfix` pushes its version commit. Each fix then
  lands on top of it. Staging follows the cut the same way.

### 5.6 Dev builds

The consumer picks `<DEV_BUILD_MODE>` when they configure the repository:

| Mode | Trigger | Good for | Recommended for |
|---|---|---|---|
| Manual | `workflow_dispatch` only | Few testers; each build is a download someone asked for. | `app` |
| Automatic | `workflow_dispatch` plus `push` to `<DEV>` | `<DEV_ENV>` or testers always on the newest code. | `web` |

The mode is the presence of the `push` trigger in `build-develop.yml`. It is one line, so
the choice is visible in the file and needs no runtime switch.

Automatic mode details:

- `push` uses `paths-ignore` for docs and `VERSION`. A merge
  that changes nothing shipped builds nothing. (A path filter on `push` is safe:
  no ruleset waits for this workflow.)
- Concurrency group `build-develop`, no cancel in progress. GitHub keeps one running
  run and one waiting run per group; a newer waiting run replaces the older
  one. So a burst of merges gives one build with all of them, not one build
  each.
- No PR is lost. The newest run checks out its own merge commit, and `<DEV>`
  is linear, so that commit holds every earlier merge. The only cost is that
  a merge in the middle of a burst gets no build of its own.
- The build number still comes from `run_number`. Gaps in the numbers are
  normal.

**Where a dev build goes.** An `app` dev build stays a workflow artifact of
the run: no tag, no GitHub release, no bot token. A download needs a GitHub
login, and artifacts expire after 90 days at most. A `web` dev build deploys
to `<DEV_ENV>`, which always runs the newest build.

### 5.7 Order of tags

SemVer orders `X.Y.0-dev.N` below `X.Y.0-rc.M` below `X.Y.0`. After the
cut, `<DEV>` builds `X.(Y+1).0-dev.N`, so dev builds and pre-releases never
share a version.

Pick a version format that your ecosystem already sorts the right way:

| Version style | `<DEV_VERSION>` | Sorts below `X.Y.Z`? |
|---|---|---|
| SemVer (npm, NuGet, Cargo, Go modules, most others) | `X.Y.Z-dev.<BUILD>` | Yes, by SemVer rules. |
| PEP 440 (Python) | `X.Y.Z.dev<BUILD>` | Yes. |

**`run_number` trap.** `run_number` counts per workflow file. Never rename
`build-develop.yml` or `pre-release.yml`. A new file restarts the count, and new builds get
lower numbers than older builds.

### 5.8 Life of one version

This section follows version `1.2.0` from its first dev build to the next
Hotfix PR. It shows every kind of build and both sync paths. At the start,
`v1.1.0` is in production and `<DEV>` holds `1.2.0`.

```mermaid
gitGraph
  commit id: "older history"
  branch dev
  commit id: "DEV at 1.2.0"
  commit id: "feature A (squash), dev build 1.2.0-dev.41"
  commit id: "feature B (squash), dev build 1.2.0-dev.42"
  checkout main
  commit id: "Release PR 1.1.0" tag: "v1.1.0"
  checkout dev
  branch release/1.2.0
  checkout dev
  commit id: "bot: VERSION 1.3.0" type: HIGHLIGHT
  commit id: "feature C (squash), dev build 1.3.0-dev.43"
  checkout release/1.2.0
  merge main id: "cut: merge MAIN (keeps 1.2.0)" tag: "v1.2.0-rc.7"
  commit id: "stacked fix (squash)" tag: "v1.2.0-rc.8"
  checkout main
  branch hotfix/1.1.1
  commit id: "bot: VERSION 1.1.1"
  commit id: "hotfix (squash)" tag: "v1.1.1-rc.9"
  checkout main
  merge hotfix/1.1.1 id: "Hotfix PR 1.1.1" tag: "v1.1.1" type: HIGHLIGHT
  checkout dev
  commit id: "sync 1.1.1 to dev (squash, no VERSION)"
  checkout release/1.2.0
  merge main id: "sync 1.1.1 into release (merge, keeps 1.2.0)"
  checkout main
  merge release/1.2.0 id: "Release PR 1.2.0" tag: "v1.2.0" type: HIGHLIGHT
  checkout dev
  commit id: "sync 1.2.0 to dev (squash)"
  checkout main
  branch hotfix/1.2.1
  commit id: "bot: VERSION 1.2.1"
```

In the diagram, `main` is `<MAIN>` and `dev` is `<DEV>`. A tag on a commit is
a build of that commit: `-rc.N` from `pre-release.yml`, a plain `vX.Y.Z` from
`release.yml`. A dev build has no tag; its version is in the commit label. For
`web`, the same builds deploy to `<DEV_ENV>`, `<STAGING_ENV>`, `<HOTFIX_ENV>`
and `<PROD_ENV>`.

| # | Step | Version effect | Section |
|---|---|---|---|
| 1 | Feature PRs squash into `<DEV>`. Dev builds `1.2.0-dev.41`, `-dev.42` (manual or automatic). | `<DEV>` `1.2.0` | 5.5 |
| 2 | Hotfix `1.1.1` is open and idle, so it blocks nothing. A human runs Promote to release. The cut creates `release/1.2.0`, merges `<MAIN>` into it (a real merge: `<DEV>` never holds `<MAIN>`'s commits), and moves `<DEV>` to `1.3.0`. | `<DEV>` `1.3.0` | 8.5 |
| 3 | Pre-release `1.2.0-rc.7` for testers or staging. | - | 5.5 |
| 4 | A stacked fix squashes into `release/1.2.0`. Pre-release `-rc.8`. | - | 18.1 |
| 5 | A production bug. A fix PR opens into the idle Hotfix `1.1.1`; the Release PR turns red (P5). The fix PR squashes into it. Pre-release `1.1.1-rc.9`. | hotfix branch `1.1.1` | 18.2, 18.3, 21 |
| 6 | Hotfix `1.1.1` merges and ships, like a release. At the merge: milestone `1.1.1` closes, sync PRs go into `<DEV>` and into `release/1.2.0`, and Hotfix `1.1.2` opens (not shown). | `<MAIN>` `1.1.1` | 18.2, 19, 20.1 |
| 7 | Both sync PRs merge. `<DEV>` keeps `1.3.0`, the release branch keeps `1.2.0`, and the Release PR no longer conflicts on `VERSION`. Hotfix `1.1.2` is idle, so P5 is green. | - | 19.2, 19.5 |
| 8 | Release `1.2.0` merges and ships. At the merge: milestone `1.2.0` closes, "Sync 1.2.0 to `<DEV>`" opens, the bot closes the idle Hotfix `1.1.2` with a comment, and Hotfix `1.2.1` opens. | `<MAIN>` `1.2.0` | 18.1, 19.2, 18.2 |

## 6. Repository setup: rulesets, environments, release bot, settings

### 6.1 Rulesets

| Ruleset | Target | Rules |
|---|---|---|
| `dev` | `<DEV>` | Require a PR. Require status checks `gate` and `policy`. Require code scanning results (CodeQL; high or higher blocks). Block force push and deletion. Allowed merge method: squash only (sync PRs squash too, section 19.2). |
| `main` | `<MAIN>` | Require a PR. Require status checks `gate` and `policy`. No code scanning rule: every line on a Release or Hotfix PR head was scanned in its own PR (section 8.3). Block force push and deletion. Allowed merge method: merge commit only. |
| `release-branches` | `release/**` | Require a PR. Require `gate`, `policy` and code scanning. Allowed merge methods: squash (stacked fixes) and merge commit (sync PRs, section 19.5). |
| `hotfix-branches` | `hotfix/**` | Require a PR. Require `gate`, `policy` and code scanning. Allowed merge method: squash. |
| `bot-branches` | `release/**`, `hotfix/**`, `sync/**/*` | Restrict creation. |
| `no-force-push` | `release/**`, `hotfix/**`, `sync/**/*` | Block force push; only admins bypass. |
| `release-tags` | tags `v*` | Restrict creation, update and deletion. |

Bypass actors, and nobody else (bypass mode: always):

- Repository admins on every ruleset. The admin role covers the repository
  owner. Admins repair what automation cannot and merge a PR that P10 blocks
  (section 21).
- `<BOT_APP>` on `dev` (the bump commit at the cut), `release-branches` (the
  cut's merge of `<MAIN>`), `hotfix-branches` (the version commit),
  `bot-branches` and `release-tags`.

A bypass skips every rule in its ruleset, force push included. So force push
lives in its own `no-force-push` ruleset, which the bot cannot bypass. A leaked
bot key then cannot rewrite release, hotfix or sync branches. `GITHUB_TOKEN`
cannot be a bypass actor. That is one reason the bot is an app.

Ruleset patterns: `**` at the end does not cross a `/`. `release/**` matches
`release/1.2.0`, but `sync/**` does not match `sync/dev/1.1.1`, so sync
branches use `sync/**/*`.

"Require branches to be up to date" is off. The push run on `<DEV>` catches a
broken mix of two PRs right after the merge. Turn it on when several people
merge every day and a red `<DEV>` costs more than the extra rebases.

There are two required status checks. `gate` comes from `ci.yml` (section
10.4). `policy` has `<BOT_APP>` pinned as its source, so only `pr-guard.yml`
can post it (section 21.1).

No ruleset requires an approving review. Sync PRs merge back work that is
already merged, and they merge by themselves once `gate` is green (section
19). An approval rule would make each one wait for a human.

### 6.2 Environments: `release`, `bot` and `guard`

**`release`:**

- Deployment branches: `<DEV>` and `<MAIN>` only.
- Secret: the bot's private key (`BOT_APP_KEY`). Variable: the bot's app ID
  (`BOT_APP_ID`).
- Any publish credential (feed key, registry token) goes here too. Deploy
  credentials live in their own environments (section 6.4).
- Optional: a required reviewer, so a human approves each publish.

**`bot`:**

- Deployment branches: `release/**` and `hotfix/**`.
- Only the bot key and app ID. No publish credential, no required reviewer.
- `app` pre-releases use it. Both branch kinds take PRs only, so their
  workflow files are reviewed like `<DEV>`'s. A `web`
  pre-release deploys with the credentials of `<STAGING_ENV>` or `<HOTFIX_ENV>`.

**`guard`:**

- Deployment branches: `<DEV>`, `<MAIN>`, `release/**` and `hotfix/**`. A
  `pull_request_target` run always runs on the default branch.
- Only the bot key and app ID. `pr-guard.yml` uses it, with a token limited to
  `contents: read`, `pull-requests: read` and `statuses: write` (section 21.1).

A job reads these only with `environment: release`, `bot` or `guard`. A
workflow run from a PR branch cannot deploy to any of them. So it never
sees the secrets, even if the PR edits a workflow.

### 6.3 The release bot (`<BOT_APP>`)

A GitHub App, installed only on the repository.

- Permissions: `contents: write`, `pull-requests: write`, `issues: write`
  (milestones), `statuses: write` (the `policy` status). Nothing else.
- No `workflows: write`. With it, a leaked key could push a workflow to
  `<DEV>` through the bump bypass and reach the `release` secrets. The cost is
  that the bot cannot push a sync that carries a workflow file change. A
  maintainer finishes that sync (E49).
- Jobs get a short-lived token with `actions/create-github-app-token`.
- It makes every write that changes release state: release and hotfix
  branches, their PRs, version commits, sync PRs, milestones, tags and
  releases.

Why not `GITHUB_TOKEN`:

- A PR that `GITHUB_TOKEN` opens starts its runs as `action_required`. A human
  must approve them. Pushes made with `GITHUB_TOKEN` start no workflows at all.
  An app token behaves like a normal user, so the bot's PRs and commits get CI
  like any other change.
- `GITHUB_TOKEN` cannot be a ruleset bypass actor. The app can.

`GITHUB_TOKEN` keeps the jobs that need no release write: reading, the preview
card (`pull-requests: write`) and SARIF upload (`security-events: write`).

### 6.4 Project profiles: app or web

Each repository picks one `<PROFILE>`.

| Concern | `app` | `web` |
|---|---|---|
| What ships | Installable files users download: desktop app, mobile app, browser extension, CLI. | A service you run: web app, API. |
| Who runs which version | Each user, on their own device. Old versions stay installed. | You. One version runs per environment. |
| Dev build (`<DEV>`) | A workflow artifact. No tag, no GitHub release (section 5.6). | Deploy to `<DEV_ENV>`. No tag, no GitHub release. |
| Pre-release (`release/**`, `hotfix/**`) | GitHub release marked pre-release, tag `<RC_TAG>`, files attached, for testers. | Deploy to `<STAGING_ENV>` (release branch) or `<HOTFIX_ENV>` (hotfix branch). No tag. |
| `<DEV_BUILD_MODE>` (recommended) | Manual: each build is a download someone asked for. | Automatic: `<DEV_ENV>` always runs the newest `<DEV>`. |
| `<RELEASE_RC_MODE>` / `<HOTFIX_RC_MODE>` (recommended) | Manual: testers start an RC when they want one. | Manual by default. Automatic where an env must follow every merge; the Hotfix env is small, so automatic costs little there. |
| Release (`<MAIN>`) | Tag `<REL_TAG>`, GitHub release with notes and files. | Deploy to `<PROD_ENV>`, then tag `<REL_TAG>` and a GitHub release with notes. The deployed artifact (or the image digest) is recorded on the release, so you can trace what runs in `<PROD_ENV>`. |
| PR preview (section 14) | Download link to the built files. | Download link to the built artifact, and a deploy to `<PREVIEW_ENV>`. |
| Provenance (section 9.6) | Attest the files. | Attest the artifact or the image digest. |
| A bad release | Users already installed it. Ship a hotfix. | Ship a hotfix. Rollback belongs to the hosting side behind `<DEPLOY_CMD>` (for example, the platform's own "redeploy previous version"), not to CI. |
| Secrets | The bot key (`release`, `bot` and `guard` environments). | The bot key, plus one deploy credential per environment. |
| Extra workflows | none | none |

In both profiles, a release always gets its tag and GitHub release. The version
rules and release notes depend on release tags.

**Web environments.** A `web` repository provides these five. The size is a
guide for the hosting side. Staging runs the next version, the Hotfix env runs
production plus its fixes, and production runs the current release. A Release
PR replaces what production runs; a Hotfix PR patches it.

| Environment | Runs | Size | Deployment branches | Secrets | Protection |
|---|---|---|---|---|---|
| `<PREVIEW_ENV>` | One isolated deploy per open PR into `<DEV>`, `release/**` or `hotfix/**`: one change | small, many | all branches (a PR run has no branch ref) | the preview deploy credential, which reaches only preview deploys | none |
| `<DEV_ENV>` | `<DEV>`: every merged feature | moderate | `<DEV>` | the dev deploy credential | none |
| `<STAGING_ENV>` | The Release PR head: the next version | low to moderate | `release/**` | the staging deploy credential | none |
| `<HOTFIX_ENV>` | The Hotfix PR head: production plus its fixes | low | `hotfix/**` | the hotfix deploy credential | none |
| `<PROD_ENV>` | `<MAIN>`: the current release | high | `<MAIN>` | the production deploy credential | optional required reviewer: a human approves each production deploy |

- Prefer short-lived cloud credentials through OIDC over a stored secret. The
  setup: `id-token: write` on the deploy job, and a cloud role that trusts
  only that environment.
- `<DEPLOY_CMD>` must be safe to run again with the same artifact. A rerun
  after a failure deploys the same version again.
- The deploy runs in its own job, in its environment. The build job never
  holds a deploy credential.

### 6.5 Repository settings

| Setting | Value | Why |
|---|---|---|
| Default branch | `<DEV>` | PRs target it by default. |
| Merge methods | Squash and merge commit on. Rebase off. | Rulesets choose per branch. |
| Delete branch on merge | On | GitHub deletes every merged head branch, bot branches too. An open PR into a deleted head moves to its base (E20). |
| Allow auto-merge | On | Sync PRs merge themselves once green (section 19). |
| Actions: workflow permissions | Read | Jobs raise what they need. |
| Actions: allow GitHub Actions to create and approve PRs | Off | The app opens PRs. Nothing approves. |
| Actions: require actions pinned to a full SHA | On | Turns I17 into a hard rule. |
| Actions: allowed actions | GitHub-owned + a listed set | A new third-party action is a deliberate choice. |
| Secret scanning + push protection | On | Scans the whole history and blocks a known secret before it lands (9.5). Free on public repositories; private ones need GitHub Secret Protection. |
| Private vulnerability reporting | On | A safe channel for reports. |

## 7. Workflow catalog

| File | Display name | Triggers | Purpose |
|---|---|---|---|
| `checks.yml` | Checks | `workflow_call` only | One definition of all checks: tests, guard, lint, SAST, workflow audit. |
| `build.yml` | Build | `workflow_call` only | One definition of the build: package, attestation, upload. |
| `ci.yml` | CI | push `<DEV>`, `<MAIN>`, `release/**`, `hotfix/**`; PR; weekly (new scanner rules on unchanged code) | Finds code changes, calls `checks.yml`, builds a dry-run package and the PR preview, posts the card, reports `gate`. |
| `codeql.yml` | CodeQL | push (same branches); PR (no path filter); weekly | CodeQL `security-extended` for `<CODEQL_LANG>` and `actions`. Rulesets require its results, except `main`. |
| `build-develop.yml` | Build develop | `workflow_dispatch`; `push` to `<DEV>` when `<DEV_BUILD_MODE>` is automatic | Dev build (section 5.6). Calls `checks.yml` and `build.yml`. |
| `pre-release.yml` | Create pre-release | `workflow_dispatch` on `release/**`, `hotfix/**`; `push` to each branch kind whose mode is automatic (`<RELEASE_RC_MODE>`, `<HOTFIX_RC_MODE>`) | Release candidate `<RC_TAG>`, or a deploy to `<STAGING_ENV>` or `<HOTFIX_ENV>`. Calls `checks.yml` and `build.yml`. |
| `promote-to-release.yml` | Promote to release | `workflow_dispatch` | The cut: `release/X.Y.0`, the `<DEV>` bump, the Release PR. No inputs. |
| `release.yml` | Release | push `<MAIN>`: a merged Release PR or Hotfix PR; `workflow_dispatch` for a retry | Checks, build, deploy, publish; milestone close, next Hotfix PR, sync PRs. |
| `create-hotfix.yml` | Create hotfix | `workflow_dispatch` | Opens a Hotfix PR when none is open, for example after a human closed one (section 18.5). |
| `pr-guard.yml` | PR guard | `pull_request_target` | Judges each PR with the default branch's code and posts its `policy` status. Also posts the status of the Release or Hotfix PR it depends on (section 21.1). |
| `dependabot.yml` | (config) | weekly | Grouped updates with a cooldown, for actions and each lockfile. |

Every workflow has `permissions: contents: read` at the top. Common to every
job:

1. `timeout-minutes` set (2 to 10 for checks, 15 for build).
2. `actions/checkout` with `persist-credentials: false`, without exception.
3. Every action pinned to a full SHA, with the tag in a comment.
4. Every job that writes has its own `if: github.ref == ...` guard (I5), even
   when it needs a guarded job.

## 8. Workflows in detail

### 8.1 `checks.yml`: one definition of "green"

A reusable workflow (`on: workflow_call`). PRs, dev builds, pre-releases,
Promote to release and releases all call it. So every build runs the same
checks on the exact commit it builds.

The caller grants `contents: read`, `security-events: write` and
`pull-requests: read`.

| Job | Runs | Notes |
|---|---|---|
| `test` | `<TEST_CMD>` | |
| `guard` | `<GUARD_CMD>` | Project rules a linter cannot see. Collects all problems, then fails. |
| `lint` | `<LINT_CMD>` | Security rules only, not style. Tools from `<TOOLS_DIR>` or the ecosystem's own analyzers. |
| `semgrep` | `<SAST_RULES>` blocking; registry packs informational | Exact Semgrep version. SARIF to the Security tab. |
| `workflows` | actionlint, then zizmor | Exact versions. Zero findings allowed; an ignore needs an inline reason. |

Semgrep has two tiers:

1. The repository's own rules, blocking (`--error`). Few rules, each proven
   clean on the code. Typical rules: no raw HTML sinks, no dynamic code
   execution, no disabled certificate checks.
2. Registry packs (`p/security-audit`, `p/secrets`, the language pack),
   informational. A false positive in a registry rule never blocks a PR.
   Promote a pack to blocking when it reports nothing on the code.

Taint questions ("does a secret reach a log?") belong to CodeQL, not to a
pattern rule. A pattern on argument text flags harmless strings.

### 8.2 `ci.yml`: the PR and branch gate

**Triggers:** section 7. No `paths-ignore`: the `changes` job decides instead
(section 10.4).

**Concurrency.** Section 10.1.

**Jobs.**

```mermaid
flowchart LR
  changes["changes<br/>code changed?"] --> checks
  changes --> package
  checks["checks<br/>uses checks.yml"] --> package
  package["package<br/>dry-run build<br/>PR: preview + card data"] --> canary
  canary["canary<br/>pull-requests: write<br/>no checkout"]
  changes --> gate
  checks --> gate
  package --> gate
  gate["gate<br/>if: always()<br/>the required check"]
  style changes fill:#1976d2,stroke:#0d47a1,stroke-width:2px,color:#fff
  style checks fill:#1976d2,stroke:#0d47a1,stroke-width:2px,color:#fff
  style package fill:#1976d2,stroke:#0d47a1,stroke-width:2px,color:#fff
  style canary fill:#f57c00,stroke:#e65100,stroke-width:2px,color:#fff
  style gate fill:#388e3c,stroke:#1b5e20,stroke-width:2px,color:#fff
```

- `changes`: a path filter (for example `dorny/paths-filter`). Output `code` is
  false when the PR changes only `**/*.md` or `docs/**`. On push and schedule
  events it is always true.
- `changes` also outputs the PR `kind`, read from the event as data:
  - `sync`: a sync PR. The head is `sync/*` in this repository.
  - `normal`: every other PR, and every push.
- `checks` and `package` run only when `code` is true. `checks` runs every
  job of `checks.yml` on every PR.
- `package` runs `<PACKAGE_CMD>` as a dry run, so a broken build appears on the
  PR, not on release day. It skips sync PRs. A PR into `<MAIN>` gets no
  preview: staging and the Hotfix env are its preview.
  - On push: uploads the artifact as a workflow artifact.
  - On a `normal` PR into `<DEV>`, `release/**` or `hotfix/**`: builds the
    preview (section 14), uploads it, then
    renders the PR card as Markdown. The card leaves the job as a base64
    output.
- `canary` holds the only write permission (`pull-requests: write`). It has no
  checkout and runs no file from the PR. It decodes the card and creates or
  updates one review comment. It runs only when `package` built a preview, and
  skips fork PRs and Dependabot PRs. The
  Dependabot check uses `github.event.pull_request.user.login`, not
  `github.actor`, because the actor changes when someone reruns the job.
  `canary` is not part of `gate`, so a failed card never blocks a merge.
- `policy` is not a job of `ci.yml`. `pr-guard.yml` posts it as a separate
  required status, with the default branch's code (section 21.1).
- `gate` runs always. It fails when any of `changes`, `checks`, `package`
  ended `failure` or `cancelled`. A job skipped by `if:` counts as
  passed (section 10.4).

**Sticky card.** `canary` looks for an earlier review by `github-actions[bot]`
whose body starts with the card heading. If it finds one, it updates it.
Otherwise it creates a `COMMENT` review. `set -euo pipefail` makes a failed
lookup stop the job, so an API error never posts a second card.

### 8.3 `codeql.yml`

- `security-extended` query suite. The weekly run catches new queries on
  unchanged code.
- A matrix over `<CODEQL_LANG>` and `actions`, one category per language. The
  `actions` language scans the workflows and composite actions, where the CI
  logic lives (section 2): script injection, untrusted checkouts, excessive
  permissions.
- For compiled languages, use `build-mode: none` where CodeQL supports it, or
  `autobuild`. Use a manual build only when those fail.
- Push trigger: `<DEV>`, `<MAIN>`, `release/**`, `hotfix/**`.
- No path filter. The `dev`, `release-branches` and `hotfix-branches`
  rulesets wait for code scanning results on every PR. So CodeQL must report
  on each of those PRs, docs-only ones too.
- CodeQL scans Release and Hotfix PRs like any PR. The `main` ruleset has no
  code scanning rule, so nothing waits for that result.

### 8.4 `build-develop.yml`: Build develop

**Concurrency.** Section 10.1.

**Job `version`** (`if: github.ref == 'refs/heads/<DEV>'`, 2 minutes):
`<READ_VERSION_CMD>`, then `next-version check --branch <DEV>` with all tags on stdin
(paginated). A bad version fails here in seconds, before the long checks.
Output: the version.

**Job `checks`** (needs `version`): calls `checks.yml`.

**Job `build`** (needs `version`, `checks`): calls `build.yml` (section 8.7)
with the version and the dev build label `<BUILD>`.

`build` needs no secret and no environment.

**Job `publish`**, by `<PROFILE>`:

- `app`: no `publish` job. `build.yml` already uploaded the files as a
  workflow artifact. The job summary names the version and the commit.
- `web` (needs `build`; `environment: <DEV_ENV>`): download the artifact,
  run `<DEPLOY_CMD>`. No tag, no GitHub release. The environment's
  deployment history shows each dev build.

### 8.5 `promote-to-release.yml`: Promote to release (the cut)

**Trigger.** Manual only. **Concurrency:** section 10.1.

**No branch choice.**

- No inputs.
- It always reads and cuts `<DEV>`. Its checkout names `<DEV>` explicitly.
- The GitHub run form always shows "Use workflow from". A run from any other
  ref stops at the job guard (`if: github.ref == 'refs/heads/<DEV>'`). The
  `release` environment refuses that ref too. So a workflow file changed on a
  feature branch never reaches the bot key.

**Steps.**

1. **`version` job.** `next-version check --branch <DEV>`. Refuse when:
   - a Release PR is open;
   - a sync PR into `<DEV>` is open, draft or not (the previous merge is not
     back in `<DEV>` yet);
   - a Hotfix PR has work (section 18.3): a human ships it, or closes those
     PRs. An idle Hotfix PR does not block;
   - a `release/*` branch exists (an abandoned one: E3).
2. **`checks` job.** Calls `checks.yml` on the head of `<DEV>` (commit `C`).
3. **`cut` job** (`environment: release`, app token):
   1. If the head of `<DEV>` is no longer `C`, stop red before any push.
      Dispatch the workflow again (E4).
   2. Locally, create `release/X.Y.0` at `C`. Merge the head of `<MAIN>`
      into it and keep the release branch's `VERSION` (19.4 says why). On a
      conflict in any other file, drop the merge. The branch stays at `C`,
      and step 3.5 opens a sync PR so a human decides (19.5, cut case).
   3. Push `release/X.Y.0` once, in its final state.
   4. On `<DEV>`, one bot commit: `VERSION` = `X.(Y+1).0`. Push.
   5. If step 3.2 dropped the merge, open the cut-case sync PR (19.5). The
      Release PR conflicts the same way, so GitHub blocks it until that sync
      PR merges.
   6. Open PR "Release X.Y.0" from `release/X.Y.0` into `<MAIN>`. Normal PR,
      not a draft.

The release flow. Blue: the trigger. Orange: a human step. Green: the
bot.

```mermaid
flowchart TD
  trigger(["A maintainer runs Promote to release (manual)"]) --> cut["Cut: release/X.Y.0 from DEV<br/>DEV moves to X.(Y+1).0"]
  cut --> rpr["Release PR into MAIN"]
  rpr --> fixes["Stacked fix PRs and pre-releases"]
  fixes --> merge["A maintainer merges the Release PR"]
  merge --> publish["Publish vX.Y.0"]
  merge --> hotfix["Open Hotfix X.Y.1"]
  merge --> sync["Sync PR into DEV"]
  style trigger fill:#1976d2,stroke:#0d47a1,stroke-width:2px,color:#fff
  style fixes fill:#f57c00,stroke:#e65100,stroke-width:2px,color:#fff
  style merge fill:#f57c00,stroke:#e65100,stroke-width:2px,color:#fff
  style cut fill:#388e3c,stroke:#1b5e20,stroke-width:2px,color:#fff
  style rpr fill:#388e3c,stroke:#1b5e20,stroke-width:2px,color:#fff
  style publish fill:#388e3c,stroke:#1b5e20,stroke-width:2px,color:#fff
  style hotfix fill:#388e3c,stroke:#1b5e20,stroke-width:2px,color:#fff
  style sync fill:#388e3c,stroke:#1b5e20,stroke-width:2px,color:#fff
```

The Release PR runs CI like any PR, because the app opened it (section 6.3).

### 8.6 `pre-release.yml`: Create pre-release

**Trigger.** Manual, on `release/**` or `hotfix/**` only (job guard and the
environments' branch policies). Also `push` to `release/**` when
`<RELEASE_RC_MODE>` is `automatic`, and to `hotfix/**` when `<HOTFIX_RC_MODE>`
is (section 5.5). Then staging follows the Release PR head, and the Hotfix env
follows the Hotfix PR head.

**Concurrency.** Section 10.1.

- `version` job: `next-version check --branch <ref>`.
- `checks` job: calls `checks.yml`.
- `build` job: calls `build.yml` (section 8.7) with the version and the
  pre-release label.
- `app`: `publish` (`environment: bot`) creates the GitHub release marked
  pre-release, tag `<RC_TAG>`, with GitHub generated notes since the last
  release tag (section 13). Old pre-releases stay. Delete them by hand from
  the Releases page when there are too many.
- `web`: `deploy` runs `<DEPLOY_CMD>`, with `environment: <STAGING_ENV>` on a
  release branch and `environment: <HOTFIX_ENV>` on a hotfix branch. No tag.

### 8.7 `build.yml`: one definition of the build

A reusable workflow. Build develop, Create pre-release and Release call it, so
the build is defined once.

**Inputs:** the version, an optional build label (dev build or pre-release).
**Permissions:** `contents: read`, `id-token: write`, `attestations: write`.

1. `<PACKAGE_CMD> <version> [label]`. The artifact carries the version. The
   working tree stays clean.
2. `actions/attest-build-provenance` on the release files.
3. Stage files flat in `assets/` and upload them as one workflow artifact.

### 8.8 `release.yml`: Release

**Concurrency.** Section 10.1.

**Job `version`** (`if: github.ref == 'refs/heads/<MAIN>'`): read `VERSION`
at the merge commit, apply the rule of the merged branch kind (section 5.4),
then refuse to re-release: it fails if tag `<REL_TAG>` exists. It takes
seconds.

**Follow-up jobs** (each needs `version`, each checks `github.ref` itself, all
in parallel with the build):

- `milestone-close`: closes milestone `X.Y.Z` (section 20.1).
- `next-hotfix`: opens the next Hotfix PR (section 18.2).
- `sync-develop`: the sync PR into `<DEV>` (section 19.2).
- `sync-release`: for a Hotfix PR merge while a release branch is open, the
  sync PR into it (section 19.5).

They do not wait for build, publish or deploy. Each is safe to rerun.

```mermaid
flowchart LR
  merge(["merge into MAIN"]) --> version
  version --> close["milestone-close"]
  version --> next["next-hotfix"]
  version --> syncdev["sync-develop"]
  version --> sync["sync-release<br/>(hotfix, release branch open)"]
  version --> checks --> build --> deploy["deploy (web)"] --> publish
  style publish fill:#d32f2f,stroke:#b71c1c,stroke-width:2px,color:#fff
  style close fill:#388e3c,stroke:#1b5e20,stroke-width:2px,color:#fff
  style next fill:#388e3c,stroke:#1b5e20,stroke-width:2px,color:#fff
  style syncdev fill:#388e3c,stroke:#1b5e20,stroke-width:2px,color:#fff
  style sync fill:#388e3c,stroke:#1b5e20,stroke-width:2px,color:#fff
```

**Job `checks`** (needs `version`): calls `checks.yml`.

**Job `build`** (needs `version`, `checks`): calls `build.yml` (section 8.7)
with `<BASE_VERSION>`.

**Job `deploy`** (`web` only; needs `build`;
`environment: <PROD_ENV>`): download the artifact, run `<DEPLOY_CMD>`. A
required reviewer on `<PROD_ENV>` makes a human approve it first.

**Job `publish`** (needs `build`, and `deploy` when it exists;
`environment: release`):

- Downloads the artifact. App token. `gh release create <REL_TAG> ...
  --target <sha> --generate-notes --notes-start-tag <previous vX.Y.Z>`
  (section 13). `app`: with the files. `web`: with the deployed artifact (or the image
  digest in the notes).
- `gh release create` is not atomic: a failed upload can leave a draft
  release. A rerun resumes a draft whose tag target is the merge commit
  (upload the missing files, then publish it). It refuses only a published
  release or a draft with another target.
- This is the first moment a tag exists. Any failure before this leaves no
  tag, so a rerun is clean. For `web`, a failed `publish` after a good
  deploy reruns alone: the version is live, and the tag records it.

The cut bumps `<DEV>` (section 8.5); nothing bumps it after publish.

### 8.9 `dependabot.yml`

- `github-actions`, weekly, commit prefix `ci`.
- One entry per lockfile: `<TOOLS_DIR>` and every lockfile of the product.
- Every entry groups its minor and patch updates into one PR. Each major
  update gets its own PR, so a breaking change is reviewed alone.
- `cooldown: default-days: 7` on every entry. A new upstream version waits a
  week before Dependabot proposes it. Most malicious releases are found and
  removed within days.

## 9. Security model

### 9.1 Token permissions

- Top level of every workflow: `permissions: contents: read`.
- Each job raises only what it needs.
- Release writes use the app token (section 6.3), never a raised
  `GITHUB_TOKEN`.

### 9.2 No credentials on disk

`actions/checkout` stores the token in `.git/config` by default. Every
checkout sets `persist-credentials: false`, push jobs too. Code that runs
later (tests, PR code, third-party tools, uploaded artifacts) cannot read a
token. A step that must push builds the remote URL from the token in that one
step.

### 9.3 Untrusted code vs write token: the split job

Code from a PR and a write token never meet in one job.

```mermaid
sequenceDiagram
  participant PR as PR code
  participant P as package job (read-only token)
  participant C as canary job (pull-requests: write)
  participant GH as GitHub API
  PR->>P: checkout (no credentials stored)
  P->>P: build preview, render card (github-script)
  P->>C: card as base64 job output (data only)
  Note over C: no checkout, runs no PR file
  C->>GH: find old card, update or create review
```

The write token exists only in `canary`, and `canary` runs no code from the
PR. It treats the card as data: decode, then send as a file body.
PR-controlled text reaches steps only through `env:`. It never appears as
`${{ }}` inside `run:` or a github-script `script:`. This blocks script
injection.

This gives the safety of the two-workflow `workflow_run` pattern in one file.
It stays safe only while the write job never checks out PR code. Never use
`pull_request_target` for anything that checks out PR code.

`pr-guard.yml` uses `pull_request_target` on purpose. It needs the bot key to
post `policy`, and a PR must not change the rules that judge it. It checks out
only its own workflow commit (`github.workflow_sha`, the default branch;
sparse, `.github/actions`). It reads PR data through the API and runs no file
from the PR. zizmor reports `dangerous-triggers`; an inline ignore gives the
reason.

### 9.4 Supply chain

- Every action pinned to a full SHA, tag in a comment. The repository setting
  rejects any other pin.
- CI-only tools come from `<TOOLS_DIR>` with `<LOCKED_INSTALL>`, or from an
  exact version (`semgrep==<x.y.z>`). Never a range, never `latest`.
- Install scripts stay off wherever the ecosystem allows it.
- Dependabot updates all of them after a 7-day cooldown (section 8.9).
- zizmor's `adhoc-packages` audit fails a workflow that installs a package
  outside a lockfile.
- Keep the product's own dependencies few. Each one is code you ship.

### 9.5 Secret scanning

- GitHub secret scanning covers the whole history and every new push. Push
  protection blocks a known secret before it lands. No CI job scans for
  secrets.
- Free on public repositories. A private repository needs GitHub Secret
  Protection; without it, the setup adds a secret scanner job to `checks.yml`.
- Close a false positive, such as a fake token in a test, as an alert with
  its reason. Never hide it with a folder-wide rule.

### 9.6 Provenance

- GitHub shows a SHA-256 digest for every release file, so CI writes no
  checksum file.
- `actions/attest-build-provenance` records that this workflow, at this
  commit, built these files.
- Release notes tell users how to verify:
  `gh attestation verify <file> --repo <owner>/<repo>`.
- For a container image, attest the image digest instead
  (`subject-name` + `subject-digest`, `push-to-registry: true`).

### 9.7 Bot commits

All by `<BOT_APP>`:

| Commit | Branch | From |
|---|---|---|
| `chore: version X.(Y+1).0` | `<DEV>` | Promote to release |
| The merge of `<MAIN>` (dropped on a conflict) | `release/X.Y.0` | Promote to release |
| `chore: version X.Y.Z` | `hotfix/X.Y.Z` | `next-hotfix`, `create-hotfix.yml` |

Sync PRs are bot PRs, not direct commits: they pass `gate` like any PR.

## 10. Reliability model

### 10.1 Concurrency

GitHub keeps one running and one waiting run per group; a newer waiting run
replaces the older one.

| Workflow | Group | Cancel in progress | Why |
|---|---|---|---|
| CI, CodeQL | `<name>-${{ github.ref }}` | PRs only | Stale PR runs stop; every branch commit keeps a result. |
| Build develop | `build-develop` | no | Never stop a publish halfway. One build for a burst of merges (section 5.6). |
| Create pre-release | `pre-release-${{ github.ref }}` | no | Same, per branch: a hotfix run never replaces a waiting release run. |
| Promote to release | `promote-to-release` | no | Same. |
| Release | `release` | no | A second merge's run waits for the first. Merges into `<MAIN>` come minutes apart, so one waiting run is enough. |
| PR guard | one per PR | no | Every change must reach the parent. |

### 10.2 Idempotency and retry

| Failure point | State left behind | Recovery |
|---|---|---|
| `version`, `checks` or `build` fails | No tag, no release | Rerun. |
| `deploy` fails (`web`) | No tag; production may run the old or a partly rolled-out version | Fix, then rerun failed jobs. `<DEPLOY_CMD>` redeploys the same artifact. |
| `publish` fails | No tag, or a draft release | Rerun failed jobs. It resumes a matching draft. |
| A follow-up job fails (milestone, next Hotfix PR, sync) | Release is out | Rerun that job. Each one reuses or skips what exists (sections 18.2, 19.2, 19.5, 20.1). |
| Promote to release stops (`<DEV>` moved) | Nothing pushed | Dispatch it again (E4). |
| Promote to release stops after a push | A release branch, maybe the bump, no Release PR | Bump pushed: open the Release PR by hand. No bump: delete the branch and dispatch again. |
| Release workflow runs again on a released version | Blocked | `version` job; tag ruleset. |

### 10.3 Fail loud, never silent

- `set -euo pipefail` in every multi-line shell step.
- `timeout-minutes` on every job. A hung tool fails fast and frees its
  concurrency group.
- `if-no-files-found: error` on every upload that must have a file.
- Lookups that decide "create or update" fail on an API error. They never read
  an error as "not found".
- Every list call paginates (`--paginate`). A fixed limit silently drops items
  as the repository grows.
- Steps print GitHub `::error::` lines (`core.setFailed` in github-script), so
  the reason shows on the run page.
- Cheap checks run first: the `version` job fails in seconds, before the long
  checks.

### 10.4 Docs-only PRs and the single required check

A workflow skipped by `paths-ignore` reports no status. If a ruleset requires
one of its checks, a docs-only PR waits forever ("Expected - waiting for
status"). So:

- No workflow uses `paths-ignore` on PRs.
- In `ci.yml`, the `changes` job decides. Jobs skip by `if:` when no code
  changed. A job skipped by `if:` counts as passed.
- `gate` (`if: always()`, needs all real jobs) is the only required status
  check from `ci.yml`. (`policy` is the other one; `pr-guard.yml` posts it on
  every PR, docs-only ones too.) It fails when a needed job failed or was cancelled.
- To add a new job to the gate, add it to `gate.needs`. The ruleset never
  changes.
- GitHub states it: "A job that is skipped will report its status as
  'Success'. It will not prevent a pull request from merging, even if it is a
  required check." The same docs say a workflow skipped by a path filter, a
  branch filter or a commit message stays "Pending".
- So a commit message with `[skip ci]` on a PR skips the whole workflow, and
  `gate` waits forever. Never use it on PR commits. Push a new commit to
  recover.

CodeQL is required through the ruleset's code scanning rule, not as a status
check (section 8.3).

## 11. Shared steps and their contracts

Each entry is a composite action, a reusable workflow, or a named step (rule in
section 2). It reads action inputs, `env:` and the API, writes step outputs, and
fails with an `::error::` line. Decisions run in github-script; git and gh
commands run in bash.

| Step | Input | Output | Fails when |
|---|---|---|---|
| `version` (shared module) | a version string | true/false | n/a. Validate with a numeric round-trip, not only a regex: `$` in many regex engines also matches before a trailing newline. |
| `next-version check --branch <ref>` | `VERSION` at the ref and at the head of `<MAIN>`; tags and open release branches from the API | the version | a rule of section 5.4 breaks. Pre-release tags are ignored. |
| `next-version next <v>` | arg | `X.(Y+1).0` | `<v>` invalid. |
| `next-version next-hotfix <v>` | arg | `X.Y.(Z+1)` | `<v>` invalid. |
| `canary-card` | env: label, sha, base, artifact URL, retention | Markdown card | never. A missing URL gives a "no download" card. |
| `close-plan` | `V`, the PRs for the commits in `M^1..M`, the issues they close, milestone `V` if it exists | ordered actions: create, set, close | see section 12. |
| `policy` | base, head, changed files (with the full count), PR author, open stacked PRs, Hotfix PR state, commits on the head, tags for a pin bump | broken rules | see section 12. |
| `sync-plan` | merge kind, first-parent commits in range with their changed paths, parents and PR head ref | ordered cherry-picks, per section 19.2 steps 1 and 3 | see section 12. |
| `merge-resolve` | conflicted paths | `commit`, `keep-target-version` or `manual` | see section 12. |
| `<PACKAGE_CMD>` | version, optional dev build number or preview label | artifact files in `dist/` | the build fails; invalid version or label. |
| `<GUARD_CMD>` | the tree | `ok`, or a list of all problems | any project invariant breaks. |

Shared constants (branch names, the bot login) are repository variables.
Workflows read them through `vars.*`, so every workflow sees one value.

## 12. Testing the CI itself

The pipeline has tests too.

1. **Lab scenarios.** A lab repository (a throwaway copy of the template)
   proves each step in section 11. It uses the scenario tables below:
   version edge cases (leading zeros, part limits, trailing newline), tag
   order, card text with and without a URL. Rerun them after a change to a
   workflow or shared step.
2. **Invariant tests.** Tests that read the source tree and assert rules a
   linter cannot see: layering between generic and domain code, forbidden
   APIs, required security settings.
3. **The guard.** A job in `checks.yml`.
4. **Workflow audit.** actionlint (expressions, contexts, shellcheck on
   `run:`) and zizmor (template injection, permissions, credential
   persistence, unpinned actions, ad-hoc installs, dangerous triggers, bot
   conditions).
5. **Dry-run package on every PR**, so build breaks appear early.

Tests that use fixtures with fixed dates also fix the clock.

**Scenarios for the release steps.** One lab run per row, literal inputs,
expected outputs.

| Function | Scenario | Expected |
|---|---|---|
| `next-version check` | `<DEV>` `1.5.0`, tags `v1.4.0`, open `release/1.5.0` | fail: not above the open release branch |
| | `<DEV>` `1.6.0`, open `release/1.5.0` | ok |
| | `<DEV>` `1.6.1` | fail: Z must be 0 |
| | `release/1.5.0`, `VERSION` `1.5.0`, tags `v1.4.2` | ok |
| | `release/1.5.0`, `VERSION` `1.6.0` | fail: branch name mismatch |
| | `hotfix/1.4.3`, `<MAIN>` at `1.4.2`, tags `v1.4.0 v1.4.2` | ok |
| | `hotfix/1.4.3`, `<MAIN>` at `1.5.0`, tags include `v1.5.0` | fail: line `1.4` is not the latest |
| | `hotfix/1.4.3`, `<MAIN>` at `1.4.2`, tags include `v1.5.0-rc.3` | ok: rc tags do not count |
| | `hotfix/1.5.1`, `<MAIN>` at `1.5.0`, no tag `v1.5.0` (it never shipped, E43) | ok |
| `close-plan` | PR for a commit in `M^1..M`, no milestone | set `1.5.0` |
| | PR for a commit in `M^1..M`, closed milestone `1.4.2` | keep |
| | issue closed by such a PR | set `1.5.0` |
| | sync PR for a commit in `M^1..M` | skip |
| | milestone `1.5.0` missing | create, set, close |
| | run twice on the same input | second run plans nothing |
| `policy` | PR into `<MAIN>` from `feature/x` | P1 |
| | PR into `<DEV>` changes `VERSION` and `src/a` | P2 |
| | PR into `release/1.5.0` changes `VERSION` | P3 |
| | Release PR while the Hotfix PR is idle (no fix, no open PR into its branch) | pass |
| | Release PR while the Hotfix PR has a fix | P5 |
| | Release PR while an open PR targets the hotfix branch | P5 |
| | Hotfix PR with only the bot's version commit | P7 |
| | sync PR into `<DEV>` from the bot, no `VERSION` change | pass |
| | Release PR while a PR into `release/1.5.0` is open | P8 |
| | fork PR into `<MAIN>`, head named `hotfix/9.9.9` | P1; does not count for P5 |
| `sync-plan` | hotfix: bot `VERSION` commit, fix commit (`src/a`) | skip, pick |
| | release: old hotfix commit reachable only through the cut's merge | not in the range (first parent) |
| | release: GitHub merge commit of a PR with head `sync/release/1.4.3` | skip |
| | release: stacked PR merged with a merge commit | pick against first parent |
| | fix commit changes `src/a` and `VERSION` | pick without `VERSION` |
| | release: range holds the cut's merge of `<MAIN>` | skip the merge |
| `merge-resolve` | none | commit |
| | `VERSION` only | keep-target-version |
| | `VERSION` and `src/a` | manual |

## 13. Release notes

Every release, and every `app` pre-release, gets GitHub generated notes:
`gh release create ... --generate-notes --notes-start-tag <previous vX.Y.Z>`.
The Releases page is the changelog. There is no `CHANGELOG.md` file.

- `.github/release.yml` groups the PRs by label (for example Breaking changes,
  Features, Bug fixes, Other). It excludes the author `<BOT_APP>[bot]`, so
  sync PRs, Release PRs and Hotfix PRs do not appear in the notes. Check this
  exclusion on the first dry run.
- The job passes the previous release tag explicitly, pre-release tags
  excluded. GitHub's default "previous tag" can be an rc tag and give short
  notes.
- Pre-release notes list every PR since the last release, not only those
  since the last rc.
- Good notes need good PR titles and labels. Reviewers check both.

## 14. PR preview builds

Each same-repository PR gets a preview when `checks` pass, except PRs into
`<MAIN>` and sync PRs. Staging and the Hotfix env are the previews of the first
two, and a sync PR only merges back.

- Label: `<BASE_VERSION>-PR-<number>-<run_number>`. A PR into `release/**` or
  `hotfix/**` builds a release candidate of that version, so its label is
  `<BASE_VERSION>-rc.PR-<number>-<run_number>`. The job checks the label
  against a plain pattern, because the label becomes a file name or a tag.
- Built with no secret.
- Kept 14 days.
- One card per PR, updated on each build: the commit, the base it was merged
  onto, a download link or URL, the expiry, how to try it.
- The preview must not clash with a real install or deploy: its own name,
  version label, tag or URL.

| Profile | Preview |
|---|---|
| `app` | Builds per OS as workflow artifacts, labeled so they install next to the real app (own ID, name or folder). |
| `web` | The built artifact as a workflow artifact, to run locally, and a deploy to `<PREVIEW_ENV>` (URL in the card). The deploy runs in a separate job like `canary`, with the preview credential, never in the build job. |

## 15. Setup checklist for a new repository

1. Create `<DEV>` and `<MAIN>`. Set `<DEV>` as default.
2. Apply the settings in section 6.5.
3. Create `<BOT_APP>` (section 6.3) and install it on the repository only.
4. Create the `release`, `bot` and `guard` environments (section 6.2). Add
   `BOT_APP_KEY` and `BOT_APP_ID` to each.
5. Add `VERSION` with the first version, for example `0.1.0`, and
   `.gitattributes` with `VERSION text eol=lf` (section 5.3).
6. Add the shared steps from section 11 (composite actions and reusable workflows).
7. Add `<GUARD_CMD>` with the project's own invariants.
8. Make `<PACKAGE_CMD>` take the version as input and ship only declared
   content.
9. Add `<TOOLS_DIR>` with a lockfile.
10. Add `.github/release.yml` with the label groups and the bot excluded
    (section 13).
11. Add `<SAST_RULES>` with a few rules proven clean on the code.
12. Add the workflows (section 7). Pin each action by SHA. Run actionlint and
    zizmor locally until both are clean. On a new repository, check that
    Actions is on (Settings, Actions) before the first push. Until then, no
    push starts a run.
13. Add `dependabot.yml` with the cooldown.
14. Choose `<PROFILE>` (section 6.4) and the build options:
    `<DEV_BUILD_MODE>`, `<RELEASE_RC_MODE>` and
    `<HOTFIX_RC_MODE>` (sections 5.5, 5.6). For `web`, create `<PREVIEW_ENV>`,
    `<DEV_ENV>`, `<STAGING_ENV>`, `<HOTFIX_ENV>` and `<PROD_ENV>` with their
    branch policies and credentials (section 6.4).
15. Add the rulesets (section 6.1). Require `gate`, and `policy` from
    `<BOT_APP>` only, plus code scanning on every branch but `<MAIN>`. No
    required approvals.
16. Dry run, in order:
    - Open a docs-only PR: `gate` passes, nothing else runs.
    - Open a code PR: checks run, the preview card appears.
    - Merge. Run Build develop. Check the artifact and
      `gh attestation verify`.
    - Then the release and hotfix flow, in order:
      1. Cut `0.1.0`. Check: `release/0.1.0` exists, `<DEV>` at `0.2.0`,
         Release PR open with every check and CodeQL.
      2. Open a stacked PR into `release/0.1.0`. Check: Release PR red (P8). Merge
         the stacked PR. Check: Release PR green.
      3. Merge the Release PR. Check: `v0.1.0` with generated notes, milestone
         `0.1.0` closed with the shipped PRs, `release/0.1.0` deleted, Hotfix
         `0.1.1` open and red (idle), sync PR into `<DEV>` squash merged by
         auto-merge, `<DEV>` has the stacked fix and still `0.2.0`.
      4. Open a fix PR into `hotfix/0.1.1`. Check: Promote to release refuses.
         Merge the fix, then the Hotfix PR. Check: `v0.1.1`, milestone closed,
         Hotfix `0.1.2` open and idle, the sync PR kept `<DEV>` at `0.2.0`.
      5. Automatic mode: merge two PRs quickly into `<DEV>`. Check: one dev
         build with both.
      6. With Hotfix `0.1.2` still idle, cut `0.2.0`. Check: the cut runs,
         `release/0.2.0` contains the head of `<MAIN>`, and the Release PR has no
         conflict and is green.
      7. Merge Release `0.2.0`. Check: the bot closes Hotfix `0.1.2` with its
         comment and opens Hotfix `0.2.1`.
17. Never rename `build-develop.yml` or `pre-release.yml` after the first run.

## 16. Stack examples

Values for common stacks. The design does not change between them.

| Placeholder | ASP.NET Core / .NET | Node | Python | Go |
|---|---|---|---|---|
| How the build reads `VERSION` | section 5.3 | section 5.3 | section 5.3 | section 5.3 |
| `<TOOLCHAIN_SETUP>` | `actions/setup-dotnet` | `actions/setup-node` | `actions/setup-python` | `actions/setup-go` |
| `<TEST_CMD>` | `dotnet test` | `npm test` | `pytest` | `go test ./...` |
| `<LOCKED_INSTALL>` | `dotnet restore --locked-mode` (with `packages.lock.json`) | `npm ci --ignore-scripts` | `pip install --require-hashes -r requirements.txt` | `go mod download` (checks `go.sum`) |
| `<LINT_CMD>` | .NET analyzers with `AnalysisMode=All` for security rules, warnings as errors | ESLint security plugins | `bandit` | `gosec` |
| Typical `<PROFILE>` | `web` (ASP.NET Core app or API); `app` (desktop app, CLI tool) | `web` (web app, API); `app` (Electron app, browser extension, CLI) | `web` (web app, API); `app` (CLI tool) | `app` (CLI binaries); `web` (service) |
| `<PACKAGE_CMD>`, `web` | `dotnet publish -c Release -p:Version=<v>`, then a zip or a container image | bundler output, then a zip or a container image | a container image | a container image |
| `<PACKAGE_CMD>`, `app` | `dotnet publish -c Release -r <rid> -p:Version=<v>` per OS, then a zip or an installer | Electron or extension packager, per OS | a single-file build tool per OS | `go build -ldflags "-X main.version=<v>"` per OS |
| `<DEPLOY_CMD>` (`web`) | push the image and roll out the service, or deploy the publish output to the host | same idea | same | same |
| `<DEV_VERSION>` | `X.Y.Z-dev.N` | `X.Y.Z-dev.N` | `X.Y.Z.devN` | `X.Y.Z-dev.N` |
| `<RC_TAG>` | `vX.Y.Z-rc.N` | `vX.Y.Z-rc.N` | `vX.Y.ZrcN` | `vX.Y.Z-rc.N` |
| `<CODEQL_LANG>` | `csharp` | `javascript-typescript` | `python` | `go` |

## 17. Threats and controls

| Threat | Control |
|---|---|
| A PR edits a workflow to read a secret. | Secrets live only in the `release`, `bot` and `guard` environments and the deploy environments; PR branches cannot deploy to any of them (6.2, 6.4). |
| A PR's code reads the token from the checkout. | `persist-credentials: false` everywhere (9.2). |
| PR-controlled text injects shell commands. | `env:` only, never `${{ }}` in `run:`; zizmor `template-injection` (9.3, 12). |
| A write job runs PR code. | Split job: the write job has no checkout; `pr-guard.yml` checks out only its own workflow commit (9.3). |
| A third-party action is changed under its tag. | SHA pins, enforced by repository setting (9.4). |
| A fresh malicious version of a tool or action. | Lockfiles and exact versions; Dependabot cooldown (8.9, 9.4). |
| An install script runs in a job with a secret. | Locked installs with scripts off; secret jobs install nothing they do not need (9.4). |
| A merge skips a red check. | Rulesets require `gate` and the bot's `policy`, and code scanning on every branch but `<MAIN>` (6.1). |
| A PR changes the CI code that judges or releases it. | `pr-guard.yml` runs the default branch's code; P10 sends CI changes to an admin (section 21). |
| A build ships a commit that never passed lint or security. | Every build calls `checks.yml` (8.1). |
| A human pushes or tags directly. | Branch and tag rulesets; only the bot and admins bypass (6.1). |
| A release ships twice. | `version` job; tag ruleset (8.8). |
| A user installs a tampered file. | GitHub's file digest and provenance attestation (9.6). |
| A deploy credential leaks to PR code or a build step. | Credential only in `<PREVIEW_ENV>` / `<DEV_ENV>` / `<STAGING_ENV>` / `<HOTFIX_ENV>` / `<PROD_ENV>`, each bound to its branches (the preview one reaches only preview deploys); only the deploy job uses it; OIDC instead of stored secrets (6.4). |
| An unreviewed production deploy. | Optional required reviewer on `<PROD_ENV>` (6.4). |
| A committed secret. | GitHub secret scanning and push protection (9.5). |
| A hung job blocks releases. | `timeout-minutes` everywhere (10.3). |
| A fork PR named `release/*` or `hotfix/*` ships fork code, or blocks a release. | P1 and P5 count only bot PRs from this repository (section 21). |
| A leaked bot key rewrites release history or adds a workflow. | `no-force-push` ruleset the bot cannot bypass; no `workflows: write` (sections 6.1, 6.3). |
| A hotfix is lost, or comes back to `<DEV>` only partly. | Sync PR after every merge; a half sync stays a draft; an unsynced hotfix makes the Release PR conflict (sections 19.2, 19.5). |
| A release ships while a fix waits in a Hotfix PR. | P5 and the cut (section 18.4). |

## 18. Release branches and hotfixes

### 18.1 Life of a release branch

```mermaid
sequenceDiagram
  participant Dev as Developer
  participant RB as release/X.Y.0
  participant RP as Release PR
  Dev->>RB: stacked fix PR (squash merge)
  RB->>RP: new head, pre-release per RELEASE_RC_MODE
  RP->>RP: policy: no open stacked PRs (P8)
  Dev->>RP: merge with a merge commit
```

- Stacked fix PRs target `release/X.Y.0`. Each one needs a green `gate`.
  Squash merge.
- Pre-releases come from this branch, per `<RELEASE_RC_MODE>` (section 5.5).
- The Release PR is green when P5 and P8 pass (section 21). `pr-guard.yml`
  keeps it current when other PRs change (section 21.1).
- Merge the Release PR with a merge commit. `release.yml` publishes `vX.Y.0`.

### 18.2 Hotfix steps

The hotfix flow. Colors as in section 8.5.

```mermaid
flowchart TD
  autoT(["A Release PR or Hotfix PR merges (automatic)"]) --> open
  manualT(["A maintainer runs Create hotfix (manual)"]) --> open
  open["Hotfix PR X.Y.(Z+1) opens, idle"] --> bug{"A bug in X.Y.Z?"}
  bug -- no --> closed["Closed at the next release"]
  bug -- yes --> fix["Fix PR into the hotfix branch<br/>and pre-releases"]
  fix --> merge["A maintainer merges the Hotfix PR"]
  merge --> publish["Publish vX.Y.(Z+1)"]
  merge --> sync["Sync PRs into DEV<br/>and the open release branch"]
  merge --> next["Open Hotfix X.Y.(Z+2)"]
  next -.-> open
  style autoT fill:#1976d2,stroke:#0d47a1,stroke-width:2px,color:#fff
  style manualT fill:#1976d2,stroke:#0d47a1,stroke-width:2px,color:#fff
  style fix fill:#f57c00,stroke:#e65100,stroke-width:2px,color:#fff
  style merge fill:#f57c00,stroke:#e65100,stroke-width:2px,color:#fff
  style open fill:#388e3c,stroke:#1b5e20,stroke-width:2px,color:#fff
  style closed fill:#388e3c,stroke:#1b5e20,stroke-width:2px,color:#fff
  style publish fill:#388e3c,stroke:#1b5e20,stroke-width:2px,color:#fff
  style sync fill:#388e3c,stroke:#1b5e20,stroke-width:2px,color:#fff
  style next fill:#388e3c,stroke:#1b5e20,stroke-width:2px,color:#fff
```

1. **A Release PR or Hotfix PR merges as `X.Y.Z`.** Job `next-hotfix` runs at
   the merge (app token):
   - Creates `hotfix/X.Y.(Z+1)` from the merge commit (where tag `vX.Y.Z`
     goes).
   - Commits `VERSION` = `X.Y.(Z+1)` on it as the bot.
   - Opens PR "Hotfix X.Y.(Z+1)" into `<MAIN>`. Normal PR, not a draft. Its
     `gate` is red while it has no fix (rule P7), so nobody can ship an empty
     patch. Its body:

     ```markdown
     This PR is for hotfixes to the released version X.Y.Z.

     To ship a fix, open a fix PR into `hotfix/X.Y.(Z+1)`. Merge it, then merge
     this PR.

     No hotfix needed? You can ignore this PR. While it has no fix and no open
     PR into its branch, it does not block the next release. When release
     X.(Y+1).0 merges, the bot closes this PR.

     Once it has a fix, or an open PR into its branch, the Release PR stays red
     until this PR merges or those PRs are closed.
     ```
   - At a release merge (`X.(Y+1).0`), closes the idle Hotfix PR of the old line
     `X.Y.*`, with the comment "No hotfix for `X.Y.(Z+1)`. Closed because
     `X.(Y+1).0` is released.", and deletes its branch.
     It never closes a Hotfix PR that has work (section 18.4).
   - A rerun is safe:
     - an open Hotfix PR for `X.Y.(Z+1)` exists: reuse it;
     - a Hotfix PR for `X.Y.(Z+1)` was closed or merged: stop (a human closed
       it on purpose, or it already shipped);
     - the head of `<MAIN>` is no longer this merge commit: stop (a newer
       merge owns the next Hotfix PR).
   - Only `create-hotfix.yml` recreates a Hotfix PR that a human closed
     (section 18.5).
2. **A fix.** The fix PR targets `hotfix/X.Y.(Z+1)` and needs a green
   `gate`. Squash merge.
3. **Try it.** Create pre-release builds the hotfix branch. It runs on each
   push when `<HOTFIX_RC_MODE>` is `automatic`, or when someone runs it. It
   gives `vX.Y.(Z+1)-rc.<BUILD>`, and for `web` a deploy to `<HOTFIX_ENV>`.
4. **Ship it.** The Hotfix PR is green when P7 and P8 pass (section 21).
   Merge with a merge commit.
   `release.yml` publishes or deploys `vX.Y.(Z+1)`, exactly like a release. At
   the merge it also closes the milestone, opens Hotfix `X.Y.(Z+2)` and the
   sync PRs (section 19).

Hotfixes do not use Promote to release.

### 18.3 Idle Hotfix PR, or one with work

A Hotfix PR **has work** when it has a commit besides the bot's version
commit, or an open PR has its branch as base. Otherwise it is **idle**.

An idle Hotfix PR stays open and red (P7), so nobody ships an empty patch. It
blocks neither the cut nor the Release PR. When the next release merges, a
patch for the old line can no longer ship. So the bot then closes the Hotfix
PR with a comment (section 18.2).

A Hotfix PR whose only work is an open fix PR blocks releases (P5). It stays
red (P7) until the fix merges.

### 18.4 A Hotfix PR blocks releases

The risk is that Release `X.(Y+1).0` merges while Hotfix `X.Y.(Z+1)` has
work. Then its fixes are lost, or an old line publishes over new code. So
while a Hotfix PR has work (section 18.3), nothing else ships. To release
anyway, a human closes those PRs (or the Hotfix PR itself, E27).

| Layer | When | Rule |
|---|---|---|
| Promote to release | Before the cut | Refuses while a Hotfix PR has work. |
| `policy` on the Release PR (P5) | Each PR event, through `pr-guard.yml` | Red while a Hotfix PR has work. |
| `next-hotfix` at the merge of `X.(Y+1).0` | At the merge | An idle Hotfix PR for the older line: closed with a comment (18.3). One with work: left open. It keeps the next Release PR red (P5) until a human ships or closes it. |

### 18.5 A hotfix after a human closed the Hotfix PR

An idle Hotfix PR blocks nothing, so nobody needs to close it. If a human
closes one anyway and a bug comes later, `create-hotfix.yml` handles it:

- Manual dispatch, no inputs, only from `<DEV>` (job guard and environment, as
  for Promote to release).
- Does what `next-hotfix` does, from the head of `<MAIN>`. Refuses if a
  Hotfix PR is already open.
- First deletes the leftover branch. It does not move fixes from the old
  branch to the new branch. They stay reachable from the closed PR (E27).
- From then on, it is a normal Hotfix PR (sections 18.2 to 18.4).

## 19. Sync into DEV and into an open release branch

### 19.1 When

Every merge into `<MAIN>` comes back, through follow-up jobs of `release.yml`
(section 8.8).

| Merge into `<MAIN>` | Sync PR into `<DEV>` (19.2) | Sync PR into the release branch (19.5) |
|---|---|---|
| Release PR `X.Y.0` | Yes: stacked fixes. | No: no other release branch is open. |
| Hotfix PR `X.Y.Z` | Yes: fixes. | Yes, if a release branch is open. |

An example with real values, step by step: section 5.8, steps 6 to 8.

### 19.2 Sync PR into DEV (job `sync-develop`, app token)

PR "Sync X.Y.Z to `<DEV>`", branch `sync/dev/X.Y.Z`. If a sync PR for `X.Y.Z`
already exists, open or merged, the job reuses it or stops (a rerun is safe).

1. **Pick the commits.** Only the merged branch's own commits, on its first
   parent. The job skips a merge commit that is the cut's merge or a sync
   PR's merge. It picks any other merge commit (a stacked PR merged with a
   merge commit by mistake) as one change against its first parent.
   - Release PR: `git rev-list --first-parent <DEV>..<merge>^2`, the release
     branch's own line back to the cut: stacked fixes.
     Without `--first-parent`, the cut's merge would include all of `<MAIN>`.
     The job skips the cut's merge and sync PR merges; their content is
     already in `<DEV>`. The job finds a sync PR merge by its PR (head `sync/**`), not by
     its parents. GitHub's outer merge commit has the sync branch, not
     `<MAIN>`, as its second parent.
   - Hotfix PR: the commits on the hotfix branch since the previous `<MAIN>`
     head (`<merge>^1..<merge>^2`): the fixes.
2. Create `sync/dev/X.Y.Z` from the head of `<DEV>`. Cherry-pick the
   commits in order.
3. **`VERSION` is never synced.** The job drops each commit's change to
   `VERSION`. It skips a commit that changes only `VERSION` (the bot's hotfix
   version commit). `<DEV>` keeps its own version.
4. Resolve:
   - **No conflict:** push, open the PR, and turn on auto-merge with the
     squash method. `<DEV>` stays linear: one commit per sync.
   - **Code conflict:** stop at that commit. Push what applied (an empty first
     commit when nothing applied, so the PR can exist). Open the PR as a
     draft with the label `needs-manual-merge`, and list the commits left
     and the command to apply them. No auto-merge: a draft cannot merge, so a
     half sync never lands. A human finishes on the same branch, marks it
     ready, and squash merges it.

### 19.3 Expected conflicts

| File | Conflict? |
|---|---|
| `VERSION` | Never: it is not synced. |
| Code | Possible, when `<DEV>` changed the same lines after the cut or after the hotfix branched. A human resolves it (step 4). |

### 19.4 Why squash works here

A sync never merges history. It carries only its own commits, picked by range.
The next sync picks a new range, so it never sees old changes again. Git does
not need to know that `<MAIN>`'s commits are in `<DEV>`.

The cost is that `<DEV>` holds the content of `<MAIN>`'s commits, never the
commits themselves. A release branch cut from `<DEV>` would then conflict with
`<MAIN>` on `VERSION` after any hotfix. The cut handles this. It merges
`<MAIN>` into the new release branch (8.5), so every Release PR starts with
all of `<MAIN>`'s content.

The trade-off of squashing into `<DEV>`: when `<DEV>` edits lines that a
synced hotfix also touched, the cut's merge can conflict. A human then
resolves a cut-case sync PR (usually keeping `<DEV>`'s lines).

### 19.5 Sync PR into an open release branch

**When.**

- **Hotfix case.** A Hotfix PR `X.Y.(Z+1)` merges while `release/X.(Y+1).0`
  is open. `<MAIN>` now has the hotfix; the release branch does not. Branch
  `sync/release/X.Y.(Z+1)`, PR "Sync X.Y.(Z+1) into release/X.(Y+1).0".
- **Cut case.** The cut's merge of `<MAIN>` into a new `release/X.Y.0` has a
  conflict (8.5 step 3.2). Branch `sync/release/X.Y.0-cut`, PR "Sync `<MAIN>`
  into release/X.Y.0".

**Why it is needed.** Hotfix case: the Release PR now conflicts with `<MAIN>`
on `VERSION` (the release branch has `X.(Y+1).0`, `<MAIN>` has `X.Y.(Z+1)`),
and maybe on code. Cut case: `<MAIN>` and `<DEV>` changed the same lines.
A human decides which side wins (usually `<DEV>`'s). In both cases, the fix
for a Release PR conflict is a PR that targets the release branch, never a
change to `<MAIN>`.

**Steps** (job `sync-release` in `release.yml`, or the cut job, app token).
If the release branch already contains the head of `<MAIN>`, stop. If a sync
PR for it is open, reuse it.

1. Create the sync branch from the head of the release branch.
2. Merge the head of `<MAIN>` into it, without committing yet.
3. Resolve:
   - **No conflict:** commit.
   - **Only `VERSION`:** keep the release branch's value and commit.
   - **Any other conflict:** abort. Reset the sync branch to the head of
     `<MAIN>`, push it, and open the PR with the label `needs-manual-merge`.
     GitHub shows the conflict. A human merges the release branch into the
     sync branch locally, keeps the release branch's `VERSION`, and pushes.
4. Open the PR into the release branch.
5. Turn on auto-merge with the merge commit method.

**Why a merge commit here.** The Release PR's `VERSION` conflict disappears
only when git knows that `<MAIN>` was merged. A squash does not record that.
`release/**` therefore allows merge commits as well as squash (section 6.1).
Stacked fix PRs keep using squash. Into `<DEV>`, everything squashes (19.2).

**Gate.** Until the sync PR merges, the Release PR conflicts with `<MAIN>`,
and GitHub blocks its merge. A human who resolves a conflict follows the same
path: a PR into the release branch, merged with a merge commit.

## 20. Milestones

- Milestone title = the version, exactly `X.Y.Z`.
- Only the close job (20.1) sets PR milestones, when the version ships. Open
  PRs have no milestone. Humans set issue milestones.
- **A closed milestone is final.** It means "merged into `<MAIN>`". The
  automation never reopens it and never assigns it. GitHub milestone titles
  are unique, closed ones included.

### 20.1 `release.yml`, job `milestone-close`

It runs right after the `version` job, for both release kinds. It does not
wait for build, publish or deploy. The milestone closes even if they are still
running or fail later. A rerun of a failed build uses the same merge commit
and ships the same `V` (E43). Input: the released version `V` (`VERSION` at the
merge commit `M`) and the previous head of `<MAIN>` (`M^1`).

1. **Make sure milestone `V` exists.**
2. **Set `V` on the shipped PRs.** For each commit in `M^1..M`, find its PR
   and set `V`. Skip sync PRs and PRs already in a closed milestone.
3. **Issues follow their PRs.** Issues closed by a PR in `V` get `V`, unless
   they are in a closed milestone.
4. **Close `V`.**

A rerun changes nothing. The release notes link milestone `V`.

## 21. Policy checks in the gate

`pr-guard.yml` posts the `policy` commit status (section 21.1). It reads the
PR's file list, author and branch state through the API, as data.
It runs on every PR, docs-only ones too.

| Rule | Applies to | Fails when |
|---|---|---|
| P1 | PRs into `<MAIN>` | The head is not `release/*` or `hotfix/*` in this repository, or the PR author is not `<BOT_APP>`. A fork branch with the same name never counts. |
| P2 | PRs into `<DEV>` | The PR changes `VERSION` together with another file. |
| P3 | PRs into `release/*` or `hotfix/*` | The PR changes `VERSION`. |
| P5 | Release PRs | A Hotfix PR has work (section 18.3). An idle Hotfix PR does not count. Only PRs that pass P1's identity check count, so a fork PR cannot block a release. |
| P7 | Hotfix PRs | The head has no commit besides the bot's version commit. |
| P8 | PRs into `<MAIN>` | An open PR has the head branch as its base (a stacked PR is not merged yet). Fork PRs count too: a maintainer closes or retargets an unwanted one. |
| P10 | PRs that are not Release or Hotfix PRs | The PR changes a file under `.github/workflows/` or `.github/actions/`, unless every change is a verified action pin bump. Also fails when the API lists fewer files than the PR changed. An admin reviews such a PR and merges it with the ruleset bypass. |

**Verified pin bump (P10).** Each added line is a `uses:` line with a full
SHA and a tag comment. It pairs with the removed line before it: the same
text before `@`. The tag has no `/` and no `..`. The job resolves the tag
upstream (URL-encoded, the returned `ref` checked, an annotated tag peeled).
The new SHA must be the commit the tag names. So a weekly Dependabot
actions PR merges without an admin, and the same diff with another SHA stays
red.

P5 and P8 depend on other PRs, so they can change without any event on the
Release or Hotfix PR itself. `pr-guard.yml` posts the parent's status again
when they change (21.1).

### 21.1 `pr-guard.yml`

- Trigger: `pull_request_target` on `opened`, `reopened`, `synchronize`,
  `edited`, `closed`, `ready_for_review`, for PRs into `<DEV>`, `<MAIN>`,
  `release/**` and `hotfix/**`.
- Checks out only `github.workflow_sha`, sparse (`.github/actions`), with
  `persist-credentials: false`. GitHub runs a `pull_request_target` workflow
  from the default branch. So the actions must come from that same commit,
  not from the PR's base. A PR cannot change the rules that judge it.
- Environment `guard` (6.2). App token with `contents: read`,
  `pull-requests: read` and `statuses: write`.
- Posts `policy` on the PR's head commit. Then posts it again on each parent:
  - for a PR whose base is `release/**` or `hotfix/**`: the open PR from that
    branch into `<MAIN>`;
  - for a PR whose base is `hotfix/**`: also the open Release PR (P5 counts
    the Hotfix PR's work);
  - for a Hotfix PR (head `hotfix/**`, base `<MAIN>`): the open Release PR;
  - on `edited` with a base change: also the old base's parent, which may
    turn green.
- The rulesets pin `policy` to `<BOT_APP>`. A status that a PR workflow posts
  with another token does not count.
- A scheduled workflow cannot do this job: its result lands on the default
  branch, not on the PR's head.
- zizmor reports `dangerous-triggers`; the inline ignore says: "checks out
  the workflow commit only, never PR code".
- Concurrency: section 10.1.

## 22. Edge cases

| # | Case | Handling |
|---|---|---|
| E3 | Release PR closed without merge (abandoned) | `X.Y.0` never ships. Manual steps: cherry-pick the stacked fixes into `<DEV>`, then delete `release/X.Y.0`. |
| E4 | `<DEV>` moved between the checks and the cut | The cut job stops red before any push. Dispatch it again (8.5 step 3.1). |
| E5 | Version-jump PR (1.5.0 to 2.0.0) | A PR that changes only `VERSION` (P2); a trusted human reviews and merges it. |
| E10 | Fork PR | CI runs with a read-only token. `pr-guard.yml` runs base code only and reads PR data through the API (9.3). |
| E17 | Next minor release merges before the Hotfix PR | Blocked: section 18.4. |
| E18 | Hotfix reaches `<DEV>` | Sync PR into `<DEV>` (19.2). |
| E19 | Stacked PR open when the Release or Hotfix PR is about to merge | `policy` P8 keeps the PR red; `pr-guard.yml` posts it again on the parent (21.1). |
| E20 | Head branch deleted on merge while PRs still target it | GitHub retargets them to `<MAIN>`. P1 turns them red there. The author retargets each one by hand, usually to the new hotfix branch. |
| E23 | Sync PR into `<DEV>` stops on a code conflict | A draft with `needs-manual-merge` and no auto-merge (19.2 step 4). The cut refuses while it is open. |
| E24 | A fix already reached `<DEV>` manually | Its cherry-pick is empty, and the job skips it. |
| E25 | Sync PR into `<DEV>` still open at the next cut | Promote to release refuses. |
| E26 | Idle Hotfix PR | Red (P7), blocks nothing. The bot closes it with a comment when the next release merges (18.3). |
| E27 | Hotfix PR with fixes closed without merge | Its fixes are not in `<DEV>`. `create-hotfix` deletes the leftover branch when it opens a new Hotfix PR (18.5); its commits stay reachable from the closed PR. |
| E33 | Hotfix for an older line | The line rule refuses it (5.4). Only the latest line gets hotfixes (S12). |
| E35 | `VERSION` with a carriage return, a space or a byte-order mark | The guard fails it. |
| E38 | Promote to release run from another branch | Job guard and environment branch policy stop it. |
| E39 | Bug found after the Hotfix PR was closed, before the next release | `create-hotfix.yml` (18.5). |
| E40 | Hotfix ships while a release branch is open | Sync PRs into `<DEV>` and into the release branch (19.1). The Release PR conflicts on `VERSION`, so GitHub blocks it until the second one merges. |
| E41 | Next Hotfix PR opens while a Release PR waits | It is idle, so the Release PR stays green. A fix PR into it turns the Release PR red (P5). |
| E42 | A human squashes a sync PR | The Release PR still conflicts. The ruleset allows squash on `release/**` for stacked fixes, so no rule can block this. A maintainer reruns the `sync-release` job: the release branch still lacks the head of `<MAIN>`, so it opens a new sync PR with auto-merge (merge commit). |
| E43 | Build, publish or deploy fails after the merge | Milestone `V` is already closed. Rerun the failed jobs; the follow-up jobs reuse or skip what exists (18.2, 19.2, 19.5); it ships `V` from the same merge commit. If the cause is in the code, `V` never ships: its milestone stays closed with no tag, and the next Hotfix PR (`V` plus one patch) carries the fix. |
| E44 | Hotfix ships while no release branch is open | Sync PR into `<DEV>` only. The next cut merges `<MAIN>` into the new release branch, so the Release PR does not conflict on `VERSION`. |
| E45 | The cut's merge of `<MAIN>` conflicts | The cut drops the merge and opens a cut-case sync PR. The Release PR conflicts until it merges (8.5 step 3.2, 19.5). |
| E49 | A sync carries a change to a file in `.github/workflows` | The bot has no `workflows: write`, so that push fails. The job opens the sync PR as a draft with `needs-manual-merge` and lists the commits left; a maintainer pushes them. This is rare: hotfixes and stacked fixes seldom touch workflows. |

## 23. Design decisions

Settled decisions, with the sections that apply them.

| # | Decision | Result |
|---|---|---|
| S1 | Dev build on `<DEV>` | Two modes, consumer's choice (`<DEV_BUILD_MODE>`): manual or automatic. Named "dev build", not pre-release. No versioned dev releases: an `app` dev build is a workflow artifact (5.6). |
| S2 | Promote to release | Cut only, no inputs, always `<DEV>` (8.5). |
| S3 | Hotfix PR | Opened at every Release PR or Hotfix PR merge, normal PR (not a draft) (18.2). |
| S4 | Merge readiness | Release and Hotfix PRs are green only when no stacked PR into their branch is open (P8). |
| S5 | `<DEV>` bump | At the cut; the release branch is permanently separate from `<DEV>` (5.1, 8.5). |
| S6 | Version file | Raw `VERSION`; other manifests read it at build (5.3). |
| S7 | Milestones | Set only when the version ships, by the close job (20.1). |
| S8 | Hotfix PRs and releases | A release is blocked while a Hotfix PR has work. An idle one blocks nothing; the bot closes it with a comment when the next release merges, and never closes one that has work (18.3, 18.4). |
| S9 | Delivery | Profile `app` or `web` (section 6.4). With `web`: PR previews to `<PREVIEW_ENV>`, dev build to `<DEV_ENV>`, release branch RC to `<STAGING_ENV>`, hotfix branch RC to `<HOTFIX_ENV>`, release to `<PROD_ENV>`. |
| S10 | When a milestone closes | At the merge of its Release PR or Hotfix PR, without waiting for build, publish or deploy (20.1). |
| S12 | Release lines with hotfixes | Latest line only. No long-term support or older lines (section 5.4, line rule). |
| S15 | Hotfix after a human closed the Hotfix PR | `create-hotfix.yml` (18.5). |
| S16 | Web rollback | Out of scope. The hosting side behind `<DEPLOY_CMD>` owns rollback. Fix forward with a hotfix in CI. |
| S17 | How a merge into `<MAIN>` reaches `<DEV>` | Sync PR into `<DEV>` after every Release PR and Hotfix PR merge. Only that branch's own commits, never `VERSION`, squash merged (19.2). |
| S18 | Release notes | GitHub generated notes since the previous release tag, grouped by `.github/release.yml`. No `CHANGELOG.md` (13). |
| S19 | Automatic dev builds under a burst of merges | One build per burst (the default concurrency queue), not one per merge. Every merged PR is in the newest build (5.6). |
| S20 | Checks on Release and Hotfix PRs | Every check and CodeQL, like any PR, with no preview. Sync PRs keep every scan, with no `package` and no preview (8.2, 8.3). |
| S21 | When release candidates build | `<RELEASE_RC_MODE>` and `<HOTFIX_RC_MODE>`, each manual (default) or automatic on each push (5.5). |
| S22 | Approvals | No ruleset requires an approving review. Sync PRs merge by themselves once `gate` is green (6.1). |
| S23 | Template options | Where a standard choice has more than one sound answer, the design offers each as a placeholder option with a default. The consumer picks one (section 2). |
| S24 | Simplicity | Use GitHub's own features (delete branch on merge, generated notes, auto-merge, rulesets) before bot code. A rare case or a race of seconds gets a human step, not more automation. |

## 24. Appendix: YAML skeletons

Replace `<SHA>` with the full commit SHA of each action's release. Keep the
tag in the comment.

### 24.1 Common job head

```yaml
permissions:
  contents: read

jobs:
  test:
    runs-on: ubuntu-latest
    timeout-minutes: 10
    steps:
      - uses: actions/checkout@<SHA> # vX.Y.Z
        with:
          persist-credentials: false
      - uses: <TOOLCHAIN_SETUP>@<SHA> # vX.Y.Z
      - run: <TEST_CMD>
```

### 24.2 `ci.yml`: changes, checks, gate

```yaml
on:
  push:
    branches: [<DEV>, <MAIN>, 'release/**', 'hotfix/**']
  pull_request:
  schedule:
    - cron: '0 5 * * 1'

permissions:
  contents: read

concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: ${{ github.event_name == 'pull_request' }}

jobs:
  changes:
    runs-on: ubuntu-latest
    timeout-minutes: 5
    permissions:
      contents: read
      pull-requests: read
    outputs:
      code: ${{ github.event_name != 'pull_request' || steps.f.outputs.code == 'true' }}
      kind: ${{ steps.k.outputs.kind }}
    steps:
      - id: k
        env:
          EVENT: ${{ github.event_name }}
          HEAD: ${{ github.head_ref }}
          HEAD_REPO: ${{ github.event.pull_request.head.repo.full_name }}
          REPO: ${{ github.repository }}
        run: |
          set -euo pipefail
          kind=normal
          if [ "$EVENT" = "pull_request" ] && [ "$HEAD_REPO" = "$REPO" ]; then
            case "$HEAD" in
              sync/*) kind=sync ;;
            esac
          fi
          echo "kind=$kind" >> "$GITHUB_OUTPUT"
      - id: f
        if: github.event_name == 'pull_request'
        uses: dorny/paths-filter@<SHA> # vX.Y.Z
        with:
          predicate-quantifier: every
          filters: |
            code:
              - '**'
              - '!**/*.md'
              - '!docs/**'

  checks:
    needs: changes
    if: needs.changes.outputs.code == 'true'
    permissions:
      contents: read
      pull-requests: read
      security-events: write
    uses: ./.github/workflows/checks.yml

  package:
    needs: [changes, checks]
    if: needs.changes.outputs.code == 'true' && needs.changes.outputs.kind != 'sync'
    # ... (section 24.3)

  # `policy` is not here: pr-guard.yml posts it as its own required status (section 21.1).

  gate:
    needs: [changes, checks, package]
    if: always()
    runs-on: ubuntu-latest
    timeout-minutes: 2
    steps:
      - env:
          FAILED: ${{ contains(needs.*.result, 'failure') || contains(needs.*.result, 'cancelled') }}
        run: |
          if [ "$FAILED" = "true" ]; then
            echo "::error::a required job failed or was cancelled"; exit 1
          fi
```

### 24.3 Split job: build with no write token, post with no checkout

```yaml
  package:
    needs: [changes, checks]
    if: needs.changes.outputs.code == 'true' && needs.changes.outputs.kind != 'sync'
    runs-on: ubuntu-latest
    timeout-minutes: 10
    outputs:
      card: ${{ steps.card.outputs.card }}
    steps:
      # checkout (persist-credentials: false), toolchain
      - run: <PACKAGE_CMD>
      - id: card
        if: github.event_name == 'pull_request' && github.base_ref != '<MAIN>'
        uses: actions/github-script@<SHA> # render the card from env, set output `card`
        env:
          SHA: ${{ github.event.pull_request.head.sha }}

  canary:
    needs: [package]
    if: >-
      github.event_name == 'pull_request' &&
      needs.package.outputs.card != '' &&
      github.event.pull_request.head.repo.full_name == github.repository &&
      github.event.pull_request.user.login != 'dependabot[bot]'
    runs-on: ubuntu-latest
    timeout-minutes: 5
    permissions:
      pull-requests: write
    steps:
      # No checkout.
      - env:
          CARD: ${{ needs.package.outputs.card }}
          GH_TOKEN: ${{ github.token }}
          REPO: ${{ github.repository }}
          PR: ${{ github.event.pull_request.number }}
        run: |
          set -euo pipefail
          printf '%s' "$CARD" | base64 -d > "$RUNNER_TEMP/card.md"
          ID=$(gh api --paginate "repos/$REPO/pulls/$PR/reviews" \
            --jq '.[] | select(.user.login == "github-actions[bot]" and (.body | startswith("<CARD_HEADING>"))) | .id' | tail -n 1)
          if [ -n "$ID" ]; then
            gh api -X PUT "repos/$REPO/pulls/$PR/reviews/$ID" -F body=@"$RUNNER_TEMP/card.md" --silent
          else
            gh api -X POST "repos/$REPO/pulls/$PR/reviews" -f event=COMMENT -F body=@"$RUNNER_TEMP/card.md" --silent
          fi
```

### 24.4 Build: attestation

```yaml
  build:
    needs: [version, checks]
    runs-on: ubuntu-latest
    timeout-minutes: 15
    permissions:
      contents: read
      id-token: write
      attestations: write
    steps:
      # checkout (persist-credentials: false), toolchain
      - env:
          VERSION: ${{ needs.version.outputs.version }}
        run: <PACKAGE_CMD> "$VERSION"

      - uses: actions/attest-build-provenance@<SHA> # vX.Y.Z
        with:
          subject-path: dist/<ARTIFACT>*

      - uses: actions/upload-artifact@<SHA> # vX.Y.Z
        with:
          name: release-assets
          path: dist/
          if-no-files-found: error
```

### 24.5 Release bot token and push

```yaml
      - id: app
        uses: actions/create-github-app-token@<SHA> # vX.Y.Z
        with:
          app-id: ${{ vars.BOT_APP_ID }}
          private-key: ${{ secrets.BOT_APP_KEY }}

      - name: Push the release branch, then bump DEV
        env:
          GH_TOKEN: ${{ steps.app.outputs.token }}
          GH_REPO: ${{ github.repository }}
          VERSION: ${{ needs.version.outputs.version }}
          NEXT: ${{ needs.version.outputs.next }}
          CHECKED_SHA: ${{ needs.version.outputs.sha }}
        shell: bash
        run: |
          set -euo pipefail
          git config user.name '<BOT_APP>[bot]'
          git config user.email '<BOT_USER_ID>+<BOT_APP>[bot]@users.noreply.github.com'
          remote="https://x-access-token:${GH_TOKEN}@github.com/${GH_REPO}.git"
          # stop before any push if <DEV> moved since the checks (section 8.5 step 3.1)
          if [ "$(git ls-remote "$remote" refs/heads/<DEV> | cut -f1)" != "$CHECKED_SHA" ]; then
            echo "::error::<DEV> moved since the checks; dispatch again"; exit 1
          fi
          # release/$VERSION was prepared locally (cut merge kept, or dropped on a conflict)
          git push "$remote" "release/$VERSION:refs/heads/release/$VERSION"
          git switch --detach "$CHECKED_SHA"
          printf '%s\n' "$NEXT" > VERSION
          git commit -am "chore: version $NEXT"
          git push "$remote" HEAD:<DEV>
```

`<BOT_USER_ID>` is the bot user's numeric ID:
`gh api users/<BOT_APP>%5Bbot%5D --jq .id`.

### 24.6 Release: version first, follow-ups at the merge, publish last

```yaml
on:
  push:
    branches: [<MAIN>]
  workflow_dispatch:

permissions:
  contents: read

concurrency:
  group: release
  cancel-in-progress: false

jobs:
  version:
    if: github.ref == 'refs/heads/<MAIN>'
    # read VERSION -> rule of the merged branch kind -> refuse re-release (tag exists)
  milestone-close:
    needs: version
    if: github.ref == 'refs/heads/<MAIN>'
    environment: release
    # section 20.1 (app token: milestones and issues)
  next-hotfix:
    needs: version
    if: github.ref == 'refs/heads/<MAIN>'
    environment: release
    # section 18.2
  sync-develop:
    needs: version
    if: github.ref == 'refs/heads/<MAIN>'
    environment: release
    # section 19.2
  sync-release:
    needs: version
    if: github.ref == 'refs/heads/<MAIN>'   # and: hotfix merge while a release branch is open
    environment: release
    # section 19.5
  checks:
    needs: version
    permissions:
      contents: read
      pull-requests: read
      security-events: write
    uses: ./.github/workflows/checks.yml
  build:
    needs: [version, checks]
    uses: ./.github/workflows/build.yml   # 24.4
  deploy:                    # <PROFILE> = web only
    needs: build
    if: github.ref == 'refs/heads/<MAIN>'
    environment: <PROD_ENV>
    # download artifact -> <DEPLOY_CMD>
  publish:
    needs: [build, deploy]     # [build] for <PROFILE> = app
    if: github.ref == 'refs/heads/<MAIN>'
    environment: release
    # download artifact -> app token -> gh release create <REL_TAG> ... --target "$GITHUB_SHA"
    #   --generate-notes --notes-start-tag <previous vX.Y.Z> (resume a matching draft)
```

### 24.7 Dependabot with cooldown

```yaml
version: 2
updates:
  - package-ecosystem: github-actions
    directory: /
    schedule:
      interval: weekly
    cooldown:
      default-days: 7
    groups:
      actions:
        patterns: ['*']
        update-types: [minor, patch]   # each major update: its own PR
    commit-message:
      prefix: ci

  - package-ecosystem: <ECOSYSTEM>   # one entry per lockfile
    directory: /<TOOLS_DIR>
    schedule:
      interval: weekly
    cooldown:
      default-days: 7
    groups:
      minor-and-patch:
        patterns: ['*']
        update-types: [minor, patch]
    commit-message:
      prefix: ci
```
