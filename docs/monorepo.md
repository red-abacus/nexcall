# Parent repository and submodules

## Repository ownership

The parent repository links three independent Git repositories:

| Path | Origin | Update branch |
| --- | --- | --- |
| `apps/web` | `git@github.com:red-abacus/nexcall-web.git` | `master` |
| `apps/api` | `git@github.com:red-abacus/nexcall-api.git` | `master` |
| `apps/mobile` | `git@github.com:red-abacus/nexcall-mobile.git` | `master` |

The parent owns `.gitmodules`, run configurations, the launcher, these docs and optional root deployment workflows. It tracks each application as a gitlink pointing to an exact commit. Application source and history belong to the respective submodule. Initial commit IDs are recorded in [imports.json](imports.json); the parent Git index/commits are authoritative for subsequent pins.

The former snapshot setup was converted in place, retaining the same application paths and local dependencies. App documentation and workflows were restored to their original repository versions. No commits or pushes were performed by the conversion.

## Clone and initialize

```sh
git clone --recurse-submodules git@github.com:red-abacus/nexcall.git
```

For an existing clone, from the parent root:

```sh
git submodule update --init --recursive
git submodule status
```

SSH access to each private repository is required. Initialization checks out the commits recorded by the parent, usually with detached HEADs. A configured `branch = master` controls explicit remote updates; it does not automatically pull new commits.

## Update applications

First inspect `git status` inside affected apps and preserve any local work. To match the parent's recorded versions after pulling the parent:

```sh
git submodule update --init --recursive
```

To deliberately advance one application to its remote `master`:

```sh
git submodule update --remote apps/web
```

Review and test the updated application, then stage its new pointer with `git add apps/web` in the parent. Commit and push the parent only when requested. Use the analogous path for API or mobile. Do not use forced updates to discard local changes.

## Develop inside an application

Run Git commands in the application's directory. Before making development commits, switch to the intended existing branch or create a branch when requested. App changes must be committed and pushed to that app's repository first, then the new submodule pointer committed and pushed in the parent. Otherwise other users cannot fetch the recorded app commit.

`git add apps/web` in the parent records only the app's current commit; it does not commit uncommitted source changes inside the app. Inspect both parent and submodule statuses. Each app's existing deployment triggers still apply when pushing to its own repository.

## Local commands and IDE

The root `.idea/runConfigurations` configurations continue to target `apps/web/package.json`. Use Node 22.23.2, run **UI - Install dependencies** on a new checkout, then **UI - Local**. Existing installed dependencies were retained during conversion.

The launcher changes to the selected app directory and preserves arguments and exit status:

```sh
./scripts/run web npm start
./scripts/run api ./gradlew test bootJar
./scripts/run mobile flutter test
```

Activate each app's supported runtime first. Follow the [web build guide](../apps/web/docs/build-and-run.md), [API README](../apps/api/README.md), and [mobile manifest](../apps/mobile/pubspec.yaml). Mobile requires Dart below 3. The parent does not merge dependency managers or upgrade frameworks.

## Optional parent deployment workflows

The root [web](../.github/workflows/deploy-web-gke-prod.yml) and [API](../.github/workflows/deploy-api-gke-prod.yml) workflows remain manual wrappers targeting the existing services. They initialize submodules at the parent's pinned commits. Workflows inside submodules execute only in their own GitHub repositories, not as nested workflows in the parent.

For the parent workflows, configure `SUBMODULES_READ_TOKEN` with read access to the parent and all three application repositories. The parent's default GitHub token does not provide cross-repository access. Checkout handles the SSH submodule URLs using that token. Also configure Google Workload Identity trust for the parent repository, existing deployment secrets, and the `non-prod` GitHub environment before dispatching.

Parent workflows use separate `WEB_APP_BASE_URL` and `API_APP_BASE_URL` variables. App repository workflows retain their original variables and push triggers. Deployment documentation inside each app describes that app's standalone repository; root workflow paths and variables are described here. No workflow was dispatched during conversion.

Docker build contexts remain app-local:

```sh
docker build -t nexcall-web:local apps/web
./scripts/run api ./gradlew bootJar
docker build -t nexcall-api:local apps/api
```
