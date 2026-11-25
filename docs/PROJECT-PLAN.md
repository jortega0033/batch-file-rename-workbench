# Batch File Rename Workbench

A creative archive organizer can rename a small, staged set of files with a repeatable rule, inspect collisions before committing, and prove that an interrupted rename can finish without losing the original bytes.

## Product requirements

**Users and problem.** A photographer, media librarian, or editor may receive a set of generically named frames and want names that sort in a meaningful order. Applying a rule by hand is slow. A batch tool needs to explain why a target name is blocked and make the result easy to verify, particularly when two files trade names or a write stops partway through.

**First durable milestone.** Open the bundled fictional archive in a local browser, save a rule version, and generate a preview with original names, target names, and a signed-off count. Reopen the local server and verify that the same inventory and rule version produce the same plan fingerprint. No file moves are necessary for this milestone.

**End-to-end outcome.** Select a staged batch, choose a validated rule version, review all rows, generate a plan, confirm that exact plan, and see a completed result with original and final name-to-hash mappings. A simulated interruption leaves a visible recovery choice. After completion, the organizer can undo the latest eligible apply and compare its restored inventory to the original one.

**Scope.** Work entirely inside one application-created demo workspace on a local Linux filesystem. The first release accepts bundled synthetic file sets only. A flat visible collection contains up to 50 regular files with at most 64 KiB per file. The rules are literal prefix, literal substring replacement, ASCII case conversion, and zero-padded sequence naming. The journal and staging directories are private to the app workspace. A report is readable, reproducible data with no absolute host paths.

**Release boundary.** Folder traversal, metadata and media decoding, external archive import, shared folders, simultaneous external editors, deleting collection files by user request, OS integrations, cloud sync, and physical power-loss guarantees are outside this release. Manual access to staged files during a job is unsupported. Runtime flags, tests, and first-run text identify the collection as synthetic.

**Accessibility.** All apply and recovery actions are available with the keyboard. Row status appears in text, with focus routed to the first blocking issue or confirmation heading. A progress summary has a polite live region and a persistent status view. App confirmation describes the exact selected plan and count of moves. Undo uses a separate confirmation that states the inventory being restored.

**Proposed product decisions.** Keep names ASCII and selections explicit for the first release. Preserve extensions by default. Confirm future demand for Unicode, actual imported folders, and larger batches before expanding the safety and portability model.

## Technical design

### Runtime and domain

Use local Linux, Python 3 standard-library HTTP, SQLite, hashing, and filesystem modules, and bundled HTML, CSS, and JavaScript in Chrome. No runtime downloads or dependencies beyond those preflighted tools. Listen on 127.0.0.1 and serve the UI from that origin. Keep GET routes read-only and require the expected Origin, JSON content type, and bounded request bodies for POST mutations. Serve assets from a fixed registry and render filenames as text. Seed with a documented command into a newly created application-owned directory; refuse to reseed an existing workspace.

