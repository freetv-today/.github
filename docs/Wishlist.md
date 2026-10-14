# FreeTV Wishlist

This document records future improvements and ideas that are worth preserving but are not scheduled commitments.  Before implementation, each item must be reviewed against the current production architecture, reconciled with any related mini-spec, and divided into bounded development work.

The strategic sequence belongs in `Freetv-Roadmap.md`.  This document remains the detailed inventory.  Confirmed defects belong in `Bugs.md`.

## Status Summary

| Area | Roadmap group | Current disposition |
| --- | --- | --- |
| Automated test foundation | Developer foundation | Current |
| Archive-first show removal | Content lifecycle | Future design project |
| Incoming Hopper | Content lifecycle | Future design project |
| Browser-extension submission integration | Content lifecycle | Depends on the Hopper contract |
| Thumbnail Manager responsiveness and search | Existing Admin workflows | Future improvement |
| Containerized development and GHCR | Developer foundation | Future design project |
| Global Problems view | Existing Admin workflows | Future improvement |
| Audit/activity history | Existing Admin workflows | Requires design before implementation |
| Internet Archive health checking and quarantine | Content lifecycle | Design after safe removal foundation |
| Repeatable development database reset | Developer foundation | Future improvement |
| Playlist management, statistics, and default selection | Existing Admin workflows | Related project group |
| Playlist deletion/archive | Content lifecycle | Depends on playlist management and safe item removal |
| Cross-playlist awareness and identifier replacement | Existing Admin workflows | Related project group |

---

## Temp directory clean up in Production

**Status:** Current production-hardening priority.

Add a safe cleanup command/script for generated Admin Dashboard working artifacts under `temp/`, with configurable retention, and then run it periodically through production cron after validation.

The design must inventory every `temp/` consumer before defining retention.  It must distinguish disposable working artifacts from the current publication Undo and thumbnail Undo state.

The first implementation should provide:

* Dry-run reporting
* Explicit apply mode
* Constrained and validated cleanup paths
* Configurable retention by artifact category where necessary
* Protection for current rollback/Undo material
* Clear counts and failure reporting
* Tests against representative temporary-directory fixtures
* A documented production cron command and verification procedure

The cleanup command should be proven manually before scheduling it.

---

## Automated Test Foundation

Introduce automated testing incrementally after the initial post-v3 operational-hardening work.  The entire test foundation does not need to wait for every future playlist-management feature, but tests for those workflows should be added as their contracts stabilize. Different behaviors require different testing levels.

### PHP unit tests

Potential testable functions:

* Playlist filename validation
* Metadata normalization
* Changed-field detection
* Supported-field filtering
* Default-playlist selection rules

### PHP integration tests

Use a dedicated test database to verify:

* Invalid filename returns `400`
* Unknown playlist returns `404`
* Unauthorized requests return `401`
* Metadata updates modify only permitted columns
* Database errors return safe responses
* Changing the default preserves exactly one default
* JSON files are not modified by migrated endpoints

### Frontend tests

Add a JavaScript test runner such as Vitest for:

* Playlist signal state changes
* Proxy-response validation
* Metadata save failures
* Network failures
* Successful database save followed by refresh failure
* Stale playlist metadata prevention
* Default-playlist state refresh

### Browser tests

Consider a small number of end-to-end tests for critical workflows:

* Add playlist
* Add show
* Edit show
* Edit playlist
* Change default playlist
* Switch playlists

---

## Archive-First Show Removal

Replace permanent playlist-show deletion with a safe archive-first workflow before implementing the Hopper or broader quarantine automation.

The detailed proposal is recorded in:

* `FreeTV Content Lifecycle Mini Spec.md`

The initial implementation should focus on:

* A `deleted_playlist_shows` archive structure based on the final live schema
* Transactional archive-and-delete behavior
* Individual deletion only
* Useful snapshot metadata
* Deletion timestamp and reason
* Administrator attribution when the audit/identity contract supports it
* Preservation of the separation between Admin deletion and publication

