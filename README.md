<div align="center">

# 🛠️ secrets-to-env-action

**GitHub Action that exports all repository secrets as environment variables.**

A TypeScript action (Glovo fork of [oNaiPs/secrets-to-env-action](https://github.com/oNaiPs/secrets-to-env-action)) that you call from a workflow step. It is not a service: nothing is deployed, and consumers pin it by tag.

[![TypeScript](https://img.shields.io/badge/TypeScript-5.3-3178C6?logo=typescript&logoColor=white)](package.json)
[![Node.js](https://img.shields.io/badge/Node.js-20_runtime-339933?logo=nodedotjs&logoColor=white)](action.yml)
[![Jest](https://img.shields.io/badge/Jest-29-C21325?logo=jest&logoColor=white)](package.json)
<br/>
[![Build](https://img.shields.io/badge/CI-build_%2B_e2e-2088FF?logo=githubactions&logoColor=white)](#cicd)
[![test files](https://img.shields.io/badge/test_files-1-blue)](__tests__/main.test.ts)
[![License](https://img.shields.io/badge/License-MIT-yellow)](LICENSE)

</div>

---

## Table of contents

- [Overview](#overview)
- [Build & local development](#build--local-development)
- [Testing](#testing)
- [Code quality](#code-quality)
- [CI/CD](#cicd)
- [Repository layout](#repository-layout)
- [Contributing & conventions](#contributing--conventions)
- [Ownership & support](#ownership--support)

---

## Overview

GitHub does not expose secrets as environment variables by default; each one has to be mapped by hand in `env:`. This action reads the JSON of all secrets (`${{ toJSON(secrets) }}`) and calls `core.exportVariable` for each one, so later steps in the job see them as env vars. You can include, exclude, prefix and rename them on the way.

<table>
<tr><th>Before</th><th>After</th></tr>
<tr>
<td>

```yaml
- run: echo "Value of MY_SECRET1: $MY_SECRET1"
  env:
    MY_SECRET1: ${{ secrets.MY_SECRET1 }}
    MY_SECRET2: ${{ secrets.MY_SECRET2 }}
    ...
```

</td>
<td>

```yaml
- uses: Glovo/secrets-to-env-action@v1
  with:
    secrets: ${{ toJSON(secrets) }}
- run: echo "Value of MY_SECRET1: $MY_SECRET1"
```

</td>
</tr>
</table>

| | |
|---|---|
| Language / build | TypeScript ([`src/main.ts`](src/main.ts)), compiled and bundled with `@vercel/ncc` into [`dist/index.js`](dist/index.js) |
| Runtime | `node20` ([`action.yml`](action.yml)) |
| Versions | Git tags `v1`, `v1.4`, `v1.5`, `v1.6` (`v1` is the moving major tag) |
| Origin | Fork of [oNaiPs/secrets-to-env-action](https://github.com/oNaiPs/secrets-to-env-action); the Glovo changes are the Node version bump in [`action.yml`](action.yml) |

> [!NOTE]
> The `secrets` input is **mandatory**. Without it the action cannot read the secrets. If the value is not valid JSON, the step fails with a message telling you to add `secrets: ${{ toJSON(secrets) }}`.

> [!NOTE]
> `github_token` is always excluded, because GitHub already exports it.

### Inputs

| Input | Required | Default | Meaning |
|---|---|---|---|
| `secrets` | yes | | JSON of all secrets, always `${{ toJSON(secrets) }}` |
| `prefix` | no | | Prefix added to every exported name, e.g. `SECRET_` |
| `include` | no | all | Comma-separated names or **regexes**; only matching secrets are exported. Names that do not exist are ignored |
| `exclude` | no | none | Comma-separated names or **regexes** to skip |
| `override` | no | `true` | If `false`, a variable that already exists in the environment is kept |
| `convert` | no | none | Rename the exported variables: `lower`, `upper`, `camel`, `constant`, `pascal`, `snake`. Any other value fails the step |
| `convert_prefix` | no | `true` | If `false`, the `prefix` is left out of the conversion |

> [!NOTE]
> [`action.yml`](action.yml) does not declare `override`, but [`src/main.ts`](src/main.ts) reads it, so it works. Case conversion uses [change-case](https://github.com/blakeembrey/change-case#core) packages for `camel`, `constant`, `pascal` and `snake`.

### Examples

**Basic**

```yaml
steps:
- uses: actions/checkout@v3
- uses: Glovo/secrets-to-env-action@v1
  with:
    secrets: ${{ toJSON(secrets) }}
- run: echo "Value of MY_SECRET: $MY_SECRET"
```

**Include or exclude** (comma separated, regex supported). To export secrets that start with a string, use `include: PREFIX_*`.

```yaml
- uses: Glovo/secrets-to-env-action@v1
  with:
    secrets: ${{ toJSON(secrets) }}
    exclude: MY_SECRET, MY_OTHER_SECRETS*   # MY_SECRET is not exported
```

```yaml
- uses: Glovo/secrets-to-env-action@v1
  with:
    secrets: ${{ toJSON(secrets) }}
    include: MY_SECRET, MY_OTHER_SECRETS*   # only these are exported
```

**Prefix**

```yaml
- uses: Glovo/secrets-to-env-action@v1
  with:
    secrets: ${{ toJSON(secrets) }}
    prefix: PREFIXED_
- run: echo "Value of PREFIXED_MY_SECRET: $PREFIXED_MY_SECRET"
```

**Do not override existing variables**

```yaml
env:
  MY_SECRET: KEEP_ME
steps:
- uses: Glovo/secrets-to-env-action@v1
  with:
    secrets: ${{ toJSON(secrets) }}
    override: false
- run: echo "Value of MY_SECRET: $MY_SECRET"   # prints KEEP_ME
```

**Convert names**

```yaml
- uses: Glovo/secrets-to-env-action@v1
  with:
    secrets: ${{ toJSON(secrets) }}
    convert: lower
- run: echo "Value of my_secret: $my_secret"
```

**Convert, but keep the prefix as written** (`convert_prefix: false`)

```yaml
- uses: Glovo/secrets-to-env-action@v1
  with:
    secrets: ${{ toJSON(secrets) }}
    prefix: PREFIX_
    convert: lower
    convert_prefix: false
# secret MY_SECRET becomes PREFIX_my_secret
```

---

## Build & local development

> [!NOTE]
> Prerequisites: Node.js 20 and Yarn (the build uses `yarn install --frozen-lockfile`). No credentials are needed.

```bash
yarn install --frozen-lockfile   # install dependencies
yarn run all                     # build, format, lint, package into dist/, test
```

| Script | What it does |
|---|---|
| `yarn build` | `tsc` |
| `yarn format` / `yarn format-check` | Prettier write / check on `**/*.ts` |
| `yarn lint` | ESLint on `src/**/*.ts` |
| `yarn package` | `ncc build` into `dist/` with source map and licenses |
| `yarn test` | Jest |

> [!IMPORTANT]
> [`dist/`](dist) is what GitHub runs ([`action.yml`](action.yml): `main: dist/index.js`) and it is committed. Run `yarn run package` and commit the result with every source change. The Build workflow fails if `dist/` differs from a fresh build.

---

## Testing

```bash
yarn test   # Jest, __tests__/main.test.ts
```

- 1 test file, [`__tests__/main.test.ts`](__tests__/main.test.ts), with 16 `test`/`it` calls. It mocks `@actions/core` and checks the exported variables.
- [`e2e.yml`](.github/workflows/e2e.yml) runs the action for real. See [CI/CD](#cicd).

---

## Code quality

| Tool | Role |
|---|---|
| [ESLint](.eslintrc.json) | Lint `src/**/*.ts` (`eslint-plugin-github`, `eslint-plugin-jest`, `eslint-plugin-prettier`) |
| [Prettier](.prettierrc.json) | Format TypeScript |
| [Dependabot](.github/dependabot.yml) | Daily updates for `github-actions` and `npm` |

---

## CI/CD

| Workflow | Trigger | Effect |
|---|---|---|
| [`build.yml`](.github/workflows/build.yml) | Push to `main`, pull requests (ignoring `**.md` changes), manual | Node 20.x, `yarn install --frozen-lockfile`, `yarn run all`, then fails if `dist/` has uncommitted changes and uploads the expected `dist/` as an artifact |
| [`e2e.yml`](.github/workflows/e2e.yml) | Push to `main` or `releases/*` | Runs the action from `./` in 6 jobs (exclude, include and prefix, override, convert, convert prefix, errors) and checks the exported variables |

> [!NOTE]
> The e2e checks expect repository secrets `SECRET_1`, `SECRET_2` and `SECRET_3` with values `VALUE_1`, `VALUE_2` and `VALUE_3`. The `test-errors` job runs two deliberately bad calls with `continue-on-error: true`, so it only shows the error output and cannot fail the workflow.

Nothing is published automatically. A new version is a Git tag on the commit (`v1` moves).

---

## Repository layout

```text
.
├── action.yml          # ⭐ action metadata: inputs, node20 runtime, entry point
├── src/main.ts         # ⭐ the whole action
├── __tests__/          # Jest tests (main.test.ts)
├── dist/               # bundled output run by GitHub (generated, committed)
├── .github/workflows/  # build.yml, e2e.yml; dependabot.yml
├── package.json        # scripts and dependencies
├── CODEOWNERS          # single rule: * @oNaiPs
└── LICENSE             # MIT
```

---

## Contributing & conventions

- Contributions are welcome. Past contributor named in the old README: Tamas Kadar (`@KTamas`).
- Change [`src/main.ts`](src/main.ts), run `yarn run all`, and commit the regenerated [`dist/`](dist).
- Keep `README.md` examples in sync with [`action.yml`](action.yml) and `src/main.ts` when you add an input. Declare it in `action.yml` too.
- Commits in this repo do not use ticket prefixes (0 of the last 30 have one).
- The scripts and documentation are released under the [MIT License](LICENSE).

**How it works.** `main.ts` parses the `secrets` JSON, drops `github_token`, applies `include` and `exclude` as regexes (`new RegExp(...)` on each name), adds `prefix`, applies `convert`, skips names already set when `override` is `false`, then exports each variable with `core.exportVariable`. Errors are reported with `core.setFailed`.

---

## Ownership & support

- **CODEOWNERS:** [`CODEOWNERS`](CODEOWNERS) has one rule, `* @oNaiPs`, the upstream author. It is not a Glovo team.
- **Upstream:** [oNaiPs/secrets-to-env-action](https://github.com/oNaiPs/secrets-to-env-action).
