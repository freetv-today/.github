# FreeTV Roadmap

This roadmap describes the planned development direction for FreeTV.  It identifies broad priorities and sequencing rather than promising release dates.

Detailed feature ideas remain in `Wishlist.md`.  Project-specific behavior may be explored in separate mini-specs.  Before implementation, every roadmap area must be reviewed against the current production architecture and divided into bounded development work.

---

## Planning Principles

Future work should follow these principles:

1. Complete the operational lifecycle of existing features before introducing systems that create substantial new state or maintenance responsibilities.
2. Analyze architecture, dependencies, migration needs, testing, and deployment before implementation begins.
3. Keep MariaDB authoritative for content management while preserving JSON as the portable Viewer distribution format.
4. Keep Admin CRUD, publication, deployment, and publication Undo as distinct responsibilities.
5. Prefer mature existing solutions for commodity infrastructure rather than creating custom replacements.
6. Introduce automation to assist editorial decisions, not silently replace them.
7. Preserve the ability to develop and use the FreeTV repositories independently where appropriate.

---

## Active Priority: Production Hardening and Operational Cleanup

Complete the remaining operational responsibilities created or exposed by the v3 refactor before beginning major forward-looking features.

### Documentation Reconciliation

- Update roadmap and WISHLIST assumptions to match the live v3 architecture.
- Mark completed or superseded work clearly.
- Keep confirmed bugs separate from feature requests.
- Preserve project mini-specs as design inputs rather than scheduled commitments.
- Document recurring production maintenance procedures.

### Runtime and Temporary-File Maintenance

- Add a safe cleanup command for generated Admin working files under `temp/`.
- Use configurable retention periods where appropriate.
- Protect current publication and thumbnail Undo state from premature removal.
- Run the cleanup process periodically through production cron after validation.
- Document dry-run, apply, verification, and recovery behavior.

### Dependency and Security Maintenance

- Review development and build dependencies reported by package audits.
- Apply safe updates as a dedicated maintenance slice.
- Run complete build and regression checks after dependency changes.
- Establish a repeatable dependency-audit and update procedure.

### Production Baseline and Identification

The initial baseline and application-version visibility work is complete enough for current operational needs.  Viewer and Admin display application versions sourced from their respective `package.json` files, answering the immediate question of which app version is running.

- Maintain the established production baseline and recovery materials.
- Revisit build-specific identifiers (for example, publishing-log build identities independent of app version bumps) only when a concrete operational need arises.
- Continue to distinguish application versions, build identities, and dataset snapshot identities when introducing future diagnostics.

---

## Developer Foundation

Strengthen the development and testing environment incrementally, alongside bounded operational improvements and before undertaking large new subsystems.

### Guided FreeTV Tooling CLI

- Implement a guided Node CLI as an interface to existing `freetv-tooling` commands and shared libraries; do not duplicate the underlying workflows.
- Start with a small, verifiable diagnostics or other read-only slice, then add guided build, publication, and preparation workflows as contracts stabilize.
- Support accurate preflight checks, readable status/errors, structured JSON output, stable exit codes, secret redaction, and explicit approval for mutations.
- Keep deployment disabled by default.  The first version should prepare, verify, compare, and report release artifacts rather than deploying them automatically.
- Preserve the direct npm commands for development, automation, and CI.  See `FreeTV Tooling CLI Mini Spec.md` for the design input.

### Repeatable Development State

- Provide a guarded, development-only database reset workflow.
- Recreate the database from known schema and dataset inputs.
- Add basic post-reset verification.
- Ensure production databases cannot be targeted accidentally.

### Containerized Development

- Define supported development and testing use cases before creating images.
- Pin Node, PHP, Composer, MariaDB, ImageMagick/Imagick, and other runtime dependencies.
- Preserve source-code editing on the host while running services in containers.
- Use persistent volumes only where development state should survive container replacement.
- Consider separate contributor, testing, demo, and self-hosting profiles where justified.
- Publish GHCR images only after the container contracts are stable and documented.

### Automated Testing

- Add focused PHP unit tests for isolated rules and validation.
- Add integration tests using a disposable test database.
- Add frontend tests for important Admin state and failure behavior.
- Add a small Playwright suite for critical Admin, Viewer, publishing, and First Run journeys.
- Add regression checks ensuring migrated Admin behavior does not depend on legacy playlist JSON.
- Run deterministic tests in CI after the local test environment is reliable.

---

## Existing Admin and Content Workflows

Improve established workflows before expanding the system's responsibilities.

### Playlist Management

- Create a dedicated playlist-management page.
- Consolidate metadata, statistics, default-playlist selection, and appropriate playlist actions.
- Preserve the invariant that exactly one playlist is the default.
- Add safe playlist deletion only after its database and publication semantics are fully specified.

### Cross-Playlist Awareness

- Show where identifiers, shows, and thumbnails are shared.
- Provide explicit current-playlist versus all-playlists choices where appropriate.
- Add transactional replacement-identifier workflows with conflict previews.

### Problems and Operational Visibility

- Show a global problem count rather than hiding problems behind the currently selected playlist.
- Allow administrators to review problems from every playlist in one place.
- Preserve playlist context and provide a clear path to the affected record.
- Distinguish user-submitted reports from future system-detected health findings where useful.