The initial implementation should not include automatic thumbnail movement, bulk deletion, direct restoration, automatic deletion based on Internet Archive health, or Hopper implementation.

Terminology must be reconciled before schema work begins:

* **Disabled** — remains in live Admin data and may be repaired
* **Quarantined** — removed from normal operation for health/recovery review; exact storage model not yet decided
* **Archived/deleted** — deliberately removed from active playlist membership while retaining historical evidence
* **Retired** — possible terminal editorial state; not yet defined

These states should not be treated as interchangeable merely because they all prevent an item from appearing in published Viewer data.

I recently ran into an issue in production because I deleted some shows from the Admin Dashboard using the "Delete" button in the UI. This removed the records from the database. Here's what happened:

- Dataset validation failed during the SQL restore check because shows in freetv.json did not have contiguous sort_order values. The playlist contained 218 shows, but its order values ranged from 0 to 219, with 148 and 149 missing. Tooling requires show ordering to be contiguous and zero-based.
- The gaps were caused by deleting two show records directly through the Admin Dashboard. Those records were removed, but the remaining shows were not renumbered.
- I repaired the local database by renumbering shows in their existing sort_order, then id, sequence. This preserved their displayed order. Dataset validation then passed, data:publish updated the local freetv-data repository, and release:build successfully built and validated both release packages. You’ve since applied the same ordering repair to production.

Temporary workaround:

Until the refactor is implemented, set unwanted shows to Disabled instead of deleting them. Disabled shows remain in the database, preserving their order; the v4 Viewer now hides them.
The planned lifecycle design will replace hard deletion with Suspension: preserve the removed record’s metadata and original order in content stasis, remove it from Official content, and renumber the remaining shows as part of the same database transaction. Returning content will go through the Incoming Hopper for review rather than restoring an old database row directly.

---

## Create an Intake Hopper to Add New Titles to the DB

**Status:** Forward-looking design project.  Do not implement the submission endpoint until the lifecycle, Admin review workflow, validation rules, rejection/retry behavior, and promotion boundary have been specified.

See also:

* `FreeTV Content Lifecycle Mini Spec.md`

This would work in correlation with a Chrome browser extension that activates and functions on the Internet Archive and IMDB sites.

```
Archive.org
    ↓
"Add to FreeTV"
    ↓
incoming item

IMDb
    ↓
"Add to <archive identifier>"
    ↓
enrich same pending item
    ├── imdb
    ├── description
    ├── years
    └── thumbnail
```

Then:

```
Admin review
    ↓
approve
    ↓
playlist_shows
    ↓
next publication
    ↓
Viewer
```

Ingestion API can cheaply perform an immediate duplicate check against MariaDB. Cron can reconcile/clean up later, but there's no reason to knowingly accept obvious duplicates and wait for cron. The browser extension potentially eliminates the need for FreeTV to scrape IMDb server-side. The Admin's browser already has access to the loaded IMDb page and its image.

### Browser Extension Submission Queue

The extension should maintain a visible local queue so a contributor can understand the status of prepared and submitted items.

Possible client-facing states include:

* Draft
* Ready to submit
* Uploading
* Rejected by automated validation
* Ready to retry
* Accepted into the Hopper
* Awaiting Admin review

Server acceptance means that the payload was validated and stored.  It does not mean the item was editorially approved or published.

Rejected submissions should preserve the original queue item, provide a structured reason, identify the affected field where possible, and indicate whether correction/resubmission is allowed.

A successful private-beta submission should return an unguessable receipt or status credential that grants access only to the contributor-facing status of that submission.

This has been spec'ed out in `FreeTV Content Lifecycle Mini Spec.md`

### Thumbnail Ingestion and Normalization -- Part of Hopper Intake and Browser Extension

