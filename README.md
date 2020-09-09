# npm-bump
[![GitHub tag (latest SemVer)](https://img.shields.io/github/v/tag/bcomnes/npm-bump)](https://github.com/bcomnes/npm-bump)
[![Version and Release](https://github.com/bcomnes/npm-bump/workflows/Version%20and%20Release/badge.svg)](https://github.com/bcomnes/deploy-to-neocities/actions)
[![Marketplace link](https://img.shields.io/badge/github%20marketplace-npm--bump-brightgreen)](https://github.com/marketplace/actions/npm-bump)

`npm i` && `npm test` && `npm version {major,minor,patch}` && `npm publish` as an action.  Full [npm lifecycle](https://docs.npmjs.com/misc/scripts) support and [gh-release](https://ghub.io/gh-release) auth support.  Opinionated and has a few assumptions.

## Usage

Generate a publish token on npm then set it as an action secret (`NPM_TOKEN` in this example).

``` yaml
name: Version and Release

on:
  workflow_dispatch:
    inputs:
      newversion:
        description: 'Semantic Version Bump Type (major minor patch)'
        required: true

env:
  node_version: 14

jobs:
  version_and_release:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v2.3.2
      with:
        # fetch full history so things like auto-changelog work properly
        fetch-depth: 0
    - name: Use Node.js ${{ env.node_version }}
      uses: actions/setup-node@v2.1.1
      with:
        node-version: ${{ env.node_version }}
        # setting a registry enables the NODE_AUTH_TOKEN env variable where we can set an npm token.  REQUIRED
        registry-url: 'https://registry.npmjs.org'
    - name: Test, version and publush to npm
      uses: bcomnes/npm-bump@v1.0.0
      with:
        git_email: bcomnes@gmail.com
        git_username: ${{ github.actor }}
        newversion: ${{ github.event.inputs.newversion }}
        push_version_commit: true # if your prePublishOnly step pushes git commits, you can omit this input or set it to false.
        github_token: ${{ secrets.GITHUB_TOKEN }} # built in actions token.  Passed tp gh-release if in use.
        npm_token: ${{ secrets.NPM_TOKEN }} # user set secret token generated at npm
```

This will give you a push-button triggered action that runs `npm i`, `npm test`, `npm version {major,minor,patch}`, `git push --follow-tags` and finally `npm publish`.

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
- See [swyx's](https://dev.to/swyx/semi-automatic-npm-and-github-releases-with-gh-release-and-auto-changelog-4b5a) article for a more in depth description.

```json
{
  "devDependencies": {
    "auto-changelog": "^1.16.2",
    "gh-release": "^3.5.0"
  },
  "scripts": {
    "prepublishOnly": "git push --follow-tags && gh-release -y",
    "version": "auto-changelog -p --template keepachangelog auto-changelog --breaking-pattern 'BREAKING CHANGE:' && git add CHANGELOG.md"
  }
}
```

### Inputs

- `git_email` (**REQUIRED**): The email address used to create the version commit with.
- `git_username` (**REQUIRED**): The name to use for the version commit. e.g. github.actor
- `newversion` (**REQUIRED**): The version bump type to perform (e.g. major, minor, path). See npm version docs for more info.  Pass this as an interactive variable.
- `push_version_commit` (Default: `false`): Run `git push --follow-tags` after running `npm version`.  Enable this if you don't configure a prepublishOnly hook that pushes git commits.
- `skip_test` (Default: `false`): 'Skip npm test step.  Usually a bad idea, but sometimes not.'
- `publish_cmd` (Default: `npm publish`): The command to run after npm version.  Useful if you are just using npm to version a package, but not publish to npm (like an action).
- `github_token`: Pass the secrets.GITHUB_TOKEN to enable gh-release capabilities.
- `npm_token`: An npm token scoped for publishing.  Required in most cases.  Used to create the release.

### Outputs

None.

## FAQ

### I'm getting 404/bad auth errors on npm.  Why?

You must set the `registry-url` input on the `actions/setup-node` action to 'https://registry.npmjs.org' at a minimum.  Github actions does some wacky stuff to `.npmrc` like setting up a `NODE_AUTH_TOKEN` input for the npm token.  `npm-bump` takes advantage of this behavior so its an assumed requirement. See [this article](https://docs.github.com/en/actions/language-and-framework-guides/publishing-nodejs-packages) for more info on this bizarre behavior.   Also if you script modifications to a local `.npmrc`, this can mess up the `actions/setup-node` configuration.

### Can I publish to the github registry?

Yes, just pass `secrets.GITHUB_TOKEN` as the `npm_token` input, and set your registry endpoint to `https://npm.pkg.github.com` in the `actions/setup-node` action.

### Can publish to both npm and github?

No, not right now.  I couldn't think of why this would be a good reason.  Open an issue if you have ideas.

### Do I have to publish to a registry?

Nope, you can completely override the `npm publish` command with whatever you want (e.g. `npm run release` which can run whatever you want related to a release.)  This enables you to publish to things like the Github marketplace, create github releases etc.

### Can you offer a major version tag/branch alias?  I want automatic updates!

Nope!  This was always weird/bad pattern of github actions.  Luckily github offers a solution for this.  Create a `.github/dependabot.yml` with, at a minimum, the following config:

```yaml
# Basic dependabot.yml file with
# minimum configuration for two package managers

version: 2
updates:
  # Enable version updates for npm
  # Enable updates to github actions
  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
      interval: "daily"

```

## License

MIT