### Thumbnail Manager

- Inspect and improve responsive behavior on small screens.
- Refine search with clear contains, whole-word, and exact matching modes.
- Preserve shared-thumbnail warnings and active-reference awareness.

### Audit and Activity History

- Design an audit/activity model before adding a superficial recent-actions widget.
- Record who performed meaningful actions, when they occurred, which records were affected, and what changed.
- Use this foundation where future deletion, moderation, and recovery workflows require attribution.

---

## Content Lifecycle

Develop content removal, health, recovery, and intake as related but deliberately separated systems.

### Phase 1: Content Stasis and Safe Removal

- Resolve the current destructive-delete behavior by designing a suspension/content-stasis workflow that preserves removed playlist-show metadata and original ordering for historical reference.
- Remove the active playlist membership and renumber the remaining shows to contiguous, zero-based `sort_order` values in the same database transaction, preserving their relative display order.  Test rollback if any part fails.
- Record an explicit reason, timestamps, and administrator attribution where the audit/identity contract supports it.
- Distinguish Disabled, Quarantined, Suspended/Archived, and Retired in the domain model before finalizing storage or UI semantics.
- Returning removed content should enter the Incoming Hopper for review rather than restoring an old live row directly; detailed lifecycle contracts remain to be finalized.
- Do not automatically move or delete shared thumbnails.  Keep Admin state changes separate from publication and publication Undo.
- See `FreeTV Content Lifecycle Mini Spec.md` and the production ordering-integrity incident documented in `Wishlist.md`.

### Phase 2: Content Health and Quarantine

- Add rate-limited Internet Archive availability checking for existing content.
- Support manual and scheduled checks through one shared request budget.
- Flag potential problems for human review rather than automatically deleting or disabling records.
- Preserve recoverable metadata and source history.
- Define quarantine, recovery, and retirement states before choosing schemas or UI.

### Phase 3: Incoming Hopper

- Treat the Hopper as the controlled entrance for new and replacement content.
- Keep candidates separate from live content until explicit approval.
- Design identity, duplicate rules, validation, rejection reasons, retry policies, promotion transactions, and Admin review screens before creating the public endpoint.
- Allow quarantined or archived content to create recovery candidates without bypassing normal review.

### Phase 4: Browser Extension and Contributor Access

- Continue the browser extension as a capture and submission-queue prototype.
- Define the versioned submission contract only after the Hopper workflow is stable.
- Begin private beta access with narrowly scoped credentials.
- Add a separate account/token service only when additional contributors justify it.
- Prefer established authentication and token components such as Laravel Fortify and Sanctum if that phase is reached.

---

## Viewer and Playback

Viewer v4 introduced a custom FreeTV player, so its original integration is no longer a forward-looking roadmap milestone.  Verify the current released behavior and retain only concrete follow-up improvements.

- Improve playback-error detection and reporting.
- Refine direct-file playback and remaining iframe fallbacks where actual unsupported cases justify changes.
- Audit Viewer utilities inherited from retired systems and remove only code proven unused.
- Explore offline downloads, native clients, and removable-storage libraries as separate future projects.

Core catalog browsing and playback should remain independent of optional dynamic services.

---

## Dynamic Viewer Services

Possible future API-driven enhancements include:

- Anonymous content-centric viewing metrics
- Popular and trending content
- Playback-health metrics
- Other optional Viewer services

These services must remain optional.  Viewer access to the published catalog should not require accounts or continuous availability of dynamic services.

---

## Publishing Platform and Portability

Maintain JSON as the portable published content format.

Potential future publishing capabilities include:

- A deliberate Viewer distribution/configuration contract
- Publisher-specific branding and endpoints
- Custom PWA manifests and icons
- White-label Viewer deployments
- A "Powered by FreeTV" publisher model

Potential consumers include:

- FreeTV Web/PWA Viewer
- CDN-hosted deployments
- Desktop applications
- Native Android applications
- Offline content bundles
- Removable-storage libraries
- Third-party FreeTV-powered publishers

MariaDB remains authoritative for content management.  Published JSON remains the portable distribution contract, allowing consumers to remain independent of the Admin implementation and database technology.

---

## Documentation

- Keep repository READMEs focused on developer setup, architecture, and repository-specific workflows.
- Create a future wiki for deeper system-administration, operational, and troubleshooting material.
- Create a future user manual for everyday Viewer and Admin tasks.
- Keep operational cheat sheets and personal testing notes separate from the strategic roadmap.

---

## Sequencing Summary

The current intended order is:

1. Remaining production hardening and operational cleanup
2. Guided FreeTV Tooling CLI, with incremental automated testing and reproducible development-state work alongside it
3. Improvements to existing Admin/content workflows
4. Transactional content stasis and safe removal, including playlist-order integrity
5. Content health checking, human-reviewed quarantine, and recovery design
6. Incoming Hopper and editorial review system
7. Browser-extension submission integration
8. Remaining Viewer, playback, publishing-platform, and distribution projects as separately planned milestones

This ordering is directional rather than a release commitment.  Smaller independent improvements may be completed between larger milestones when they do not destabilize the current foundation or create unplanned operational responsibilities.


