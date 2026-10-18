---
name: nix-development
description: Set up, modify, or troubleshoot reproducible development environments built with Nix flakes. Use this skill whenever a user mentions flake.nix, devShells, nix develop, shell.nix migration, nixpkgs unstable, devenv, direnv, reproducible project tooling, or wants to add development dependencies with Nix. It supports language-agnostic projects and requires explicit approval before edits and again before validation commands are run.
---

# Nix Development Environments

Create and maintain project development environments using Nix flakes, `nixpkgs` unstable, `devenv`, and `direnv`. Keep the flake's public interface small and place shell-specific details in separate Nix files.

## Interaction contract

Treat Nix configuration as executable project policy. Do not modify it or activate it implicitly.

1. Inspect the repository and explain the proposed changes first.
2. Ask for one explicit approval before creating, editing, renaming, or deleting files. Group related edits into one clear request.
3. After the edits are approved and complete, propose validation commands and ask for a separate approval before running any of them.
4. Do not run `direnv allow` without calling it out by name and receiving approval. It authorizes automatic evaluation of project configuration.

If the user declines either approval, provide the plan or commands only and stop at that boundary.

## Defaults and scope

- Require a flake for new work. Use `nixpkgs/nixos-unstable` unless the project already pins a different revision and the user asks to change it.
- Support all languages. Add language packages, formatters, linters, services, and environment variables only when the project needs them.
- Prefer `nix develop` and `devShells.<system>.default` as the portable interface.
- Use `devenv` for its declarative modules, services, scripts, tests, and git hooks when those features are needed. Do not create duplicate definitions in both a raw `mkShell` and devenv without explaining which one is authoritative.
- Integrate `direnv` through a checked-in `.envrc` only when requested or when the project already uses it. Preserve a user's local-only environment settings.
- Never replace or delete an existing `shell.nix`, `flake.nix`, `.envrc`, `devenv.nix`, lockfile, or CI setup merely because a newer layout is preferred.

## Inspect before proposing changes

Look for, at minimum:

- `flake.nix`, `flake.lock`, `shell.nix`, `default.nix`, and the `nix/` directory
- `devenv.nix`, `devenv.yaml`, and `.envrc`
- language manifests and lockfiles, formatter/linter configuration, task runners, and CI workflows
- project documentation mentioning setup commands or supported platforms

Read the existing Nix expressions before proposing a layout. Determine whether the project already has a flake, which systems it supports, how dependencies are pinned, whether `devenv` is installed or configured, and whether `direnv` is intentionally excluded.

When a command is useful for discovery, prefer read-only commands such as `nix flake metadata` or `nix flake show`. Ask for permission before executing them if command execution has not already been approved by the user.

## File layout

For a new or lightly configured repository, propose this layout unless the repository's established convention is better:

```text
.
├── flake.nix
├── flake.lock
├── devenv.nix              # devenv-specific modules and services, when used
├── .envrc                  # direnv entry point, when used
└── nix/
    └── devshell.nix        # packages, variables, and shell hooks
```

`flake.nix` should contain inputs, supported-system plumbing, and exported outputs. Put `mkShell` arguments, package lists, environment variables, and shell hooks in `nix/devshell.nix`. Use further focused files under `nix/` when the shell grows substantially, for example `nix/packages.nix` or `nix/hooks.nix`.

Do not use `shell.nix` for new setups. If one already exists, retain it unless the user explicitly approves a migration plan. During a migration, explain compatibility implications for users who still run `nix-shell`.

## Design workflow

### 1. State the current situation

Summarize relevant existing files, detected project tooling, and any uncertainties. If requirements are incomplete, ask targeted questions such as:

- Which platforms must be supported (for example, Linux only or Linux and macOS)?
- Which project commands should work inside the shell?
- Are database or other local services required?
- Should `devenv` own the environment, or should it add features around a `devShell`?

### 2. Propose an implementation

Describe exactly which files will be added or changed and why. Include whether `flake.lock` will change, whether `.envrc` will be created or edited, and which source owns each dependency.

For a new general-purpose flake, use an explicit supported-systems strategy and expose `devShells.<system>.default`. Import the separate shell expression rather than placing the whole `mkShell` body in `flake.nix`. Keep the initial package set minimal: do not add arbitrary language toolchains just to make the template appear complete.

If the user requests `devenv`, select an integration that is compatible with the installed/version-pinned devenv setup. Keep devenv declarations in `devenv.nix` (or focused files it imports). Make clear whether users enter with `nix develop`, `devenv shell`, or both, and ensure documented entry points do not silently diverge.

For `direnv`, use the project's existing convention. A flake-based `.envrc` generally relies on nix-direnv support; a devenv-based `.envrc` relies on the appropriate devenv integration. Do not write an entry point whose required direnv extension has not been identified.

### 3. Obtain edit approval

Use a concise confirmation request similar to:

> I will add `nix/devshell.nix`, update `flake.nix` to export the default dev shell, add/update `devenv.nix` and `.envrc`, and generate/update `flake.lock` if dependency resolution requires it. May I make these edits?

Wait for an affirmative answer. If the proposed work differs from the approved plan, ask again before making the additional edits.

### 4. Edit carefully

Preserve formatting and local naming conventions. Keep comments focused on non-obvious constraints rather than restating Nix syntax. Pin flake inputs through `flake.lock`; do not substitute floating URLs for a lockfile.

When adding a shell package, prefer the package supplied by the project's pinned `pkgs`. Avoid unfree packages, insecure-package allowances, binary caches, or substituter configuration unless the user explicitly requests and approves them.

### 5. Propose validation and obtain separate approval

After edits, list only relevant commands, explain their purpose, and request approval. Select from:

```sh
nix flake check
nix flake show
nix develop --command <project-check-or-tool-version-command>
devenv test
devenv shell -- <project-check-or-tool-version-command>
direnv allow
```

Use a harmless command for shell-entry verification when possible, such as a required tool's `--version` or a project formatter's check mode. Explain expected network access, builds, downloads, or lockfile updates before commands that may cause them. Never run `nix flake update`, broad garbage collection, store deletion, or profile-changing commands unless the user explicitly asks.

### 6. Report the outcome

State the files changed, the entry command(s), validation results, and any remaining manual step. For example, identify whether the user should run `nix develop`, `devenv shell`, and/or `direnv allow` after reviewing `.envrc`.

## Troubleshooting principles

- Reproduce and inspect the actual error before changing pins or adding overrides.
- Distinguish evaluation failures, missing attributes, platform support issues, build failures, and runtime dependency problems.
- Check the pinned input and package attribute before assuming an unstable package exists or has the expected name.
- Prefer a small, documented correction over global Nix configuration changes.
- When a change may affect reproducibility or all contributors, present alternatives and request approval before editing.

## Completion criteria

Consider the task complete only when the configuration is modular, the declared entry point is documented in the final response, requested checks have either passed or their result is clearly reported, and every edit and validation command was preceded by the required approval.