Model a workspace as a root with app-owned control data and two file areas. The visible `collection` directory has synthetic content. The separate unserved `staging` directory contains job-private slots on the same filesystem. The server resolves both from the staged workspace configuration, never from a browser-provided path. A startup capability check verifies directory-relative file calls, hard-link support, and directory syncing in a throwaway app-owned probe. Fail closed with a descriptive message if unavailable. Accept one server writer, using a process-lifetime workspace lock and an in-process mutation lock for requests. Use directory descriptors and verified, flat, regular-file entry names. Reject unexpected links, nested directories, devices, and external mounts during fixture intake. Fixture files start single-linked and are individually synced on creation. No operation reads or writes files outside this app-owned workspace. [Python filesystem interfaces](https://docs.python.org/3/library/os.html)

### Rule and inventory contract

An immutable rule version records label, operation, parameters, and selection ordering. A separate row maps the current rule label to its latest version. Changes create a new version. A version holds one operation:

- Prefix adds a non-empty literal to the stem. Substring replacement matches exact ASCII text in the stem and replaces every non-overlapping occurrence. An empty search string is invalid.
- Case applies ASCII upper or lower to the stem. The last extension is preserved exactly for prefix, substring, case, and sequence. For names without an extension, the full basename is the stem.
- Sequence replaces the stem with a literal prefix followed by a zero-padded decimal counter. Choose ascending or descending original-basename order explicitly. The counter increments by one per selected row. Require start 0 to 999999, width 1 to 6, and a final counter that fits the declared width. Sort comparisons use ASCII byte order.

Accept 1 to 120 ASCII bytes for visible basenames. Allow letters, digits, dot, underscore, and hyphen, with an initial alphanumeric and no trailing dot. Reject embedded separators and control characters before deriving a plan. Adopt an ASCII case-fold comparison key for projected names, including targets and unchanged entries. An input collection whose raw names already conflict under this key cannot be changed through the app. A change from one spelling to another spelling under the same key is allowed when the projected inventory has a unique owner under that key.

Archive files have stable synthetic logical IDs. Each snapshot row records ID, raw basename, size, and SHA-256. Inventory inspections compare the actual entry type, exact namespace, and hashes with the last stable revision. At a job boundary, the executor also records device and inode for each source and verifies them against current files when reconciling one partially complete move. Reject if input names, hashes, types, or required inode observations differ, and preserve existing data for review.

### Preview and job contract

The planner is a pure function of the full verified inventory snapshot, selected logical IDs, immutable rule version, and explicit ordering. Previews classify selected rows as changed, unchanged, or blocked; unchanged files still occupy their projected names. A target currently held by a selected row that will move is allowed when both files have unique names after projection. Generate exact-name and ASCII case-key owner groups. A raw-name conflict gets one `target_exists` or `duplicate_target` reason. Use `case_collision` when distinct raw names share a key. Collect and display affected IDs in stable order. Require at least one changed row to apply.

Hash the UTF-8 compact canonical JSON of schema version, stable revision, ordered full `[id, basename, size, sha256]` entries, selected IDs, rule version and parameters, and ordered projected rows with reasons. Object keys are sorted and all included identifiers and strings are ASCII. Record the SHA-256 hex digest as the plan fingerprint. Exclude presentation data and wall timestamps. The server recomputes the plan after acquiring the mutation lock, and compares both revision and fingerprint to the confirmed preview. A stale plan returns a typed conflict without creating a job. An idempotency key belongs to the operation and request body. The same key and body return the existing run; a changed body conflicts.

SQLite stores workspace revision, latest completed eligible apply, rule versions, immutable job plans, journal steps, and audit results. A frozen job records the old and new inventory, sequence and kind of each move, and one membership table. A completed apply bumps the workspace revision exactly once and becomes the latest eligible apply. An undo is a separately recorded inverse plan using the same mechanism. It is allowed only for the latest eligible apply, with exact post-apply inventory, no intervening completed mutation, and no unresolved job. Successful undo advances revision and disables redo. New drafts and rule saves do not mutate the collection.

### Staged execution and reconciliation

Acquire the workspace write lock, verify the complete steady inventory, then record a pending immutable job with a durable SQLite commit before changing any name. Persist the creation of its private staging directory. Process rows in a single stable order with no concurrent moves. Phase one moves all changed source names into distinct deterministic slots paired with their logical IDs. Only after all phase-one steps are complete does phase two move the slots to final names. The staging phase lets cycles and case-only moves release their old names before placing any target. Unchanged rows and staging entries for completed steps are not extra documents.

For each planned move, durably record the intent before creating the new link. Use an atomic no-clobber hard link into the same-device destination directory. If the link fails, preserve the source and mark the job blocked. Confirm the linked entry matches the intended device, inode and hash and sync the destination directory before unlinking the old entry. Sync the source directory before committing the move as complete. Finalized namespace verification checks all expected names and hashes and requires zero remaining staged entries. Commit the audit result and new revision together only after that check succeeds. A failure at finalization rechecks the exact inventory before replaying the final transaction and does not assign a second revision. [Linux link semantics](https://man7.org/linux/man-pages/man2/link.2.html) · [Linux directory synchronization](https://man7.org/linux/man-pages/man2/fsync.2.html)

Startup with a pending job permits reading status but prevents another apply or undo. After user confirmation, recovery checks the whole namespace relative to completed and later steps, including expected unchanged files, and classifies the one incomplete move:

| Observed old name | Observed new name | Action |
| --- | --- | --- |
| Expected regular file | Absent | Attempt the planned link. |
| Expected regular file | Same intended inode and hash | Sync the destination directory, remove just the old link, sync its directory, and record completion. |
| Absent | Expected intended inode and hash | Verify and sync both directories, then record completion. |
| Any other state | Any other state | Mark blocked, identify the mismatch, and preserve the state for inspection. |

Here, the named file is synthetic and the unlink step removes only its previous name after verifying a still-present destination link to the same bytes. Recovery resumes the same confirmed job and never changes its targets. A blocked job persists its last observed tally and prevents further mutation. Undo uses the same recovery path when interrupted. Crash claims cover process failure in a managed test workspace with the stated local filesystem assumptions. An unexpected unsupported state requires a new workspace for further rehearsals, leaving the stalled one available to inspect.

### Browser and reports

Expose typed endpoints for the stable inventory, rule versions, preview, apply, job status, resume, and undo. Mutations are explicit actions at the stated boundaries. Use concise stable error codes such as `preview_stale`, `target_exists`, `case_collision`, `job_recovery_required`, and `inventory_changed`, with logical row IDs where relevant. After confirming apply or undo, show a persistent job page with completed steps, current state, and a clear indication that name results are provisional until complete. Reloading a second tab shows the same job and recovery status. A compact downloadable JSON report includes operation and rule version IDs, plan fingerprint, final job status, exact before/after inventory, and the verified match counts.

## Original verification fixtures

All values below are expected results, independently derived from bounded fixture inputs. The working application and browser evidence are future work.

Start revision 1 with three regular synthetic files. Treat the hashes and file mapping as their full expected content contract.

| Logical ID | Original name | Exact bytes | SHA-256 |
| --- | --- | --- | --- |
| A | `frame-01.dat` | `abc` | `ba7816bf8f01cfea414140de5dae2223b00361a396177a9cb410ff61f20015ad` |
| B | `frame-02.dat` | `hello` | `2cf24dba5fb0a30e26e83b2ac5b9e29e1b161e5c1fa7425e73043362938b9824` |
| C | `notes.txt` | `keep` | `6ca7ea2feefc88ecb5ed6356ed963f47dc9137f82526fdd25d618ea626d0803f` |

**Cycle and undo.** Select A and B. Run sequence prefix `frame-`, start 1, width 2, descending order. B lands at `frame-01.dat` with its original hash. A lands at `frame-02.dat` with its original hash. C stays at `notes.txt`. Apply yields revision 2, three verified visible files, two changed rows, four completed move steps, and no staged names. Exact undo yields revision 3 and the original three-name hash mapping, adds an audit result, and leaves no staged names. A second undo is unavailable.

**Preview conflict set.** Select A and replace the literal stem substring `01` with `02`. B remains unselected and the plan blocks with `target_exists`. In a fixture variant, replace B with its same ID and bytes under `FRAME-02.dat`. The proposed `frame-02.dat` then blocks as `case_collision`. An unchanged prefix rule is invalid if its prefix is empty, while a valid lower-case rule on the already lower-case frame stems yields zero changed rows and no applicable plan. An edited rule version or changed stable inventory makes an old confirmation stale.

**Interrupted move.** Execute the cycle and stop after the first phase-two link is created but before its phase-two source name is removed. Define execution order as increasing logical ID within each phase. At the checkpoint, visible `frame-02.dat` and stage slot A refer to the `abc` inode. Stage slot B still contains `hello`. Visible `frame-01.dat` is absent and C is unchanged. Restart must show the pending job at revision 1. Resume clears the duplicate A link, completes the B move, and returns the same revision-2 name/hash set as uninterrupted execution. Run a separate crash checkpoint after source unlink before recording move completion and check the same output. A test-only mismatch variant replaces the current target with a different synthetic inode and bytes. Recovery refuses to touch either entry and reports the exact blocked step.

## Epic · Rename and recover a synthetic archive with proof

Deliver the offline editor, deterministic preview, verified rename, restart recovery, undo, and report under the synthetic workspace contract. Ship a local verification walkthrough covering the cycle, conflict, interruption, and undo fixtures. Estimated implementation and verification effort is **17–25 engineering hours**, across several focused sessions. The first durable preview spans FR-01 and FR-02. FR-03 creates verifiable execution, FR-04 adds recovery and undo, FR-05 integrates the keyboard-accessible browser journey, and FR-06 closes the release gate with independent evidence.

### FR-01 · Initialize the managed archive and persist rule versions

As an organizer, I want to start a fictional archive once and retrieve its inventory and saved rule versions after restarting.

**Estimate.** 2–3 hours. **Dependencies.** None.

- **FR-01-A.** A clean local fixture creates exactly A, B, and C under the app-owned collection. Produce the exact declared names, sizes, and SHA-256 hashes through an independent fixture manifest.
- **FR-01-B.** Reopening preserves revision 1 and both versions of a modified rule. Reinitializing an existing workspace fails before rewriting any file. Starting a second writer reports that the workspace is already in use.
- **FR-01-C.** Intake refuses nested or linked entries, an unsupported filesystem, and names outside the declared envelope. Use synthetic local inputs and verify the before/after inventory remains identical after rejection.

### FR-02 · Preview ordered plans and explain name conflicts

As an organizer, I want to see exactly which files would change or collide before I confirm a batch.

**Estimate.** 3–4 hours. **Dependencies.** FR-01.

- **FR-02-A.** Exercise all four operations with known inputs. Check extension preservation, source ordering, literal replacement, sequence width overflow, invalid names, and ASCII case-only moves.
- **FR-02-B.** Independently derive the cycle target map and both conflict cases. Verify exact blocked IDs and unchanged rows, including target names occupied by an unselected file.
- **FR-02-C.** Reopen the workspace and recompute the same fingerprint from the same inputs. A changed rule version, inventory, or selection changes the canonical fingerprint. Invalid and unchanged-only previews cannot execute.

### FR-03 · Apply a confirmed plan with staged no-clobber moves

As an organizer, I want an approved batch to move files without overwriting an existing entry or altering file contents.

**Estimate.** 4–6 hours. **Dependencies.** FR-02.

- **FR-03-A.** A successful cycle stages both files first and completes four ordered moves, producing the exact revision-2 name/hash set with an empty staging inventory.
- **FR-03-B.** Simulate an existing synthetic destination at the move boundary. The operation reports a blocked step, preserves both entries, and records no false completion. Exact stale revision or fingerprint mismatch before job start leaves files and job history unchanged.
- **FR-03-C.** Retrying the same confirmation key returns the existing job. A competing apply cannot start with another key while a job is pending. Verify completed or failed logical IDs through the structured audit output.

### FR-04 · Resume a stopped job and undo a completed batch

As an organizer, I want to finish a stopped job safely and restore the original names when the last batch is eligible.

**Estimate.** 3–5 hours. **Dependencies.** FR-03.

- **FR-04-A.** Stop at each declared link and journal checkpoint and restart against the same isolated fixture. Resume to the exact uninterrupted hash map with one completed job and one revision increment.
- **FR-04-B.** An observed mismatch marks the job blocked, names the discrepant row, leaves the checkpoint bytes intact, and prevents a new write. Check the same semantics from a reloaded browser state.
- **FR-04-C.** Undo the completed cycle and compare every visible basename and hash to revision 1. Verify the new revision and retained audit rows, refusal on a changed inventory, and recovery from an interruption during undo.

### FR-05 · Deliver a keyboard-accessible review and proof workflow

As an organizer, I want to review errors, confirm an exact preview, and find persistent recovery and results after a reload.

**Estimate.** 3–4 hours. **Dependencies.** FR-02 and FR-04.

- **FR-05-A.** In local Chrome, complete rule editing, preview, and confirmation using only the keyboard. Confirm the preview lists original names, targets, statuses, selected count, and the fingerprint.
- **FR-05-B.** Verify conflicts remain readable as text with an actionable focus target. Reload a pending job in another tab and inspect consistent recovery state and disabled competing actions.
- **FR-05-C.** Export the completed proof. Verify matching job, version, and fingerprint IDs, exact name-to-hash rows, and absence of absolute workspace paths from the JSON.

### FR-06 · Prove the original workflow in a clean offline setup

As a maintainer, I want independent evidence that the documented project can be rehearsed from a new workspace and verified later.

**Estimate.** 2–3 hours. **Dependencies.** FR-05.

- **FR-06-A.** Initialize the documented app workspace in a clean Linux checkout and run the synthetic browser demo with networking unavailable outside loopback. Record actual tool versions, results, and any skips.
- **FR-06-B.** Use an independent fixture verifier to compare exact name sets and per-file hashes before apply, after cycle, after interrupted recovery, and after undo. Count journal steps and verify each observed revision.
- **FR-06-C.** Document the operating assumptions, recovery choices, evidence locations, and known limits. Record that the run used synthetic fixture bytes and a newly created app-owned workspace throughout.
