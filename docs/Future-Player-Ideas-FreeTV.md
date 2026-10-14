# FreeTV — Future Player Ideas & Development Notes

**Status:** Working notes
**Context:** Future work following the v3 server/data refactor
**Related development:** `freetv-player-dev` / future `freetv-viewer` refactor

---

## 1. Where the Player Fits

The current FreeTV v3 Viewer was intentionally kept relatively stable while the server architecture was being refactored.

The major v3 work was about:

* moving administrative data into MariaDB;
* making MariaDB authoritative for Admin workflows;
* publishing static JSON artifacts for the Viewer;
* removing the Viewer's dependence on the old server-side JSON/config architecture;
* getting the server, data, and tooling architecture into a clean production state.

The Player work is therefore **future work**, rather than something that needs to be folded into the v3 server refactor.

The eventual direction is a more capable Viewer/Player built around the prototype work in the development Player.

The current plan is roughly:

```text
                 ┌──────────────────┐
                 │    Admin GUI     │
                 │     MariaDB      │
                 └────────┬─────────┘
                          │
                    Publish static
                       artifacts
                          │
                          ▼
                 ┌──────────────────┐
                 │  FreeTV Viewer   │
                 │                  │
                 │ Catalog / UI     │
                 │ Future Player    │
                 └────────┬─────────┘
                          │
                          ▼
                 Internet Archive
```

The Player should not become another database-driven application.

The published FreeTV data remains the authoritative catalog consumed by the Viewer, while the Player can obtain additional technical/file information from Internet Archive when it needs it.

---

# 2. The Development Player

`freetv-player-dev` was created as a place to experiment with the future Player without destabilizing the production Viewer.

The prototype is useful because it lets us experiment with Player behavior independently of the current production application.

The eventual goal is not necessarily to ship the development repository itself as a separate public product.

Instead, its useful ideas and implementation can be folded into a future `freetv-viewer` refactor.

The development Player should therefore be thought of as:

> a laboratory for the next generation of the FreeTV Viewer.

---

# 3. Direct Internet Archive Playback

One of the clearest future changes is moving away from the current iframe-based playback approach.

The future Player should load Internet Archive media **directly**.

Instead of:

```text
FreeTV
   ↓
iframe
   ↓
Internet Archive player
   ↓
video
```

the desired architecture is closer to:

```text
FreeTV Viewer
      ↓
   FreeTV Player
      ↓
Internet Archive media/file
      ↓
browser video element
```

This gives FreeTV much more control over the playback experience.

The Player can know:

* when playback starts;
* when playback pauses;
* when playback ends;
* how far someone watched;
* what file is being played;
* whether playback failed;
* what episode is currently playing;
* what controls are presented;
* how errors are displayed;
* how the Viewer responds to playback events.

This is one of the main reasons the future Player is worth building rather than simply embedding the IA player indefinitely.

---

# 4. Player Responsibilities vs. Catalog Responsibilities

A useful architectural distinction emerged from these discussions.

The FreeTV catalog should remain relatively small and curated.

It should answer questions such as:

* What shows does FreeTV offer?
* What playlist is this show in?
* What is the show's title?
* What is its IMDb identifier?
* What episodes/items belong to the show?
* What should the Viewer display?

It should **not** attempt to duplicate every piece of Internet Archive metadata.

Internet Archive remains the source for detailed technical/file information.

Therefore:

```text
FreeTV data
    =
curated catalog information

Internet Archive
    =
source media + detailed file information

Player
    =
connects the two at playback time
```

This keeps the FreeTV data portable and avoids turning the catalog into a giant technical metadata database.

---

# 5. Episode and File Name Normalization

One of the more interesting future ideas was dealing with imperfect Internet Archive metadata.

Internet Archive collections are not always named or organized in a way that produces a good FreeTV presentation.

A show may be perfectly usable, but have:

* awkward episode names;
* inconsistent filenames;
* inconsistent numbering;
* metadata that does not match the way FreeTV wants to present the show;
* filenames that are technically correct but ugly for viewers.

We discussed the possibility of having FreeTV normalize this information at the Player/client layer.

For example, the underlying IA file might have an awkward name, while FreeTV could present:

```text
Season 1
  Episode 1 — Pilot
  Episode 2 — The Second Episode
```

without requiring the original IA metadata to be changed.

### Important principle

We do not need a general-purpose metadata-cleaning system for every IA item.

A targeted workaround may be appropriate when:

> the show is unique/useful/popular and the only significant problem is naming.

In that case, FreeTV could maintain a known normalization rule rather than rejecting otherwise usable content.

The Player is a particularly logical place for some of this logic because it already has to translate FreeTV's episode information into an actual IA file to play.

---

# 6. The Player Could Resolve Files Dynamically

This leads to another useful architectural possibility.

FreeTV's catalog could identify an episode without necessarily hard-coding every final media URL.

The Player could:

1. receive the FreeTV episode;
2. identify the corresponding IA item;
3. inspect the available files;
4. select the appropriate playable file;
5. normalize the presentation name;
6. begin playback.

