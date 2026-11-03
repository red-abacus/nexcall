# NexCall parent repository

This repository links three independently built applications as Git submodules:

- `apps/web`: Angular web client. Read its `AGENTS.md`, architecture and build guides before edits. Its Angular workspace and reusable packages remain inside `apps/web`.
- `apps/api`: Java/Spring Boot API. Read its `README.md`, `docs/agent-workflow.md` and relevant sections of `docs/java-spring-guidelines.md` before edits.
- `apps/mobile`: Flutter mobile client. Read its `README.md`, `pubspec.yaml` and platform build configuration before edits. It currently requires Dart below 3; upgrades are separate work.

Run application commands from that application's directory, or use `scripts/run <app> <command> [args...]`. Preserve each application's lockfiles, wrapper, toolchain and configuration. Do not introduce cross-application source imports or unify their dependency managers as an incidental change.

Each app has its own Git history and remote; the parent tracks gitlinks, not app source files. Preserve submodule boundaries. Inspect status inside each affected app as well as the parent. Submodule updates normally check out detached commits; select an appropriate existing branch before authorized development commits. Application-local guidance applies within that app. Root deployment workflows live in `.github/workflows`; see `docs/monorepo.md` for paths and migration details. Preserve calling, authentication, API contracts and application boundaries across changes. Verify each affected app independently.

Do not copy credentials, generated output or local dependency caches into source control. Do not create commits, branches, pushes, releases or deployments unless requested. Preserve unrelated work.
