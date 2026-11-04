<p align="center">
  <a href="https://github.com/rubygems/configure-rubygems-credentials/actions">
  <img alt="configure-rubygems-credentials-action status" src="https://github.com/rubygems/configure-rubygems-credentials/workflows/build-test/badge.svg">
  </a>
</p>

## Configure RubyGems Credentials for GitHub Actions

This Ethos Link fork adds the optional `owner` input required for the first
publication of a new gem. It retains the upstream credential handling and also
works for subsequent releases of existing gems.

Use this pinned configuration as the baseline for either case:

```yaml
permissions:
  contents: read
  id-token: write

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - name: Configure RubyGems credentials
        uses: ethos-link/configure-rubygems-credentials@9b2cf2e75bb26157e8b9909ce82f2736929cc571
        with:
          owner: "12345"
```

Replace the owner with the immutable numeric account ID shown on RubyGems.org.
The ID is public configuration, not a secret. New gems also need an unexpired
pending trusted publisher matching the calling repository, workflow and any
configured environment. Existing gems need their trusted publisher configured.
Configuring credentials does not publish a gem; run your build and `gem push`
steps separately. This action stays in the caller's release workflow, so changing
the credentials action does not require changing the publisher's workflow name.

The remaining examples document the original upstream action. Its current
release does not yet support `owner`; use the pinned fork above when supplying it.

Configure your RubyGems credentials and for use in other
GitHub Actions. This action implements OIDC support, writes gem credentials files,
and exports environment variables used by both `rubygems` and
`bundler` for your other Actions to use.

### Table of Contents

<!-- toc -->

- [Usage](#usage)
  - [Trusted Publisher (recommended)](#trusted-publisher-recommended)
  - [OIDC API Key Role](#oidc-api-key-role)
  - [Static API token in repository secrets](#static-api-token-in-repository-secrets)
  - [Use with the RubyGems CLI](#use-with-the-rubygems-cli)
- [License Summary](#license-summary)
- [Security Disclosures](#security-disclosures)

<!-- tocstop -->

## Usage

There are three ways to configure RubyGems credentials:

1. **Trusted Publisher (recommended)**: Uses OIDC without any API tokens or secrets.
2. **OIDC API Key Role**: Uses OIDC with a pre-configured API Key Role on RubyGems.org.
3. **Static API token**: Uses a RubyGems API token stored in repository secrets.

> **Note**: The OIDC-based configurations (**Trusted Publisher** and **OIDC API Key Role**) require the `id-token: write` permission in your workflow, for example:
>
> ```yaml
> permissions:
>   id-token: write
>   contents: read
> ```

### Trusted Publisher (recommended)

The simplest approach is to use [Trusted Publishing](https://guides.rubygems.org/trusted-publishing/).
Configure a trusted publisher for your gem on RubyGems.org, then use:

```yaml
- name: Configure RubyGems Credentials
  uses: rubygems/configure-rubygems-credentials@main
```

The action automatically uses OIDC to authenticate with RubyGems.org. For a new
gem, configure a pending trusted publisher and declare the account that will own
it using the `owner` input. Use the immutable account ID shown on your pending
trusted publishers page:

```yaml
- name: Configure RubyGems Credentials
  uses: ethos-link/configure-rubygems-credentials@9b2cf2e75bb26157e8b9909ce82f2736929cc571
  with:
    owner: "12345"
```

The same configuration works for subsequent releases. Existing gems can continue
to omit `owner`. Handles are also accepted, but numeric IDs remain stable if an
account changes its handle.

### OIDC API Key Role

Alternatively, you can create an OIDC API Key Role on RubyGems.org and reference it
with the `role-to-assume` input. The value is the **OIDC API Key Role token**
(a string starting with `rg_oidc_akr_`), which you can find on the
"OIDC: Create" page of your gem on RubyGems.org.

```yaml
- name: Configure RubyGems Credentials
  uses: rubygems/configure-rubygems-credentials@main
  with:
    role-to-assume: rg_oidc_akr_f55fe1127adjkkcn8ty6
```

You can use this action with the `rubygems` or `bundler` command line tools,
or run this action multiple times
to use different RubyGems.org accounts or OIDC API Key roles in the same GitHub Actions
workflow. As an example, here is a complete workflow file that pushes a gem release.

```yaml
on:
  push:
    tags: ['v*']

jobs:
  job:
    runs-on: ubuntu-latest
    permissions:
      contents: write
      id-token: write
    steps:
      - uses: rubygems/configure-rubygems-credentials@main
        with:
          role-to-assume: rg_oidc_akr_f55fe1127adjkkcn8ty6
      - uses: actions/checkout@v6
      - name: Set remote URL
        run: |
          git config --global user.email "$(git log -1 --pretty=format:'%ae')"
          git config --global user.name "$(git log -1 --pretty=format:'%an')"
          git remote set-url origin "https://x-access-token:${{ secrets.GITHUB_TOKEN }}@github.com/$GITHUB_REPOSITORY"
      - name: Set up Ruby
        uses: ruby/setup-ruby@v1
        with:
          ruby-version: '4.0.5'
          bundler-cache: false
      - name: Install dependencies
        run: bundle install
      - name: Release
        run: bundle exec rake release
```

See [action.yml](action.yml) for the full documentation for this action's inputs
and outputs.

> [!NOTE]
> It is recommended that you replace the version ranges in the Actions with specific SHA hashes to ensure that the workflow does not break with updates, and does not pick up a new version if any of these Actions were compromised.

### Static API token in repository secrets

```yaml
- name: Configure RubyGems Credentials
  uses: rubygems/configure-rubygems-credentials@main
  with:
    api-token: ${{ secrets.RUBYGEMS_API_TOKEN }}
```

In this example, the secret `RUBYGEMS_API_TOKEN` contains a string like `rubygems_1a072a969ecdd84bb190c3c218e13e3c6f5d419f3f0f5b22`.

### Use with the RubyGems CLI

This workflow does _not_ install the rubygems CLI
into your environment.

## License Summary

This code is made available under the MIT license.

## Security Disclosures

If you would like to report a potential security issue in this project, please do not create a GitHub issue. Instead, please follow the instructions [here](https://rubygems.org/pages/security) or [email the RubyGems security team](mailto:security@rubygems.org).