```
Thumbnail identity:
/thumbs/<imdb>.jpg

Storage:
filesystem, not MariaDB

Input:
JPEG

Normalization:
maximum width 1000px
preserve aspect ratio
never upscale

Sharing:
one physical thumbnail may serve multiple show rows/playlists

Manual workflow:
Add/Edit provides optional JPG upload

Future primary workflow:
Hopper/extension supplies thumbnail

Replacement:
warn about global impact before overwriting shared thumbnail
```
---

## Containerized Development Environment

Provide a reproducible development and testing environment that reduces host-machine drift and simplifies contributor setup.

The design phase should determine:

* Intended audiences and supported workflows
* Development versus production-oriented images
* Node, PHP, Composer, MariaDB, ImageMagick/Imagick, and web-server versions
* Source mounts, persistent volumes, and disposable state
* Development ports and environment defaults
* Sample, official, and fresh dataset choices
* Health checks and deterministic reset behavior
* Security boundaries and secret handling
* Versioning and GHCR publication strategy

The initial goal is a reliable developer environment, not an immediate collection of images for every possible user group.

Containerization should support the automated-test foundation by making disposable databases, clean installs, and Playwright environments reproducible locally and in CI.

---

## Responsive layout for the Thumbnail Manager

Most of the other areas of the Admin Dashboard were made responsive so they look good on large and small screens. The Thumbnail Manager was the most recent addition to the Dashboard and we haven't done anything regarding responsive layout. This page should look good on small screens too. We'll have to test first, make notes, and then come up with a plan to make this UI responsive. 

```
inspect current responsive behavior
→ document breakpoints/problems
→ design layout changes
→ implement
→ acceptance test
```
---

## Playlist Management Page

Create a dedicated playlist management page, tentatively:

- `EditPlaylist.jsx`
- Route: `/dashboard/playlists/<filename>`

The playlist name on the Admin Dashboard would link to this page.

The page could consolidate:

- Playlist metadata editing
- Playlist information
- Playlist statistics
- Default-playlist selection
- Last-updated information
- Future playlist-specific actions

Keep the existing Metadata and Information modals until the replacement page has been implemented and manually verified.

After verification:

- Remove `AdminPlaylistMetaModal`
- Remove `AdminInfoModal`
- Remove their state and handlers from `dashboard.jsx`
- Simplify the Current Playlist panel

### Compact Playlist Statistics

Display a small amount of useful playlist information directly in the Current Playlist panel.

Possible values:

- Total number of shows
- Active show count
- Disabled show count
- Last-updated date

Example:

```text
Default TV Shows · 412 records · 398 active
```
Keep this display compact. Detailed information should remain on the dedicated playlist management page.

### Playlist Deletion

Add a safe method for deleting playlists.

Current proposed rules:

* Playlist deletion belongs to Admin playlist management, not the publication system.
* A playlist may only be deleted when it contains zero active shows.
* The current default playlist may not be deleted. Another playlist must first be made the default.
* These rules must be enforced by the backend, not only by the UI.
* The delete action could live in the future Playlist Management page or, until that exists, the Playlist Metadata modal.
* Require explicit confirmation before deletion.

A non-empty playlist must first have its shows deliberately removed, archived, or reassigned through the appropriate Admin workflows. Playlist deletion must not implicitly cascade-delete shows or playlist memberships.

### Relationship to Publication

Deleting a playlist from MariaDB and removing a playlist from the published Viewer are separate operations.

Deleting an eligible empty playlist should:

1. Remove/archive the playlist through the Admin workflow.
2. Leave the currently published Viewer artifacts unchanged.
3. Cause the Publish page to detect that the previously published playlist is pending removal.
4. On the next playlist publication:
   * remove the obsolete playlist JSON artifact;
   * remove its entry from `index.json`;
   * preserve unrelated playlist artifacts and timestamps.
5. Publication Undo should restore the previous published playlist artifact and `index.json` entry without recreating the deleted MariaDB playlist.

This preserves the architectural rule that Publication Undo restores publication state only. It does not undo Admin CRUD operations.

