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
| [`.github/workflows/quality-pr.yml`](.github/workflows/quality-pr.yml) | Fails a pull request that introduces a Checkstyle violation or a SonarQube issue on one of its own lines. Called by two stubs, one per trigger — see below. |
| [`.github/workflows/backport.yml`](.github/workflows/backport.yml) | Backports a merged pull request to the branches its `backport-*` labels name. |

### Calling them

`quality-pr.yml` is called by two stubs, because its two checks do not reach the same pull requests.
Checkstyle is a verdict the build gives on its own with no credentials involved, so it runs on every
pull request. The SonarQube half needs a token that can write to the XWiki SonarCloud projects, and a
job holding that token must only ever build code whose author could have pushed it into the
repository themselves — Maven executes whatever a pull request's own poms tell it to, so building a
stranger's change next to the token is handing them the token. That gives three cases:

| Trigger | Which pull requests | What they are checked with |
| --- | --- | --- |
| `pull_request` | a branch pushed into the repository | Checkstyle and SonarQube |
| `pull_request_target` | a fork whose author GitHub reports as an organization member | Checkstyle and SonarQube |
| `pull_request` | any other fork | Checkstyle only, SonarQube after the merge |

Membership is read from `github.event.pull_request.author_association`, which GitHub computes and no
pull request author can set. It is the only way to ask the question inside a workflow: a
`GITHUB_TOKEN` cannot read organization membership, and a token that could would be one more secret
to hold. It covers members whose membership is private. `COLLABORATOR` — someone granted access to
one repository — is deliberately not trusted, that access being possibly read-only.

The dependency bots — `renovate-bot`, `renovate[bot]` and `dependabot[bot]` — are excluded from both,
in the shared workflow rather than in the stubs: a version bump writes no line either check has an
opinion on, and there are enough of those pull requests, each pushed again on every rebase, to be most
of what this would ever spend. The author is what identifies them, not the `renovate/*` branch name,
which whoever opens a pull request chooses and which would therefore be a way to ask for the checks to
be skipped. `github-actions[bot]` is *not* excluded: the backport action opens pull requests under it
and those carry real code.

The two guards are exact negations of each other, so exactly one of them builds any given pull
request and the check to require in the branch protection is named `Quality / Analyze` either way.
**Changing one guard means changing the other**; drifting apart means either building twice or, worse,
not at all.

`quality-pr.yml`, in `<repository>/.github/workflows/quality-pr.yml`:

```yaml
## Hold each pull request to the quality rules the code base is held to, and fail it when the change
## breaks one, so that it is caught before the merge instead of turning the master quality build red
## afterwards. Everything but the trigger lives in the shared workflow, which every XWiki repository
## calls, so that a change to what is checked is made once. See https://github.com/xwiki/.github
##
## This is the half of the trigger that "pull_request" can serve on its own. Its companion,
## quality-pr-sonar.yml, takes the pull requests this one cannot check in full: those from a fork,
## which a pull_request run is handed no repository secrets for, and so no SonarQube token.
name: Pull request quality checks

on:
  pull_request:
    types: [opened, synchronize, reopened]

## One run per pull request: a new commit makes the running one obsolete.
concurrency:
  group: quality-pr-${{ github.event.pull_request.number }}
  cancel-in-progress: true

jobs:
  analysis:
    ## A reusable workflow names its check "<this job> / <the called job>", so this reads
    ## "Quality / Analyze". quality-pr-sonar.yml names its own job identically on purpose: exactly
    ## one of the two ever runs for a given pull request, so the check to require is called the same
    ## whichever one it was.
    name: Quality
    ## The pull requests this workflow answers for, which are the ones quality-pr-sonar.yml leaves to
    ## it: a branch pushed into this repository, which a pull_request run does hand SONAR_TOKEN to,
    ## and a fork whose author GitHub does not report as one of ours, which gets the Checkstyle
    ## verdict alone. This condition is the exact negation of that workflow's, so that a pull request
    ## is built once and not twice; changing one of the two means changing the other.
    if: >-
      github.event.pull_request.head.repo.full_name == github.repository
      || !contains(fromJSON('["OWNER", "MEMBER"]'),
                   github.event.pull_request.author_association)
    ## For a reusable workflow the GITHUB_TOKEN permissions are the caller's. The analysis only
    ## reads the sources.
    permissions:
      contents: read
    ## Referenced by branch and not by commit hash, on purpose: pinning would mean a pull request in
    ## every repository for every change to the shared workflow, which is the duplication this
    ## indirection removes. What pinning protects against, a third party repointing the ref under
    ## us, does not apply to a repository the XWiki committers own themselves.
    uses: xwiki/.github/.github/workflows/quality-pr.yml@master
    with:
      ## A pull_request run from a fork is given no repository secrets, so there is no token to
      ## analyze with and Checkstyle is the whole verdict. SonarCloud sees such a change after the
      ## merge, in the master analysis, as it did before this workflow existed.
      sonar: ${{ github.event.pull_request.head.repo.full_name == github.repository }}
    secrets:
      SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
```

`quality-pr-sonar.yml`, in `<repository>/.github/workflows/quality-pr-sonar.yml`:

