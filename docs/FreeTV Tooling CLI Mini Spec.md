# FreeTV Tooling CLI Mini Spec

## Status

Future work.  The current npm commands remain the supported primitives.  A CLI wrapper would provide a guided human interface without replacing those commands.

## Purpose

FreeTV Tooling now coordinates Viewer builds, Admin builds, data publication, release packaging, production assembly, verification, and local development.  The individual commands work, but maintainers must remember which workflow to run, its prerequisites, and the manual steps surrounding it.

A future Node CLI can turn that operational knowledge into a guided workflow while retaining direct commands for experienced users, automation, and CI.

## Design Model

```text
Interactive CLI and machine-readable commands
                    ↓
Existing Tooling scripts and shared libraries
                    ↓
Viewer, Server, Data, assembly, and validation
```

The CLI should orchestrate existing capabilities rather than duplicate their implementation.

## Possible Entry Point

```bash
npm run freetv
```

or eventually:

```bash
freetv
```

The main menu could offer:

```text
FreeTV Tooling

Choose a workflow:
  Production build
  Viewer dev data publication
  Dataset publication
  First Run package build
  Development environment
  Diagnostics and repair
  Deployment preparation
```

## Guided Production Workflow

A production-build wizard could:

1. Check configured repository paths.
2. Confirm expected branches and report dirty working trees.
3. Verify required runtimes and dependencies.
4. Confirm local content has been reconciled with production.
5. Detect Viewer source changes that may require a service-worker cache update.
6. Run the complete `build:all` pipeline.
7. Preserve or compare the previous verified assembly.
8. Report added, changed, and removed public files.
9. Produce a concise deployment file list.
10. Stop before deployment unless the user explicitly configured and selected a deployment adapter.

Example output:

```text
FreeTV Production Assistant

✓ Repository configuration checked
✓ Source repositories clean
✓ Production content synchronized
✓ Viewer cache version checked
✓ Viewer built
✓ Admin built
✓ Data exported
✓ Production package assembled
✓ Production package verified

Next:
  Review the production output
  Deploy the verified public files
```

## Doctor Command

```bash
freetv doctor
```

The diagnostic command could inspect:

- Node.js and npm versions;
- PHP and Composer versions;
- required PHP extensions;
- MariaDB availability and database readiness;
- configured repository and output paths;
- expected branches and repository cleanliness;
- sibling-repository dependencies;
- `.env` availability without displaying secrets;
- development ports;
- staging and production-output permissions;
- Viewer development-data state;
- service-worker cache configuration;
- production assembly state; and
- recent build or validation failures.

Diagnostics should distinguish errors, warnings, and informational findings.  Safe repairs may be offered, but destructive or state-changing repairs require confirmation.

## Human and Agent Interfaces

Commands should support both readable terminal output and structured output:

```bash
freetv doctor --json
freetv build production --json
freetv deploy plan --json
```

Structured output should contain stable fields and meaningful exit codes so humans, CI, and future LLM agents can use the same operational contract.

A deployment plan might report:

```json
{
  "status": "ready",
  "verified": true,
  "environment": "production",
  "source_commits": {
    "viewer": "abc123",
    "server": "def456",
    "data": "789abc"
  },
  "changed_files": [
    "index.html",
    "service-worker.js",
    "assets/index-a1b2c3.js"
  ],
  "requires_confirmation": true
}
```

Secrets, passwords, tokens, private keys, and database credentials must never appear in terminal or JSON output.

## Optional Deployment Configuration

Deployment automation should be disabled by default.  An advanced user could deliberately create a local configuration file that identifies a deployment method and destination.

```yaml
deployment:
  enabled: true
  method: git

  git:
    repository_path: ../freetv-public-website
    branch: main
    auto_commit: false
    auto_push: false

  ssh:
    host: example.com
    user: deploy-user
    destination: /home/example/domains/example.com/public_html
    identity: ~/.ssh/example_deploy
```

The configuration should contain paths and connection names, not passwords, PATs, or private-key contents.  Authentication belongs to established systems such as SSH configuration, Git credential managers, `gh auth login`, or CI environment secrets.

Ordinary commits and pushes should use Git.  GitHub CLI may provide authentication, pull-request, or release support, but it does not replace the basic Git workflow.

## Capability Levels

Deployment support can be introduced gradually:

1. **Prepare only** — build, verify, compare, and report files.
2. **Synchronize** — copy verified public output into a local deployment-repository checkout.
3. **Commit** — optionally create a local deployment commit.
4. **Push** — optionally push and trigger hosting deployment.
5. **SSH deploy** — optionally transfer verified files to an explicitly configured destination.

Each level must be deliberately enabled.  Tooling must never silently fall back from one deployment method to another.

## Safety Requirements

- Every state-changing command supports `--dry-run` where meaningful.
- Production destinations are explicitly displayed before action.
- Destination paths are validated against broad or dangerous targets.
- An unverified assembly cannot be deployed.
- Dirty deployment checkouts block synchronization unless the user resolves them.
- Commit, push, SSH upload, migration, restart, rollback, and deletion require explicit authorization.
- Deployment actions produce a local audit summary.
- Tooling never exposes secrets in logs or diagnostic bundles.

## Agent-Assisted Operation

A future LLM agent could diagnose failures, interpret structured results, propose repairs, prepare builds, and validate outcomes through the CLI.  The agent should operate through the same stable commands available to humans.

The permission progression should be:

```text
Observe → Plan → Prepare → Explicit approval → Apply → Verify
```

The CLI, rather than unrestricted shell access, becomes the controlled operational interface.

## Initial Implementation Strategy

The first CLI version should be tested by the primary FreeTV maintainer and should focus on workflows already repeated manually.  Real deployment notes should determine which checks and prompts are valuable.  Visual polish is secondary to accurate state detection, clear explanations, and safe recovery instructions.

## Non-Goals for the First Version

- Replacing the underlying npm commands
- Automatically deploying by default
- Storing credentials
- Managing hosting accounts
- Performing unattended schema migrations
- Hiding important build or validation failures
- Attempting to automate every uncommon maintenance task

