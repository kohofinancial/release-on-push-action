# Github Release On Push Action

> [!NOTE]
> We don't use this to create releases any longer; releases are created directly in the
> reusable CI/CD workflow.

> Stop using files for versioning. Use git tags instead!

Github Action to create a Github Release on pushes to master.

## Features

- Flexible version bumping scheme with a project default or overrides using Pull Request Labels
- Creates Release Notes automatically (with a list of commits since the last release)

## Rationale

CI & CD systems are simpler when they work with immutable monotonic identifers
from the get-go. Trigger your release activites by subscribing to new tags
pushed from this Action.

For automation, Github Releases (and by extension git tags) are better than
versioned commit files for these reasons:

- Agnostic of language & ecosystem (i.e. does not rely on presence of package.json)
- Tagging does not require write permissions to bump version
- Tagging does not trigger infinite-loop webhooks from pushing version bump commits

## Example Workflow

``` yaml
on: 
  push:
    branches:
      - master

jobs:
  release-on-push:
    runs-on: ubuntu-latest
    env:
      GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
    steps:
      - uses: rymndhng/release-on-push-action@master
        with:
          bump_version_scheme: minor
```

Allowed values of `bump_version_scheme`:

- minor
- major
- patch
- **norelease**: Performs no release by default. Creation of release delegated to labels on Pull Requests.