```yaml
## Hold each pull request to the quality rules the code base is held to, and fail it when the change
## breaks one, so that it is caught before the merge instead of turning the master quality build red
## afterwards. Everything but the trigger lives in the shared workflow, which every XWiki repository
## calls, so that a change to what is checked is made once. See https://github.com/xwiki/.github
##
## This is the half of the trigger that "pull_request" cannot serve. A pull request from a fork is
## given no repository secrets, so quality-pr.yml can only run Checkstyle on it, and the SonarQube
## half needs SONAR_TOKEN; "pull_request_target" is the trigger that releases the secrets to a fork.
## Who they are released to is the whole question: this run builds the pull request's own code, and
## Maven executes what that code's poms tell it to -- a build extension, a plugin resolved from a
## repository the pom adds -- so the token is exactly as safe as the author is trusted. The guard
## below therefore limits this workflow to authors GitHub reports as members of the organization
## owning this repository, the people on https://github.com/orgs/xwiki/people, for whom building
## their code with the token grants nothing they could not already reach. Every other fork keeps the
## Checkstyle-only verdict quality-pr.yml gives it.
##
## Two consequences of pull_request_target are worth knowing. It runs the copy of this file held by
## the base branch and not the one in the pull request, so a change to this file cannot be tried out
## in a pull request and takes effect only once merged -- and a pull request targeting a branch that
## does not carry this file gets no run at all. And what to check out has to be said explicitly, the
## default being the base branch, which holds none of the change.
name: Pull request quality checks (fork)

on:
  pull_request_target:
    types: [opened, synchronize, reopened]

## One run per pull request: a new commit makes the running one obsolete.
concurrency:
  group: quality-pr-sonar-${{ github.event.pull_request.number }}
  cancel-in-progress: true

jobs:
  analysis:
    ## Named as in quality-pr.yml, and for the same reason: a reusable workflow names its check
    ## "<this job> / <the called job>", exactly one of the two workflows runs for a given pull
    ## request, and the check to require should be called the same whichever one it was.
    name: Quality
    ## A fork whose author GitHub reports as a member of the organization owning this repository.
    ## author_association is computed by GitHub and is no part of what the pull request's author
    ## sends, so it cannot be claimed by crafting anything; it is also the only way to ask this
    ## question here, a workflow's GITHUB_TOKEN being unable to read organization membership and a
    ## token that could being one more secret to hold. It covers members whose membership is private,
    ## which this organization has. COLLABORATOR, someone granted access to this repository alone, is
    ## deliberately left out: that access can be read-only, and read-only is not the "could have
    ## pushed this branch themselves" that this guard stands in for.
    ## This condition is the exact negation of quality-pr.yml's, so that a pull request is built once
    ## and not twice; changing one of the two means changing the other.
    if: >-
      github.event.pull_request.head.repo.full_name != github.repository
      && contains(fromJSON('["OWNER", "MEMBER"]'),
                  github.event.pull_request.author_association)
    ## For a reusable workflow the GITHUB_TOKEN permissions are the caller's. The analysis only reads
    ## the sources, and narrowing them here is also what keeps this pull_request_target run from
    ## carrying the writable token it would otherwise be handed.
    permissions:
      contents: read
    ## Referenced by branch and not by commit hash, on purpose: pinning would mean a pull request in
    ## every repository for every change to the shared workflow, which is the duplication this
    ## indirection removes. What pinning protects against, a third party repointing the ref under
    ## us, does not apply to a repository the XWiki committers own themselves.
    uses: xwiki/.github/.github/workflows/quality-pr.yml@master
    with:
      ## refs/pull/<number>/head is a ref of this repository that GitHub maintains itself, so the
      ## fork's commit is reached without configuring a remote of the fork or fetching from it.
      ref: refs/pull/${{ github.event.pull_request.number }}/head
      sonar: true
    secrets:
      SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
```

Both carry the LGPL header the XWiki repositories put on every file; it is left out above.

The shared workflow takes three optional inputs:

- `maven-profiles` (default `snapshot,legacy`): the profiles every Maven command of the analysis runs
  with. They have to match the profiles the ci.xwiki.org "quality" build analyzes `master` with, or a
  module the pull request gate never sees ends up analyzed only after the merge.
- `sonar` (default `false`): whether to run the SonarQube half on top of the Checkstyle one. False is
  the shape a run with no `SONAR_TOKEN` has to take.
- `ref` (default empty, meaning the default of the event): what to check out. A `pull_request` run
  defaults to the pull request merged into its base, which is what deserves the verdict; a
  `pull_request_target` run defaults to the base branch, which holds none of the change, so that stub
  passes `refs/pull/<number>/head` — a ref of the repository itself that GitHub maintains, so nothing
  is fetched from the fork.

The `SONAR_TOKEN` secret is optional and needed only when `sonar` is true.

Two properties of `pull_request_target` are worth remembering when touching `quality-pr-sonar.yml`.
It runs the copy of that file held by the **base branch**, not the one in the pull request, so a
change to it cannot be tried out in a pull request and takes effect only once merged — and a pull
request targeting a branch that does not carry the file gets no run at all, so a stable branch needs
it backported before fork pull requests against it are analyzed. It is also handed a writable
`GITHUB_TOKEN` unless its caller narrows the permissions, which is why the stub sets
`contents: read` and why the shared workflow checks out with `persist-credentials: false`.

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
