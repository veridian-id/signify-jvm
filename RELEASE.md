# Release process

Published coordinates are `id.veridian:signify:<version>`, and `<version>` is the release
tag. A release happens when someone pushes a semver tag; merging to `main` no longer
publishes anything.

## Where the version comes from

CI passes the tag to Gradle as `RELEASE_VERSION` with the leading `v` stripped, so pushing
`v0.1.3` publishes `0.1.3`. `baseVersion` in `gradle.properties` is the snapshot seed: the
release job advances it to the next patch after a published release so the next branch and PR
inherit it.

| Condition | Effective version | Example |
| --- | --- | --- |
| `RELEASE_VERSION` set (tag push) | exactly that value | `0.1.3` |
| `PR_NUMBER` set | `baseVersion-pr<N>-<commit>-SNAPSHOT` | `0.1.2-pr7-378fa88-SNAPSHOT` |
| neither (local/dev) | `baseVersion-<commit>-SNAPSHOT` | `0.1.2-378fa88-SNAPSHOT` |

`<commit>` is the abbreviated HEAD commit hash from the `org.ajoberstar.grgit` plugin. CI
sets exactly one of the two variables, so a build is either a snapshot or a release — never
both.

Check what a build would publish:

```bash
./gradlew -q properties | grep '^version:'
```

## Releasing

```bash
git tag -a v0.1.3 -m "release 0.1.3"
git push origin refs/tags/v0.1.3
```

The tag points at the commit that gets published, so create it on `main` after the merge you
want to ship. Only `vMAJOR.MINOR.PATCH` tags release: the trigger glob is `v*` (Action
filters cannot express digit classes), and the `resolve` job re-checks the exact shape, so
`v0.1.3-rc1`, `v1`, or `version-3` run CI without publishing.

Both paths trigger it: pushing a tag over Git fires `push`, and creating one on the server —
the Releases page, `gh release create`, the `git/refs` API — fires `create`. `create` fires for
branch creation too, so the jobs only run for it when the ref is a `v*` tag.

## Pipeline

```mermaid
flowchart TD
  PR["PR: baseVersion-prN-sha-SNAPSHOT to GitHub Packages"] --> M["merge to main: tests only, no publish"]
  M --> T["vMAJOR.MINOR.PATCH tag, pushed over Git or created on the server"]
  T --> C["unit-tests + e2e-tests on the tagged commit"]
  C --> R["resolve: semver shape + who created the tag"]
  R --> P["publish + jreleaserDeploy as the tag version"]
  P --> B["commit: baseVersion advanced to the next patch on main"]
  B -->|next branch and PR inherits| PR
```

### `snapshot` job (pull requests)

```bash
./gradlew build publishMavenJavaPublicationToGitHubPackagesRepository
```

Snapshots go to GitHub Packages only; the Maven Central snapshot repository is deliberately
left out so PR commits do not accumulate in Central.

### `resolve` job (tag events)

1. **Semver** — the tag name taken from `GITHUB_REF` must match
   `^v[0-9]+\.[0-9]+\.[0-9]+$`; otherwise nothing is published and the run says so in the summary.
2. **Author** — `gh api repos/<repo>/collaborators/<actor>/permission` must come back `write`
   or `admin`. This is the CI backstop for the tag ruleset below: a tag created by someone
   who cannot release stops here.
3. **Next patch** — `major.minor.(patch + 1)` is output for the release job.

### `release` job (tag events that pass both checks)

1. Publishes `RELEASE_VERSION=<tag version>` from `build/staging-deploy` through JReleaser.
2. Commits `chore: bump baseVersion to <next patch> [skip ci]` on `main`, so new branches and
   PRs inherit it. The step exits without a commit when `main` is already at that number.

CI never creates a tag — the tag already exists and is the version. The commit only moves the
snapshot seed.

## Protecting tag creation

Workflow permissions cannot stop a tag from being created, so the gate is repository settings
plus the CI check above.

| Control | What it does | Where |
| --- | --- | --- |
| Tag ruleset (the real gate) | Blocks tag creation and deletion for everyone except the bypass list — add Admin and Maintain, or a named team. Applies to tags created through Releases too. | Settings → Rules → Rulesets → New ruleset → New tag ruleset, target `v*` |
| Semver name pattern | Rejects tags that are not `v<major>.<minor>.<patch>` at creation time, so a bad tag never reaches CI | Same ruleset, "Restrict tag names" → must match regex `^v[0-9]+\.[0-9]+\.[0-9]+$` |
| `main` protection | Tags should point at reviewed commits; branch protection keeps `main` itself protected. A ruleset cannot say "tags may only point at `main`", so keep the convention that release tags are created on `main` | Settings → Rules → rulesets targeting `main` |
| CI author check | Backstop when the ruleset is bypassed, e.g. an integration that creates tags | `resolve` job, `Check who created the tag` |
| Role reality | Read-only collaborators and outside contributors cannot push tags to this repository at all; tags pushed to a fork do not run this workflow | — |

Ruleset JSON for import, equivalent to the two settings above:

```json
{
  "name": "Release tags",
  "target": "tag",
  "enforcement": "active",
  "conditions": { "ref_name": { "exclude": [], "include": ["v*"] } },
  "rules": [
    {
      "type": "tag_name_pattern",
      "parameters": {
        "name": "semver release tag",
        "operator": "regex",
        "pattern": "^v[0-9]+\\.[0-9]+\\.[0-9]+$",
        "description": "release tags must be vMAJOR.MINOR.PATCH"
      }
    },
    { "type": "creation" },
    { "type": "deletion" }
  ]
}
```

Then add the bypass actors in the UI (roles, teams, or apps) — `bypass_actors` is part of the
ruleset form, not the import file.

## Requirements

- Secrets used by the release job: `MAVEN_CENTRAL_GPG_KEY_BASE64`,
  `MAVEN_CENTRAL_GPG_PASSPHRASE`, `MAVEN_CENTRAL_OSSRH_USERNAME`,
  `MAVEN_CENTRAL_OSSRH_TOKEN`, signing key `3B36C37F3005FE8BB8B30DC26B12ADFA0BE0D092`.
- Maven Central rejects snapshot versions, so a release must publish a clean version
  (JReleaser runs with `applyMavenCentralRules = true`), which the semver check guarantees.
- `GITHUB_TOKEN` needs `packages: write` and `contents: write` on the release job, the second
  for the seed bump commit. Branch protection that forbids the bot pushing to `main`, or that
  requires every change to arrive as a pull request, fails that step.

## Notes

- Re-running a published tag fails at `jreleaserDeploy` — neither registry accepts a duplicate
  version — so a release cannot be overwritten and a failed re-run leaves the seed alone.
- The bump is patch-only. Major and minor lines start by choosing the number to tag; the seed
  follows on the next successful release run.
- `artifactId` is pinned to `signify` in `publish-common.gradle`; renaming it is a publication
  change, not a version change.