This gives us more flexibility when IA file layouts differ between collections.

It also means that changes to IA's file organization do not necessarily require rebuilding the entire FreeTV catalog.

The exact amount of dynamic resolution still needs to be determined.

---

# 7. Playback Events / Data Collection

Another major discussion concerned collecting anonymous playback information.

The old system had concepts around logging/beacons, but the future approach should be much more deliberate.

The Player is in a unique position because it knows what actually happened during playback.

Potential events include:

```text
play started
    ↓
25% watched
    ↓
50% watched
    ↓
75% watched
    ↓
completed
```

Other potentially useful events:

* playback started;
* playback paused;
* playback resumed;
* playback ended;
* playback failed;
* file could not be loaded.

The exact event set should remain small initially.

The purpose is not to collect everything simply because the Player can.

---

# 8. Popular Content

The first obvious use for playback statistics was **Popular Content**.

Rather than guessing what people like based on catalog metadata, FreeTV could eventually know which shows/episodes are actually being watched.

Conceptually:

```text
Player
   ↓
anonymous playback events
   ↓
aggregate statistics
   ↓
Popular Content
```

This could eventually support things such as:

* most-watched shows;
* most-watched episodes;
* recently popular content;
* viewing trends over time.

This was considered substantially more concrete than immediately trying to build a recommendation engine.

---

# 9. Recommendations Are a Later Possibility

Recommendations were discussed as a possible future use of the collected data.

However, the idea was deliberately not to jump immediately from:

> "We have viewing data"

to:

> "We need an algorithmic recommendation system."

The sensible first step is simply collecting useful aggregate information.

Once there is enough data, FreeTV could investigate whether it supports useful features such as:

* related shows;
* frequently watched together;
* popular within a category;
* recommendations based on viewing history.

But this is **future exploration**, not part of the initial Player implementation.

---

# 10. Privacy / Data Collection Principle

The analytics concept should remain intentionally limited.

The Player does not need to become an invasive tracking system.

The useful information is primarily about **content usage**, rather than building detailed profiles of individual people.

The initial mental model was therefore closer to:

```text
"What content is being watched?"
```

rather than:

```text
"What does this particular person do?"
```

Any eventual implementation would need its own privacy/security design before deployment.

---

# 11. Resume / Playback State

A more capable Player also creates opportunities for better playback state.

Potential future functionality includes remembering:

* what show was being watched;
* what episode;
* approximately where playback stopped.

This could eventually produce a:

> Continue Watching

experience.

The important point is that this does not necessarily require server-side accounts.

Some playback state could potentially remain local to the Viewer.

Exactly which state belongs locally versus remotely is a future architecture decision.

---

# 12. Episode Navigation

A custom Player can also understand the relationship between episodes.

That opens the possibility of controls such as:

* Previous Episode
* Next Episode
* episode list
* automatic next episode
* return to show

The Player therefore becomes more than a generic `<video>` wrapper.

It can understand:

```text
Playlist
  ↓
Show
  ↓
Season / Episode
  ↓
IA Item
  ↓
IA File
  ↓
Playback
```

That relationship is valuable for creating a television-like viewing experience.

---

# 13. Playback Controls

A custom Player gives us control over the interface rather than inheriting whatever controls the embedded IA player happens to provide.

Potential controls/features discussed include:

* play/pause;
* seek;
* volume;
* fullscreen;
* playback speed;
* progress;
* subtitles/captions;
* episode navigation;
* playback state;
* error messages;
* loading state.

Not all of these need to exist in the first version.

The important architectural decision is that the Player should make them possible.

---

# 14. Playback Errors

Direct playback also gives FreeTV an opportunity to handle failures intelligently.

Instead of simply displaying a generic embedded-player failure, the FreeTV Player can distinguish between situations such as:

```text
Unable to locate IA item
        ↓
Unable to find playable file
        ↓
File exists but cannot be loaded
        ↓
Playback interrupted
```

That could eventually produce useful human-readable messages.

It also provides useful information for the future playback metrics system.

---

# 15. Download / Offline Viewing

Offline playback came up as a longer-term possibility.

A future Viewer could potentially allow someone to:

```text
Find show
   ↓
Download episode
   ↓
Store locally
   ↓
Play without network access
```

This becomes particularly interesting on mobile/native platforms.

However, this is **not a near-term Player requirement**.

There are significant questions around:

* storage;
* file size;
* browser storage limits;
* managing downloaded files;
* deleting downloads;
* IA availability;
* device storage;
* permissions;
* licensing/usage considerations.

Therefore the idea belongs on the future roadmap rather than in the initial Player refactor.

---

# 16. Android / Native Possibilities

We also discussed how the future Viewer might eventually expand beyond a normal desktop web application.

One possible path is:

```text
Web Viewer / Player
        ↓
Tauri or similar desktop/native wrapper
```

and potentially:

```text
Web Player
        ↓
Android WebView
```

The attraction of a WebView approach is that much of the Player/UI could remain shared.

However, if FreeTV eventually wants serious offline storage, SD-card access, native downloads, background behavior, or other native functionality, a native Android implementation may eventually make more sense.

