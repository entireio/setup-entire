# setup-entire

Push to Entire from GitHub Actions without storing a secret.

The action installs `entire` and `git-remote-entire`. It then trades the job's GitHub OIDC token for a short-lived Entire service-account token and exports it as `ENTIRE_TOKEN`.

## One-time setup

The `entire sa` commands need `entire` v0.13.1 or later.

Create a service account in your project, give it a repository, and bind it to the GitHub repository that runs the workflow:

```bash
entire sa create my-project ci
entire sa grant my-project/ci --repo my-repo --role writer
entire sa bind my-project/ci --provider github-actions --pin repository=my-gh-org/my-gh-repo
```

Add `--pin ref=refs/heads/main` to allow only workflows on `main`. `entire sa providers github-actions` lists every pin.

`create` prints the `svc_` ID. You can do the same in the web app under Project settings > Service accounts.

A service account binds to one GitHub repository. Use one service account per workflow purpose.

## Usage

```yaml
permissions:
  id-token: write
  contents: read

jobs:
  push:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
        with:
          fetch-depth: 0
      - uses: entireio/setup-entire@v1
        with:
          service-account: svc_01M6JDV1WP4PTGV5WBVAQDX3K5
      # Use your repo's entire:// URL.
      - run: git push entire://aws-us-east-2.entire.io/et/my-project/my-repo HEAD:main
```

## Inputs

| Name | Default | Description |
| --- | --- | --- |
| `service-account` | none | `svc_<ulid>` to act as. Omit to install only. |
| `auth-url` | `https://us.auth.entire.io` | Entire auth server. Any region works for any repo. |
| `version` | pinned per release | `entireio/cli` release to install. |
| `install` | `true` | Put `entire` and `git-remote-entire` on `PATH`. |
| `export-env` | `true` | Export the token as `ENTIRE_TOKEN`. |

## Outputs

| Name | Description |
| --- | --- |
| `token` | The access token, masked in logs. |
| `expires-in` | Lifetime in seconds. |

The token lives 45 minutes. A job that runs longer must run the action again.

Runners: Linux and macOS, x64 and arm64. Windows is not supported yet.

## Pinning

Use `@v1` to get fixes. For a stronger supply-chain guarantee, pin the full commit SHA and let Dependabot update it:

```yaml
- uses: entireio/setup-entire@<40-char-sha> # v1.0.0
```

## Troubleshooting

- **No OIDC token**: GitHub gave the job no OIDC token. Either the workflow or job lacks `permissions: id-token: write`, or the run is a pull request from a fork. GitHub turns a fork run's write permissions into read, and `id-token` has no read level, so a fork run never gets a token.
- **Token exchange failed (HTTP 400)**: the service account has no binding, or a pin does not match the run. Check `repository` against `github.repository` and `ref` against `github.ref`.
- **Repository not found** at push time: the service account has no grant on that repo.

## Developing

CI runs on every pull request:

- `lint`: actionlint on the workflows, shellcheck on the action's scripts.
- `install`: installs the CLI on Linux and macOS runners.
- `auth`: runs the full exchange and an `ls-remote`. It needs these repository variables, so it skips when they are unset:
  - `ENTIRE_TEST_SERVICE_ACCOUNT`: a `svc_` ID bound to `entireio/setup-entire`, with read access to the test repo.
  - `ENTIRE_TEST_REMOTE`: an `entire://` URL the account can read.
  - `ENTIRE_TEST_AUTH_URL` (optional): defaults to `https://us.auth.entire.io`.

## Releasing

1. Bump `version`'s default in `action.yml` to the newest `entireio/cli` release.
2. Publish a GitHub release tagged `vX.Y.Z`. Turn on immutable releases for the repository.
3. Move the major tag: `git tag -f v1 vX.Y.Z && git push -f origin v1`.
4. Change inputs only in a new major version.
