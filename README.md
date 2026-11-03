# NexCall

Parent Git repository linking the NexCall web client, API and mobile client as Git submodules.

| Application | Location | Toolchain and instructions |
| --- | --- | --- |
| Web | [apps/web](apps/web) | Angular 15/npm; [build guide](apps/web/docs/build-and-run.md) |
| API | [apps/api](apps/api) | Spring Boot/Gradle, Java 25 toolchain; [setup](apps/api/README.md) |
| Mobile | [apps/mobile](apps/mobile) | Flutter, Dart `>=2.7.0 <3.0.0`; [manifest](apps/mobile/pubspec.yaml) |

Each app retains its own dependencies and build configuration. Run commands inside its directory, or use the root launcher:

```sh
./scripts/run web npm ci
./scripts/run web npm start
./scripts/run api ./gradlew test
./scripts/run mobile flutter pub get
./scripts/run mobile flutter run
```

Activate each app's supported runtime first. For web, use `nvm install && nvm use` inside `apps/web`. Mobile needs a compatible legacy Flutter/Dart SDK; the import does not upgrade it to Dart 3. API runtime configuration and external services are described in its README.

Each app is a Git submodule with its own history and origin remote. The parent records an exact commit for each app; `.gitmodules` declares the repositories and `master` update branches. Clone the complete workspace with:

```sh
git clone --recurse-submodules git@github.com:red-abacus/nexcall.git
```

For an existing checkout, run `git submodule update --init --recursive`. This requires SSH access to all three repositories. See [submodule operations](docs/monorepo.md) before updating or committing app changes.

See [monorepo operations](docs/monorepo.md) for builds, deployment workflows and submodule updates. Parent deployment workflows are manual; the individual app repositories retain their own deployment triggers.

## IDE run configurations

Open the monorepo root in IntelliJ IDEA or WebStorm. Shared configurations in `.run` target `apps/web/package.json`:

- **UI - Install dependencies**: runs `npm ci` using the web workspace's `.npmrc` and lockfile.
- **UI - Local**: runs `npm start` with the default local environment.
- **UI - Dev**, **UI - Staging**, **UI - Prod**: run the existing environment-specific serve scripts.

Set the IDE project Node.js interpreter to the version in `apps/web/.nvmrc` (currently **22.23.2**). Run **UI - Install dependencies** once, then **UI - Local** and open `http://localhost:4200/`. All serve configurations use port 4200, so run one at a time. Named environments retain their existing API endpoints and mock settings. JavaScript/Node.js support must be enabled in the IDE.