The conclusion was not to commit to that architecture now.

The immediate preference is a **desktop-first standalone Viewer**, while mobile/native work remains deferred until the relevant technology and requirements justify it.

APK sideloading remains technically possible in the future; app-store distribution is not currently a requirement.

---

# 17. Native vs. Web Responsibilities

The broader architecture can therefore remain layered.

### Shared/core concepts

* FreeTV catalog;
* show/episode model;
* IA identifiers;
* file resolution;
* playback concepts;
* episode navigation;
* normalization rules.

### Web implementation

* browser video playback;
* local browser state;
* normal Web APIs.

### Future native implementation

Potentially:

* local filesystem;
* large offline libraries;
* downloads;
* native media controls;
* device-specific capabilities.

The goal is not to decide all of this now.

The goal is to avoid designing the current Viewer in a way that makes future evolution unnecessarily difficult.

---

# 18. What Should NOT Happen

Several architectural boundaries are worth preserving.

### Don't put everything into the FreeTV catalog

The catalog should not become a mirror of all IA metadata.

### Don't build analytics before we know what we need

Start with a small number of useful playback events.

### Don't build recommendations just because analytics exist

First determine whether aggregate viewing data actually provides useful signals.

### Don't make the current v3 refactor depend on the future Player

The server/data architecture should remain complete and useful without the future Player.

### Don't over-engineer mobile support now

The desktop/web Player is the immediate target.

### Don't solve every bad IA metadata case generically

Targeted normalization rules may be sufficient for valuable content.

---

# 19. Possible Future Player Architecture

A mature version could eventually look conceptually like this:

```text
                   FreeTV Published Data
                           │
                           ▼
                  ┌─────────────────┐
                  │     Viewer      │
                  │                 │
                  │ Catalog / UI    │
                  └────────┬────────┘
                           │
                     select episode
                           │
                           ▼
                  ┌─────────────────┐
                  │  FreeTV Player  │
                  │                 │
                  │ File Resolver   │
                  │ Normalizer      │
                  │ Playback        │
                  │ Episode Nav     │
                  │ State           │
                  │ Error Handling  │
                  └────────┬────────┘
                           │
                           ▼
                  ┌─────────────────┐
                  │ Internet Archive│
                  │                 │
                  │ Item / Files    │
                  └─────────────────┘
                           │
                           ▼
                       Playback
                           │
                           ▼
                  ┌─────────────────┐
                  │ Optional Events │
                  │ / Metrics       │
                  └─────────────────┘
```

This is intentionally a conceptual architecture, not an implementation specification.

---

# 20. Suggested Development Order

When the Viewer refactor begins, the ideas naturally fall into roughly this order.

## Phase 1 — Player foundation

* Replace iframe playback.
* Directly load IA media.
* Establish Player component architecture.
* Basic controls.
* Loading/error states.
* Fullscreen.
* Episode navigation.

## Phase 2 — Better FreeTV integration

* Episode/file resolution.
* IA metadata lookup where necessary.
* Episode/file-name normalization.
* Better playback errors.
* Show/episode context around the Player.

## Phase 3 — Local playback state

* Remember current episode.
* Remember playback position.
* Continue Watching.
* Local preferences/state.

## Phase 4 — Playback telemetry

* Define a minimal event model.
* Anonymous playback events.
* Aggregate statistics.
* Validate that the data is actually useful.

## Phase 5 — Content intelligence

Potentially:

* Popular Content.
* Trending content.
* Related content.
* Eventually recommendations, if the data justifies them.

## Phase 6 — Native/offline possibilities

Only if the need emerges:

* Downloads.
* Offline playback.
* Native storage.
* Tauri/native Viewer.
* Android/native capabilities.

---

# 21. Current vs. Future

The most important distinction from all of these discussions is:

### Current v3

**Goal:** finish the architecture and get FreeTV launched.

* MariaDB-backed Admin.
* Published static Viewer data.
* Current Viewer.
* Existing playback architecture.
* Minimal Viewer changes.

### Future Viewer / Player

**Goal:** make FreeTV's playback experience substantially better.

* Direct IA playback.
* Custom Player.
* Better episode/file handling.
* Metadata normalization.
* Playback state.
* Potential anonymous usage metrics.
* Popular Content.
* Potential recommendations.
* Eventually offline/native capabilities.

This separation was intentional.

We do not need to solve the future Player in order to finish v3.

---

# 22. The Big Idea

The common thread through all of these discussions is that the future Player is not merely a prettier video control.

It becomes the layer that understands the relationship between:

```text
FreeTV's curated catalog
          +
Internet Archive's actual media
          +
the person's playback experience
```

That creates opportunities that the current iframe-based approach cannot provide:

* better playback;
* better episode handling;
* smarter IA integration;
* meaningful playback state;
* useful aggregate viewing data;
* eventually better discovery;
* potentially offline/native experiences.

But none of those possibilities need to contaminate the clean v3 architecture.

The immediate objective remains:

> **Launch the stable v3 FreeTV architecture first. Then use the Player development work as the foundation for the next Viewer generation.**

