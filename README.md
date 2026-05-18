# npm-bump
[![GitHub tag (latest SemVer)](https://img.shields.io/github/v/tag/bcomnes/npm-bump)](https://github.com/bcomnes/npm-bump)
[![npm bump](https://github.com/bcomnes/npm-bump/actions/workflows/release.yml/badge.svg)](https://github.com/bcomnes/npm-bump/actions/workflows/release.yml)
[![Marketplace link](https://img.shields.io/badge/github%20marketplace-npm--bump-brightgreen)](https://github.com/marketplace/actions/npm-bump)

`npm version {major,minor,patch}` && `npm publish` as an action. Full [npm lifecycle](https://docs.npmjs.com/misc/scripts) support with [releasearoni](https://github.com/bcomnes/releasearoni) for GitHub release creation. Opinionated and has a few assumptions.

## Usage

npm-bump uses [npm trusted publishing](https://github.blog/changelog/2025-07-31-npm-trusted-publishing-with-oidc-is-generally-available/) via OIDC — no npm token needed. Set up a trusted publisher on [npmjs.com](https://npmjs.com) for your package pointing at your repository and workflow file, then use this workflow:

``` yaml
name: Version and Release

on:
  workflow_dispatch:
    inputs:
      version-type:
        description: 'Version type'
        type: choice
        options:
          - patch
          - minor
          - major
          - custom
        default: 'patch'
        required: true
      new-version:
        description: 'Custom version (only used when version-type is "custom")'
        required: false

env:
  FORCE_COLOR: 3

permissions:
  contents: write
  id-token: write  # required for OIDC trusted publishing

concurrency: # prevent concurrent releases
  group: npm-bump
  cancel-in-progress: true

jobs:
  version_and_release:
    runs-on: ubuntu-latest
    outputs:
      tagName: ${{ steps.npm-bump.outputs.release-tag }}
    steps:
    - uses: actions/checkout@v6
      with:
        # fetch full history so things like auto-changelog work properly
        fetch-depth: 0
    - name: Use Node.js
      uses: actions/setup-node@v6
      with:
        node-version-file: package.json  # requires engines.node in package.json
    - run: npm i
    - run: npm test
    - name: Version and publish to npm
      id: npm-bump
      uses: bcomnes/npm-bump@v3
      with:
        version-type: ${{ github.event.inputs['version-type'] }}
        new-version: ${{ github.event.inputs['new-version'] }}
        push-version-commit: true # if your prePublishOnly step pushes git commits, you can omit this input or set it to false.
        github-token: ${{ secrets.GITHUB_TOKEN }}
    - run: echo ${{ steps.npm-bump.outputs['release-tag'] }}
```

This will give you a push-button triggered action that runs `npm version {major,minor,patch}`, `git push --follow-tags` and finally `npm publish`.

It is advisable to set a `prePublishOnly` lifecycle hook that runs, at a minimum, git commit pushing, so that local runs of `npm version && npm publish` will push the version commits to git the same way as this action will.

```json
{
  "scripts": {
    "prepublishOnly": "git push --follow-tags"
  }
}
```

With that lifecycle set, you can omit the `push-version-commit` input, or set it to false.

The following dependencies and npm lifecycle scripts are recommended for a fully automated release process that includes:

- changelog generation
- github release creation with changelog contents
- automated action based package publishing
- parity with a local release process (you can still run npm version && npm publish and get all of the above benefits)

```json
{
  "devDependencies": {
    "releasearoni": "^0.1.0"
  },
  "scripts": {
    "version": "releasearoni version",
    "prepublishOnly": "releasearoni"
  }
}
```

Additionally, you should run your tests in order to block a release that isn't passing.  Automate only when green.

### Inputs

- `version-type` (Optional): Dropdown value from `workflow_dispatch` — one of `major`, `minor`, `patch`, `custom`. When provided, the action resolves the version internally. Use `custom` together with `new-version` for an explicit version string.
- `new-version` (Optional): Explicit version string (e.g. `1.2.3`) or bump type. Required when `version-type` is `custom` or when `version-type` is not set.
- `git-email` (Optional): Email for the version commit. Defaults to `<actor>@users.noreply.github.com`.
- `git-username` (Optional): Name for the version commit. Defaults to `github.actor`.
- `push-version-commit` (Default: `false`): Run `git push --follow-tags` after `npm version`. Enable if you don't push in a `prepublishOnly` hook.
- `publish-cmd` (Default: `npm publish`): Command to run after `npm version`. Override to skip registry publishing or run a custom release script.
- `github-token`: Pass `secrets.GITHUB_TOKEN` to enable GitHub release creation via releasearoni.
- `major-branch` (Default: `false`): After publishing, create or force-reset a floating major version branch (e.g. `v1`, `v2`) to the release commit and force-push it to origin. Enables consumers to pin to a major version ref (`uses: you/action@v3`) and receive updates automatically. Requires `contents: write` permission.

### Outputs

- `release-tag`: The name of the created git tag as described by git describe --tags

## FAQ

### I keep getting "Git working directory not clean" errors

Something about your workflow is creating or modifying files before versioning.

npm-bump runs `git status --short --branch` automatically before `npm version`, so the step output in your Actions log will show exactly which files are dirty.

Things to check for:

- For files that get modified during the version step, you can stage them alongside your release in the `version` lifecycle script.

### I'm getting 404/bad auth errors on npm.  Why?

Make sure you have configured a [trusted publisher](https://github.blog/changelog/2025-07-31-npm-trusted-publishing-with-oidc-is-generally-available/) on npmjs.com for your package, and that your workflow has `id-token: write` permission. npm trusted publishing requires npm CLI v11.5.1 or later.

If you have local `.npmrc` modifications in your workflow they can interfere with publishing.

### Can I publish to the GitHub Packages registry?

Set `registry-url: 'https://npm.pkg.github.com'` on `actions/setup-node` and override `publish-cmd` if needed. GitHub Packages does not support OIDC trusted publishing, so you will need a token-based approach for that registry.

### Can I consume private GitHub packages from other repos?

Yes, but you need a Personal Access Token with `packages:read` stored as an action secret, configured in your `.npmrc`.

### Do I have to publish to a registry?

Nope, you can completely override the `npm publish` command with whatever you want (e.g. `npm run release` which can run whatever you want related to a release.)  This enables you to publish to things like the Github marketplace, create github releases etc.

### Can you offer a major version tag/branch alias?  I want automatic updates!

Yes. Set `major-branch: true` and npm-bump will create or force-reset a branch named after the major version (e.g. `v1`, `v2`, `v3`) and force-push it to origin after every release. Consumers can then reference your action as `uses: you/action@v3` and automatically receive all patch and minor updates within that major version.

```yaml
    - name: Version and publish to npm
      uses: bcomnes/npm-bump@v3
      with:
        version-type: ${{ github.event.inputs['version-type'] }}
        new-version: ${{ github.event.inputs['new-version'] }}
        push-version-commit: true
        github-token: ${{ secrets.GITHUB_TOKEN }}
        major-branch: true
```

The branch is force-pushed, so no history is retained on it — it is purely a floating pointer to the latest release commit for that major version.

### Why isn't npm-bump running tests anymore?

Testing node stuff is usually straight forward (`npm test`).  Sometimes though, its not.  Leaving the test responsibilities to the consumer makes the action file more clear.

### Why isn't npm-bump installing dependencies anymore?

Some projects don't need dependencies to release.  Sometimes install steps require external system dependencies.  It was decided to keep the install step inside the action consumer scope in order to increase clarity around what is happening.

## License

MIT
