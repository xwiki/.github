# XWiki `.github`

The GitHub configuration the XWiki repositories share, held in one place so that a change is made
once instead of being copied into every repository and drifting there.

GitHub gives a repository named `.github` two powers, and this repository uses both. Everything
here is public because both of them require it.

## Default community health files

Files under `.github/` that GitHub recognises as
[community health files](https://docs.github.com/communities/setting-up-your-project-for-healthy-contributions/creating-a-default-community-health-file)
apply to **every public repository of the `xwiki` organization that does not define its own**. A
repository that needs something different simply keeps its own copy, which always wins.

| File | What it does |
| --- | --- |
| [`.github/pull_request_template.md`](.github/pull_request_template.md) | The description a new pull request is pre-filled with. |
| [`.github/FUNDING.yml`](.github/FUNDING.yml) | The "Sponsor" button, pointing at the XWiki Open Collective. |
| [`.github/SECURITY.md`](.github/SECURITY.md) | Where to report a vulnerability, pointing at the XWiki security policy. |

Nothing has to be done in a repository to pick these up: deleting its own copy is what makes the
default apply.

## Reusable workflows

Workflows, unlike community health files, are **not** inherited. Each repository keeps a short stub
in `.github/workflows/` that calls the shared implementation held here, so the logic lives once and
only the trigger is repeated.

| Workflow | What it does |
| --- | --- |
| [`.github/workflows/quality-pr.yml`](.github/workflows/quality-pr.yml) | Fails a pull request that introduces a Checkstyle violation or a SonarQube issue on one of its own lines. |
| [`.github/workflows/backport.yml`](.github/workflows/backport.yml) | Backports a merged pull request to the branches its `backport-*` labels name. |

### Calling them

`quality-pr.yml`, in `<repository>/.github/workflows/quality-pr.yml`:

```yaml
name: Pull request quality checks

on:
  pull_request:
    types: [opened, synchronize, reopened]

concurrency:
  group: quality-pr-${{ github.event.pull_request.number }}
  cancel-in-progress: true

jobs:
  analysis:
    name: Quality
    ## A pull_request run from a fork gets no repository secrets, so the analysis has no token to
    ## authenticate with. Skip those rather than fail them.
    if: github.event.pull_request.head.repo.full_name == github.repository
    permissions:
      contents: read
    ## Referenced by branch and not by commit hash, on purpose: pinning would mean a pull request in
    ## every repository for every change to the shared workflow, which is the duplication this
    ## indirection removes. What pinning protects against, a third party repointing the ref under
    ## us, does not apply to a repository the XWiki committers own themselves.
    uses: xwiki/.github/.github/workflows/quality-pr.yml@master
    secrets:
      SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
```

It takes one optional input, `maven-profiles` (default `snapshot,legacy`): the profiles every Maven
command of the analysis runs with. They have to match the profiles the ci.xwiki.org "quality" build
analyzes `master` with, or a module the pull request gate never sees ends up analyzed only after
the merge.

`backport.yml`, in `<repository>/.github/workflows/backport.yml`:

```yaml
name: Automatic backport action

on:
  pull_request_target:
    types: ["labeled", "closed"]

jobs:
  backport:
    ## Referenced by branch and not by commit hash, on purpose: pinning would mean a pull request in
    ## every repository for every change to the shared workflow, which is the duplication this
    ## indirection removes. What pinning protects against, a third party repointing the ref under
    ## us, does not apply to a repository the XWiki committers own themselves.
    uses: xwiki/.github/.github/workflows/backport.yml@master
```

The stubs reference `@master` rather than a commit hash, so a change here reaches every repository
on its next run. That is the point of this repository, and also the reason to treat a change to a
workflow as a change to all of them: there is no per-repository staging. It is why the stubs carry
that note, and why SonarQube's `githubactions:S7637` ("use full commit SHA hash for this
dependency") is accepted on those two lines — the rule guards against a third party repointing a
mutable ref, which does not describe a repository the XWiki committers own themselves.

## Renovate preset

[`renovate/default.json5`](renovate/default.json5) is a
[Renovate config preset](https://docs.renovatebot.com/config-presets/) holding the settings every
XWiki repository wants. A repository's own `.github/renovate.json5` extends it and adds its
specifics:

```json5
{
  "extends": ["github>xwiki/.github//renovate/default.json5"],

  // ... whatever is specific to this repository
}
```

Mind how Renovate merges the two: `packageRules` are concatenated, the preset's coming first, but
every other array is replaced wholesale by the repository's value. A repository that needs an extra
`ignoreDeps` entry has to repeat the preset's entries alongside its own.

## Which repositories use this

`xwiki-commons`, `xwiki-rendering` and `xwiki-platform` today. The community health files reach the
rest of the organization on their own.

## License

LGPL 2.1, as the rest of XWiki. See [`LICENSE`](LICENSE).
