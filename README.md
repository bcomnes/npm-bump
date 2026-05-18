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
      version_type:
        description: 'Version type'
        type: choice
        options:
          - patch
          - minor
          - major
          - custom
        default: 'patch'
        required: true
      newversion:
        description: 'Custom version (only used when version_type is "custom")'
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
      tagName: ${{ steps.npm-bump.outputs.release_tag }}
    steps:
    - uses: actions/checkout@v6
      with:
        # fetch full history so things like auto-changelog work properly
        fetch-depth: 0
    - name: Use Node.js
      uses: actions/setup-node@v6
      with:
        node-version-file: package.json  # requires engines.node in package.json
        registry-url: 'https://registry.npmjs.org'
    - run: npm i
    - run: npm test
    - name: Version and publish to npm
      id: npm-bump
      uses: bcomnes/npm-bump@v2
      with:
        version_type: ${{ github.event.inputs.version_type }}
        newversion: ${{ github.event.inputs.newversion }}
        push_version_commit: true # if your prePublishOnly step pushes git commits, you can omit this input or set it to false.
        github_token: ${{ secrets.GITHUB_TOKEN }}
    - run: echo ${{ steps.npm-bump.outputs.release_tag }}
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

With that lifecycle set, you can omit the `push_version_commit` input, or set it to false.

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

- `version_type` (Optional): Dropdown value from `workflow_dispatch` — one of `major`, `minor`, `patch`, `custom`. When provided, the action resolves the version internally. Use `custom` together with `newversion` for an explicit version string.
- `newversion` (Optional): Explicit version string (e.g. `1.2.3`) or bump type. Required when `version_type` is `custom` or when `version_type` is not set.
- `git_email` (Optional): Email for the version commit. Defaults to `<actor>@users.noreply.github.com`.
- `git_username` (Optional): Name for the version commit. Defaults to `github.actor`.
- `push_version_commit` (Default: `false`): Run `git push --follow-tags` after `npm version`. Enable if you don't push in a `prepublishOnly` hook.
- `publish_cmd` (Default: `npm publish`): Command to run after `npm version`. Override to skip registry publishing or run a custom release script.
- `github_token`: Pass `secrets.GITHUB_TOKEN` to enable GitHub release creation via releasearoni.

### Outputs

- `release_tag`: The name of the created git tag as described by git describe --tags

## FAQ

### I keep getting "Git working directory not clean" errors

Something about your workflow is creating or modifying files before versioning.

npm-bump runs `git status --short --branch` automatically before `npm version`, so the step output in your Actions log will show exactly which files are dirty.

Things to check for:

- Is your `package-lock.json` (or equivalent) getting modified before versioning? Consider adding it to `.gitignore` — lock files provide a less [realistic environment](https://github.com/sindresorhus/ama/issues/479#issuecomment-310661514) in published modules.
- For files that get modified during the version step, you can stage them alongside your release in the `version` lifecycle script.

### I'm getting 404/bad auth errors on npm.  Why?

Make sure you have configured a [trusted publisher](https://github.blog/changelog/2025-07-31-npm-trusted-publishing-with-oidc-is-generally-available/) on npmjs.com for your package, and that your workflow has `id-token: write` permission. npm trusted publishing requires npm CLI v11.5.1 or later.

You must also set `registry-url: 'https://registry.npmjs.org'` on `actions/setup-node`. If you have local `.npmrc` modifications in your workflow they can interfere with this.

### Can I publish to the GitHub Packages registry?

Set `registry-url: 'https://npm.pkg.github.com'` on `actions/setup-node` and override `publish_cmd` if needed. GitHub Packages does not support OIDC trusted publishing, so you will need a token-based approach for that registry.

### Can I consume private GitHub packages from other repos?

Yes, but you need a Personal Access Token with `packages:read` stored as an action secret, configured in your `.npmrc`.

### Do I have to publish to a registry?

Nope, you can completely override the `npm publish` command with whatever you want (e.g. `npm run release` which can run whatever you want related to a release.)  This enables you to publish to things like the Github marketplace, create github releases etc.

### Can you offer a major version tag/branch alias?  I want automatic updates!

Yes. npm-bump now offers a major version ref you can install with.

### Why isn't npm-bump running tests anymore?

Testing node stuff is usually straight forward (`npm test`).  Sometimes though, its not.  Leaving the test responsibilities to the consumer makes the action file more clear.

### Why isn't npm-bump installing dependencies anymore?

Some projects don't need dependencies to release.  Sometimes install steps require external system dependencies.  It was decided to keep the install step inside the action consumer scope in order to increase clarity around what is happening.

## License

MIT
