# Quickstart

By default, every time you merge a commit to the main branch, the
[GitHub Action](https://github.com/marketplace/actions/release-plz)
runs two commands, one after the other:

- [`release-plz release-pr`](../usage/release-pr.md): creates the release pr.
- [`release-plz release`](../usage/release.md): publishes the unpublished packages.

Follow the steps below to set up the GitHub Action.

## 1. Change GitHub Actions permissions

1. Go to the GitHub Actions settings:

   ![actions settings](../assets/actions_settings.png)

2. Change "Workflow permissions" to allow GitHub Actions to create and approve
   pull requests (needed to create and update the PR).

   ![workflow permission](../assets/workflow_permissions.png)

## 2. Set the `CARGO_REGISTRY_TOKEN` secret

Release-plz needs a token to publish your packages to the cargo registry.

1. Retrieve your registry token following
   [this](https://doc.rust-lang.org/cargo/reference/publishing.html#before-your-first-publish)
   guide.
2. Add your cargo registry token as a secret in your repository following
   [this](https://docs.github.com/en/actions/security-guides/encrypted-secrets#creating-encrypted-secrets-for-a-repository)
   guide.

As specified in the `cargo publish`
[options](https://doc.rust-lang.org/cargo/commands/cargo-publish.html#publish-options):

- The token for [crates.io](https://crates.io/) shall be specified with the `CARGO_REGISTRY_TOKEN`
  environment variable.
- Tokens for other registries shall be specified with environment variables of the form
  `CARGO_REGISTRIES_NAME_TOKEN` where `NAME` is the name of the registry in all capital letters.

If you are creating a new crates.io token, specify the following scope:

![token scope](../assets/token_scope.png)

## 3. Setup the workflow

Add the release-plz workflow file under the `.github/workflows` directory.
For example `.github/workflows/release-plz.yml`.

Use one of the following examples as a starting point.

### Example: release-pr and release

This is the suggested configuration if you are getting started with release-plz.
With this configuration, when you make changes to the `main` branch:

- release-plz creates a pull request with the new versions,
  where it prepares the next release.
- release-plz releases the unpublished packages.

```yaml
name: Release-plz

permissions:
  pull-requests: write
  contents: write

on:
  push:
    branches:
      - main

jobs:
  release-plz:
    name: Release-plz
    runs-on: ubuntu-latest
    concurrency:
      group: release-plz-${{ github.ref }}
      cancel-in-progress: false
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - name: Install Rust toolchain
        uses: dtolnay/rust-toolchain@stable
      - name: Run release-plz
        uses: MarcoIeni/release-plz-action@v0.5
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          CARGO_REGISTRY_TOKEN: ${{ secrets.CARGO_REGISTRY_TOKEN }}
```

Notes:

- `fetch-depth: 0` is needed to clone all the git history, which is necessary to
  determine the next version and build the changelog.
- The `concurrency` block guarantees that if a new commit is pushed while the job of the previous
  commit was still running, the new job will wait for the previous one to finish.
  In this way, only one instance of release-plz will run in the repository at the same time for
  the same branch, ensuring that there are no conflicts.
  See the GitHub [docs](https://docs.github.com/en/actions/writing-workflows/workflow-syntax-for-github-actions#jobsjob_idconcurrency)
  to learn more.

### Example: release-pr only

Use this configuration if you want release-plz to only update your packages,
and you want to handle `cargo publish` and git tag push by yourself.

```yaml
name: Release-plz

permissions:
  pull-requests: write
  contents: write

on:
  push:
    branches:
      - main

jobs:
  release-plz:
    name: Release-plz
    runs-on: ubuntu-latest
    concurrency:
      group: release-plz-${{ github.ref }}
      cancel-in-progress: false
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - name: Install Rust toolchain
        uses: dtolnay/rust-toolchain@stable
      - name: Run release-plz
        uses: MarcoIeni/release-plz-action@v0.5
        with:
          command: release-pr
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

### Example: release only

Use this configuration if you want release-plz to only release your packages,
and you want to update `Cargo.toml` versions and changelogs by yourself.

```yaml
name: Release-plz

permissions:
  pull-requests: write
  contents: write

on:
  push:
    branches:
      - main

jobs:
  release-plz:
    name: Release-plz
    runs-on: ubuntu-latest
    concurrency:
      group: release-plz-${{ github.ref }}
      cancel-in-progress: false
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - name: Install Rust toolchain
        uses: dtolnay/rust-toolchain@stable
      - name: Run release-plz
        uses: MarcoIeni/release-plz-action@v0.5
        with:
          command: release
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          CARGO_REGISTRY_TOKEN: ${{ secrets.CARGO_REGISTRY_TOKEN }}
```

### Example: release-pr and release on schedule

In the above examples, release-plz runs every time you merge a commit to the `main` branch.

To run release-plz periodically, you can use the
[`schedule`](https://docs.github.com/en/actions/reference/events-that-trigger-workflows#schedule) event:

```yaml
name: Release-plz

permissions:
  pull-requests: write
  contents: write

# Trigger the workflow every Monday.
on:
  schedule:
    # * is a special character in YAML so you have to quote this string
    - cron:  '0 0 * * MON'

jobs:
  release-plz:
    name: Release-plz
    runs-on: ubuntu-latest
    concurrency:
      group: release-plz-${{ github.ref }}
      cancel-in-progress: false
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - name: Install Rust toolchain
        uses: dtolnay/rust-toolchain@stable
      - name: Run release-plz
        uses: MarcoIeni/release-plz-action@v0.5
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          CARGO_REGISTRY_TOKEN: ${{ secrets.CARGO_REGISTRY_TOKEN }}
```

## 4. Set input variables (optional)

The GitHub action accepts the following input variables:

- `command`: The release-plz command to run. Accepted values: `release-pr`,
  `release`. (By default it runs both commands).
- `registry`: Registry where the packages are stored.
  The registry name needs to be present in the Cargo config.
  If unspecified, the `publish` field of the package manifest is used.
  If the `publish` field is empty, crates.io is used.
- `manifest_path`: Path to the Cargo.toml of the project you want to update.
  Both Cargo workspaces and single packages are supported. (Defaults to the root
  directory).
- `version`: Release-plz version to use. E.g. `0.3.70`. (Default: latest version).
- `config`: Release-plz config file location. (Defaults to
  `release-plz.toml` or `.release-plz.toml`).
- `token`: Token used to publish to the cargo registry.
  Override the `CARGO_REGISTRY_TOKEN` environment variable, or the `CARGO_REGISTRIES_<NAME>_TOKEN`
  environment variable, used for registry specified in the `registry` input variable.
- `backend`: Forge backend. Valid values: `github`, `gitea`. (Defaults to `github`).

You can specify the input variables by using the `with` keyword.
For example:

```yaml
jobs:
  release-plz:
    name: Release-plz
    runs-on: ubuntu-latest
    concurrency:
      group: release-plz-${{ github.ref }}
      cancel-in-progress: false
    steps:
      - name: Checkout repository
        uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - name: Install Rust toolchain
        uses: dtolnay/rust-toolchain@stable
      - name: Run release-plz
        uses: MarcoIeni/release-plz-action@v0.5
        with: # <--- Input variables
          command: release-pr
          registry: my-registry
          manifest_path: rust-crates/my-crate/Cargo.toml
          version: 0.3.70
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          CARGO_REGISTRY_TOKEN: ${{ secrets.CARGO_REGISTRY_TOKEN }}
```
