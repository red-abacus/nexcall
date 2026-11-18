# NexCall parent repository

This repository links three independently built applications as Git submodules:

- `apps/web`: Angular web client. Read its `AGENTS.md`, architecture and build guides before edits. Its Angular workspace and reusable packages remain inside `apps/web`.
- `apps/api`: Java/Spring Boot API. Read its `README.md`, `docs/agent-workflow.md` and relevant sections of `docs/java-spring-guidelines.md` before edits.
- `apps/mobile`: Flutter mobile client. Read its `README.md`, `pubspec.yaml` and platform build configuration before edits. It currently requires Dart below 3; upgrades are separate work.

Run application commands from that application's directory, or use `scripts/run <app> <command> [args...]`. Preserve each application's lockfiles, wrapper, toolchain and configuration. Do not introduce cross-application source imports or unify their dependency managers as an incidental change.

Each app has its own Git history and remote; the parent tracks gitlinks, not app source files. Preserve submodule boundaries. Inspect status inside each affected app as well as the parent. Submodule updates normally check out detached commits; select an appropriate existing branch before authorized development commits. Application-local guidance applies within that app. Root deployment workflows live in `.github/workflows`; see `docs/monorepo.md` for paths and migration details. Preserve calling, authentication, API contracts and application boundaries across changes. Verify each affected app independently.

Do not copy credentials, generated output or local dependency caches into source control. Do not create commits, branches, pushes, releases or deployments unless requested. Preserve unrelated work.

## Decision log

The parent repository's `docs/decisions/README.md` indexes the canonical project decision log, including decisions that affect individual submodules. Read the relevant records before changing established behavior. Keep implementation plans and API handoffs in their application repositories and link them from the decision records.

As part of the same task, record new consequential product, architecture, API-contract and operational decisions under the parent's `docs/decisions/`, and update affected existing records when a decision is implemented, deferred or replaced. Include a stable ID, the decision/source date when known, context and consequences, source references, status, and whether the origin is an explicit user decision, an agreed requirement or a technical choice. Do not infer user approval from code or a proposal. Distinguish implementation in the checkout from verification and deployment; retain unresolved questions explicitly.

Preserve superseded decisions and link them to their replacements. Update the index and any source plan whose decision status would otherwise contradict the log. Do not leave decisions only in chat, plans or handoffs. This documentation requirement does not authorize commits or changes to unrelated application code.