### Future Archive Model For Playlists (similar to Show Archiving)

Consider a `deleted_playlists` archive table rather than permanently discarding deleted playlist metadata.

A restored archived playlist would not reclaim its previous live database ID. Restoration would re-enter the normal playlist creation workflow and receive a new live ID.

The exact archive schema, retention policy, and restore workflow remain future design work.

This has already been spec'ed out in `FreeTV Content Lifecycle Mini Spec.md`

---

## Repeatable Development Database Reset -- Not Required. Would be Nice to Have.

Create a documented or scripted workflow for rebuilding the development database from a known baseline.

The workflow could:

1. Drop the development database.
2. Recreate the database.
3. Import the checked or archived SQL dump.
4. Apply any later schema migrations.
5. Run basic verification queries.

The development database is disposable. A reliable reset process would make destructive testing safer and help expose code that depends on stale or accidental database state.

This should remain development-only and must not be usable against production without explicit safeguards.

---

## Activity and Audit History

A dashboard display of the last 10 actions may be useful, but it should be backed by a deliberate `activity_log` or `audit_log` architecture rather than transient UI state.

The system should eventually answer questions such as:

```
who deleted this show?
when?
from which playlist?
what changed?
```

Audit requirements should be designed alongside archive-first deletion, moderation, quarantine, recovery, and other workflows that require administrator attribution.

---

## Improve Problem Reporting / Issue Count Badge

Currently the system only shows problems or issues with the currently-selected playlist. It functions. But, this may allow bugs to go hidden unless the admin purposely switches playlists and checks all of the issues. Ideally, the /dashboard/problems page would show a total count of all problems from all playlists as one number (combined total). Then, on the page itself we can have the display broken down by specific playlist. No matter which playlist is selected, the Admin could view ALL problems from ALL playlists on one screen. Maybe to facilitate this, there is a "Switch Playlist" button next to the items which are not on the current playlist? The admin could click the button, switch to that playlist, and fix the problem(s).

---

### Contributor Authentication — Deferred Until Needed

The initial private prototype may use a narrowly scoped private credential.  A public contributor program may later justify a separate `accounts.freetv.today` application using established account and API-token components such as Laravel Fortify and Sanctum.Contributor accounts and token revocation should be introduced only when real contributors require them.  Viewer accounts and preference synchronization are separate future possibilities and are not requirements for the Hopper.


## Refine Thumbnail Manager Search --- Deferred (Part of a Larger Search Engine Improvement for Viewer and Dashboard)

Improve Thumbnail Manager search so Admins can perform more precise searches and better understand the results.

Ideas:

* Display the total number of matching results.
* Support whole-word matching in addition to the current substring matching.
* Allow more targeted searches by title and IMDb ID.
* Consider simple search operators or an advanced search mode if useful.
* Clearly distinguish between exact, whole-word, and partial matches.

Example:

A substring search for red currently matches:

* Alfred Hitchcock Presents
* Clifford the Big Red Dog
* Freddy's Nightmares
* The Incredible Hulk

A whole-word search for red should match only:

* Clifford the Big Red Dog

Could add some UI options like:

```
Search: [ red                 ]

Match:
(•) Contains
( ) Whole word
( ) Exact
```

This could also use the new search engine proposed for the front-end Viewer (Minisearch).

See the next section:

---

## Search Improvements for front-end Viewer