For stability, we recommend pinning the version of the action. See [Releases](https://github.com/rymndhng/release-on-push-action/releases).

See [action.yml](./action.yml) for the full list of options.

## FAQ

### Can I skip creation of a release?

There are several approaches:

1. Put `[norelease]` in the commit title.
2. If the commit has an attached PR, add the label `norelease` to the PR.
3. Set the action's `bump_version_scheme` to `norelease` to disable this behavior by default

### How do I change the bump version scheme using Pull Requests?

Iif the PR has the label `release:major`, `release:minor`, or `release:patch`, this will override `bump_version_scheme`. 

This repository's pull requests are an example of this in action. For example, [#19](https://github.com/rymndhng/release-on-push-action/pull/19).

Only one of these labels should be present on a PR. If there are multiple, the behavior is undefined.

### Do I need to setup Github Action access tokens or any other permission-related thing?

Github Actions will inject a token for this plugin to interact with the API.

You need to specify the following permission scopes for this action to work correctly:

```yaml
permissions:
  contents: write
  pull-requests: read
```

Alternatively, you can enable `read and write permission` to this token under `settings > actions > general > workflow permissions` (disabled by default), but beware that this provides all of the actions in your repository with read and write access to all scopes.

### Can I create a tag instead of a release?

Currently, no.

In order to reliably generate monotonic versions, we use Github Releases to
track what the last release version is. See [Release#get-the-latest-release](https://developer.github.com/v3/repos/releases/#get-the-latest-release).

### How can I get the generated tag or version?

On a successful release, this action creates an output parameters `tag_name`, `upload_url` and `version`. 

Example of how to consume this:

``` yaml
on: 
  push:
    branches:
      - master

jobs:
  release-on-push:
    runs-on: ubuntu-latest
    env:
      GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
    steps:
      - id: release
        uses: rymndhng/release-on-push-action@master
        with:
          bump_version_scheme: minor
          tag_prefix: v
          
      - name: Check Output Parameters
        run: |
          echo "Got tag name ${{ steps.release.outputs.tag_name }}"
          echo "Got release version ${{ steps.release.outputs.version }}"
          echo "Upload release artifacts to ${{ steps.release.outputs.upload_url }}"
```

### Can I customize the release body message?

Yes you can, with `release_body`. The contents of this be appended to the release description.

Example:

``` yaml
on:
  push:
    branches:
      - master

jobs:
  release-on-push:
    runs-on: ubuntu-latest
    env:
      GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
    steps:
      - uses: rymndhng/release-on-push-action@master
        with:
          bump_version_scheme: minor
          release_body: "When set, adds extra text to body!"
```

### Can I use Github's Generated Release Notes?

Yes you can, by setting `use-github-release-notes` to `true`. For more information, see [Automatic Release Notes](https://docs.github.com/en/repositories/releasing-projects-on-github/automatically-generated-release-notes).

Note, if enabling this feature:
- If you have set `release_body`, the `release_body` will be prepended to the Generated Release Notes
- Enabling this will disable the commit summary generated by this project

Example:

``` yaml
on:
  push:
    branches:
      - master

jobs:
  release-on-push:
    runs-on: ubuntu-latest
    env:
      GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
    steps:
      - uses: rymndhng/release-on-push-action@master
        with:
          bump_version_scheme: minor
          use_github_release_notes: true
```


### Can I change the prefix `v` from the Git Tags?

Yes, you can customize this by changing the `tag_prefix`. Here's an example of
removing the prefix by using an empty string.
 
``` yaml
on:
  push:
    branches:
      - master

jobs:
  release-on-push:
    runs-on: ubuntu-latest
    env:
      GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
    steps:
      - uses: rymndhng/release-on-push-action@master
        with:
          tag_prefix: ""
```

### Can I change the name of the Release?

Yes, you can customize this by changing the `release_name`. The `release_name` supports these template variables:

- `<RELEASE_VERSION>` contains the version number, i.e. `1.2.3`
- `<RELEASE_TAG>` contains the git tag name, i.e. `v1.2.3`

See example below for how to create a release with the name `Release 1.2.3`

``` yaml
on:
  push:
    branches:
      - master

jobs:
  release-on-push:
    runs-on: ubuntu-latest
    env:
      GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
    steps:
      - uses: rymndhng/release-on-push-action@master
        with:
          tag_prefix: "v"
          release_name: "Release <RELEASE_VERSION>"
```

### How can I configure the maximum number of commits to summarize?

Use the option `max_commits`. The default value is 50.

``` yaml
on:
  push:
    branches:
      - master

jobs:
  release-on-push:
    runs-on: ubuntu-latest
    env:
      GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
    steps:
      - uses: rymndhng/release-on-push-action@master
        with:
          max_commits: 100
```

### How can I select a commit SHA different from `GITHUB_SHA`?

By default, the release is created on the commit that triggered the workflow, as identified by the `GITHUB_SHA` environment variable. Sometimes, that's not the desired behaviour; for example, the `workflow_run` event sets `GITHUB_SHA` to the last commit on the default branch, but we might want to tag the commit that was the most recent one when the workflow started.

A different SHA can be selected using the `sha` option. If it is unset, `GITHUB_SHA` is used.

Example usage for a release triggered by a (potentially long running) workflow run:

```yaml
on:
  workflow_run:
    workflows:
      - Slow CI test workflow
    types:
      - completed
    branches:
      - master

jobs:
  release-on-push:
    if: github.event.workflow_run.conclusion == 'success'
    runs-on: ubuntu-latest
    env:
      GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
    steps:
      - uses: rymndhng/release-on-push-action@master
        with:
          sha: ${{ github.event.workflow_run.head_sha }}
```

## Development

Uses [babashka](https://github.com/borkdude/babashka)

To run tests:

1. Install [babashka](https://github.com/borkdude/babashka#installation).
2. Install [direnv](https://direnv.net/docs/installation.html)
3. Create a `.envrc` file at the project root from this template:

``` sh
export GITHUB_TOKEN=<YOUR_TOKEN>
export GITHUB_REPOSITORY=rymndhng/release-on-push-action
export GITHUB_SHA=167c690247d0933acde636d72352bcd67e33724b
export GITHUB_API_URL=https://api.github.com
export INPUT_BUMP_VERSION_SCHEME=minor
export INPUT_MAX_COMMITS=5
export INPUT_TAG_PREFIX=v
export INPUT_RELEASE_NAME="<RELEASE_TAG>"
export INPUT_DRY_RUN=1
```

2. Run Tests

``` sh
make test
make dryrun
```


## Big Thanks To

- shell-semver: https://github.com/fmahnke/shell-semver
- https://github.com/mikeal/publish-to-github-action/
- Inspiration: https://github.com/roomkey/lein-v