**Status:** Planned  
**Search engine:** MiniSearch (https://www.npmjs.com/package/minisearch)

### Goals
- Provide live, as-you-type search using the existing Search Results page.
- Improve relevance ranking and support useful full-text matching across show metadata.
- Keep the interface responsive while search indexing takes place.
- Preserve FreeTV's static JSON catalog architecture.

### Implementation approach
- Introduce a small internal search-engine API that isolates MiniSearch from the Preact components.
- Keep catalog loading independent of search-index construction.
- Preserve existing search validation and deliberately define matching behavior.
- Ensure playlist changes cannot produce stale search results.
- Keep search results compatible with the existing show display and playback workflow.

### Future optimization
If profiling with larger catalogs (approximately 5,000–10,000 titles) shows that indexing or searching affects responsiveness, move the search engine into a Web Worker behind the same API.

**Guiding principle:** Implement only what is needed now, while keeping the future worker migration straightforward.

---

## Internet Archive Health Checking and Cron Automation

**Status:** Future content-maintenance project.  Design after safe removal/archive semantics are established and before coupling health findings to a full quarantine workflow.

This section consolidates the earlier general idea of adding Internet Archive automation through cron.  See also `Incoming-Hopper-Quarantine-mini-spec.md`.

Add automated and manual health checking for Internet Archive items after the MariaDB refactor is complete.

The Admin Dashboard could provide a **Check for Dead Items** action for individual playlists. The check would query Internet Archive for each show's identifier and detect unavailable items, including records where `is_dark` is `true`.

Detected problems should **not automatically remove shows**. Instead, they should integrate with the existing Problems workflow so an administrator can review the affected show and decide whether to disable, replace, edit, or remove it.

Possible implementation:

- Add a manual **Check for Dead Items** action to the Admin Dashboard.
- Record when each playlist/show was last checked.
- Record the result of the most recent availability check.
- Create or reopen a problem report when an unavailable Internet Archive item is detected.
- Distinguish system-detected problems from user-submitted problem reports where useful.
- Surface detected problems through the existing Problems UI.
- Allow scheduled checks through cron.
- Prioritize items with the oldest or missing `last_checked` value so the entire library can be checked gradually.
- Limit the number/rate of Internet Archive requests per run to avoid unnecessary load.
- Do not automatically delete or disable shows based solely on an automated check.
- Use created_at for items that have never been checked and last_checked_at thereafter. Scheduled checks can select the oldest eligible records beyond a configurable age (for example, 120 days) and process only as many as the configured request budget allows.
- Treat Internet Archive request limits as configurable and verify current API/acceptable-use guidance before implementation. Respect rate-limit responses and back off or stop rather than risking an IP block.
- There should be one shared/global rate limiter for all Internet Archive health-check requests, regardless of whether they originate from cron, a manual Admin scan, retries, or multiple workers.
- Verify current rate-limit and abuse guidance specifically for Internet Archive metadata requests, including anonymous access to /metadata/{identifier}/is_dark, and design the checker around any documented headers or limits.

The Admin Dashboard could eventually have two complementary operational areas: Incoming Hopper for candidate content and Problems for existing content needing attention.

It also makes the lifecycle more complete than simply having a cron job that deletes dead records. The checker is really the maintenance half of a broader library curation system.

```
Incoming Hopper
    → "Should we add this?"

Automated Checker
    → "Should we keep this?"
```

Potential database changes could include health-check metadata on `playlist_shows`, such as `last_checked_at` and `last_check_status`. The exact schema should be designed when this feature is implemented rather than added during the current refactor.

This could eventually provide continuous library maintenance:

`scheduled/manual check → Internet Archive availability → update health status → flag problem → Admin review`

---

## Dependency/security maintenance

* Add a repeatable dependency-audit/update procedure for future releases.

---

## Wiki and User Manual

Create documentation for audiences and detail levels that do not belong in repository READMEs.

* Keep repository READMEs focused on developers, architecture, installation, and repository-specific workflows.
* Use a future wiki for deeper system administration, production operations, troubleshooting, and architectural concepts.
* Create a future user manual for everyday Admin Dashboard and Viewer tasks.
* Reuse the vocabulary and system boundaries established by the v3 developer documentation.

This is documentation expansion rather than a prerequisite for immediate production-hardening scripts.  New operational commands should still be documented when they are introduced.

---

## Review Support FAQs

Review Support FAQs on support.freetv.today. Make sure screenshots and instructions are current. 

Make sure there is no outdated information related to previous versions of Viewer (prior to version 4.x)



