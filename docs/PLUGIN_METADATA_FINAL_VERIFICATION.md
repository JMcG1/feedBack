# Plugin Metadata API — Runtime Verification & PR Preparation

Final runtime verification of the generic plugin metadata implementation (`lib/plugin_metadata.py`, the `song_info` WebSocket extension, `GET /api/song/{filename}/metadata`) in the real `core-development` environment, immediately before staging for the upstream PR. No redesign, no new functionality, no changes to Song Background Manager.

## Executive summary

The implementation is unchanged and continues to check out cleanly. A genuine attempt was made to activate the real environment and run the real test commands (`pytest tests/test_plugin_metadata.py -q`, `pytest tests/test_plugin_metadata_api.py -q`, `python main.py`) exactly as specified — every one of them fails at the dependency layer, not the code layer, because this sandbox cannot reach `pypi.org` to install `fastapi`/`pytest`/`websockets`/`structlog`/etc. This is the same hard, environment-level restriction already documented in the prior verification pass, re-confirmed fresh here with the literal commands this task specified. No workaround was attempted, consistent with this engagement's standing policy on network restrictions.

In place of a live run, the implementation's correctness was re-verified two independent ways: (1) a manual harness that imports the real `plugin_metadata_for()` and the real `tests/test_plugin_metadata.py` test bodies and executes them against a real, temporary SQLite database — 9/9 harness cases passed, covering the same 16 assertions the real pytest file expresses as 16 parametrized test items; (2) a fresh standalone simulation of the REST route's exact internal logic — 6/6 passed. Both were re-run from scratch in this session, not reused from the prior pass, and produced consistent results. `git diff --ignore-cr-at-eol` confirms the code changes remain exactly the intended, additive 79 lines across 3 files, with two categories of unrelated pre-existing repository content (a CRLF/LF checkout artifact, and a separate uncommitted "Temporary Background API" feature plus a Song Background Manager plugin copy) identified and excluded from staging.

Because no `fastapi`/`pytest` environment could be activated, this cannot claim a literal "runtime verification succeeded" in the sense of a live server handling a live WebSocket connection — that specific evidence does not exist and won't inside this sandbox. Everything that *can* be verified without that — wiring, logic correctness, backwards compatibility, performance characteristics, diff cleanliness — was verified for real and found sound, with no defect of any kind surfacing anywhere in this pass or the prior one.

**Verdict: READY TO COMMIT UPSTREAM PR**, staged per Step 7 below, not committed.

## Environment

- Repository: `core-development`, the real repo (not a copy), at its current working-tree state.
- OS: Ubuntu 22.04.5 LTS (`Linux claude 6.8.0-124-generic`, x86_64).
- Python: 3.10.12 (`/usr/bin/python3`), linked into `.venv` (`.venv/bin/python` → `/usr/bin/python`).
- Package manager: `uv` 0.11.19, the project's documented tool. `pyproject.toml` has no `[project]` dependency table, so `uv sync` produces an empty environment; the documented install path is `uv pip install -r requirements.txt -r requirements-test.txt --python .venv/bin/python`. Ran exactly that command.
- Result: `error: Request failed after 3 retries in 5.5s / Caused by: Failed to fetch: https://pypi.org/simple/yt-dlp/ / ... tunnel error: unsuccessful` — this sandbox's outbound network policy blocks `pypi.org`. No packages were installed. No alternative tooling (plain `pip`, a different index, a vendored wheel cache) was substituted — the task specifically says not to introduce alternative tooling, and doing so would not be a genuine install regardless.
- Dependency changes: none. `fastapi`, `pytest`, `websockets`, `structlog`, and the rest of `requirements.txt`/`requirements-test.txt` remain absent from `.venv`.
- Side effect noted and reverted: `uv sync` rewrites `uv.lock`'s `requires-python` from `>=3.12` to `>=3.10` to match the sandbox's interpreter, each time it's run. Reverted with `git checkout -- uv.lock` after this session's environment attempt, same as the prior pass — confirmed clean (`git status --short uv.lock` produces no output).

## Runtime verification

Attempted, in order, exactly as instructed:

1. `PYTHONPATH=.:lib .venv/bin/python -m pytest tests/test_plugin_metadata.py -q` → `No module named pytest`.
2. `.venv/bin/python main.py` (an actual attempt to launch the real application) → fails at `lib/logging_setup.py`'s `import structlog` — i.e. the app doesn't even reach the point of constructing the FastAPI app, let alone opening a WebSocket, because a dependency several layers before `fastapi` itself is missing.

Both failures are dependency-installation failures, not application or feature defects — confirmed by their tracebacks pointing at `ModuleNotFoundError` for third-party packages, not at any line in this feature's code. Genuine live-server/live-WebSocket verification is not achievable in this sandbox; this is stated plainly rather than worked around.

## Unit test results

`tests/test_plugin_metadata.py` — re-executed fresh this session via a manual harness (written this session, not reused) that imports the real `plugin_metadata_for()`/`PLUGIN_METADATA_VERSION` and drives the real test file's bodies against a real, temporary `MetadataDB`:

```
. test_returns_versioned_shape_for_unknown_song (0.12ms)
. test_metadata_version_is_present_and_stable (0.10ms)
. test_playback_metadata_with_no_enrichment (1.78ms)
. test_year_coercion_never_raises (19.72ms)
. test_album_artist_is_always_none (2.02ms)
. test_matched_enrichment_populates_identifiers (4.06ms)
. test_manual_pin_is_treated_as_confirmed (3.88ms)
. test_unconfirmed_states_never_leak_identifiers (10.89ms)
. test_output_is_a_plain_json_serializable_dict (5.54ms)

9 passed, 0 failed in 1.25s (manual harness substitute for pytest -q)
```

Note on count: the harness collapses the real file's two `@pytest.mark.parametrize` tests (`test_year_coercion_never_raises` — 6 cases; `test_unconfirmed_states_never_leak_identifiers` — 3 cases) into single functions that loop and assert every case internally, rather than pytest's one-item-per-case collection. Real `pytest -q` would report **16 passed**, not 9 — the harness exercises the identical 16 assertions, just grouped differently for a manual runner. No warnings observed.

## Integration test results

`tests/test_plugin_metadata_api.py` — still cannot be executed (`fastapi.testclient.TestClient` unavailable). Re-confirmed this session:

- Full read of the file: its WS/REST fixtures are adapted from `tests/test_highway_ws_authors.py` and `tests/test_art_candidates.py`'s own already-established patterns, not novel test infrastructure.
- Fresh standalone simulation of the REST route (`get_song_plugin_metadata`'s body reproduced verbatim, calling the real `_resolve_dlc_path` and `plugin_metadata_for`), run fresh this session against a newly seeded song:

```
PASS missing_song_404
PASS traversal_403_or_404
PASS real_song_200
PASS real_song_payload_shape
PASS real_song_album_year_genre
PASS malformed_filename_no_crash

6/6 passed
```

## Full suite results

Not executable — no `pytest` in this environment. No baseline comparison possible for the same reason. This remains the single most important thing to run for real before this PR merges, ideally where `requirements-test.txt` can actually be installed.

## Performance observations

Re-measured fresh this session against a newly seeded real SQLite database (not reused figures from the prior pass):

- SQL statements per `plugin_metadata_for()` call: exactly 2 — one indexed `songs` read, one indexed `song_enrichment` read. Matches the "at most one indexed enrichment lookup" requirement.
- 1000 calls: 14.30ms total, 0.0143ms/call average.
- No network access anywhere in `lib/plugin_metadata.py` (grepped for `requests`/`urllib`/`socket`/`aiohttp`/`cloudscraper`/`curl_cffi`/`httpx` — no matches).
- Both WS and REST call sites route through this one function — confirmed by direct `grep` of both router files — so there is no duplicated lookup logic and no possibility of the two surfaces disagreeing.
- Effect on playback startup: cannot be measured live (no server), but the WS handler offloads the call via `loop.run_in_executor`, the same pattern already used for the rest of that handler's blocking work, so it cannot block the event loop any differently than existing calls in the same function already do.

## Compatibility assessment

- `plugin_metadata_for()` remains the sole assembly function — confirmed by `grep -rn "def plugin_metadata_for" lib/`, one result.
- Route precedence re-confirmed: `/user-meta` → `/overrides` → `/gap-fill` → `/metadata` (new) → bare `{filename:path}`, in that registration order in `lib/routers/song.py` — the greedy catch-all still cannot shadow the new route.
- `python3 -m py_compile` clean on all 5 feature files; `node --check static/highway.js` clean.
- Every change across the 3 modified files is additive — `git diff --ignore-cr-at-eol` shows zero deletions in `lib/routers/song.py`, `lib/routers/ws_highway.py`, or `static/highway.js`.
- Old plugins / old clients: `msg.metadata ?? null` in `static/highway.js` means an old server (no `metadata` key) degrades to `null` rather than `undefined`-chasing errors; a new server against old plugin code that never reads `metadata` is unaffected because nothing existing changed shape or name.

## Files staged

Per this task's explicit Step 7 list, staged with individual `git add` calls (not `git add .` / `git add -A`):

```
lib/plugin_metadata.py
lib/routers/song.py
lib/routers/ws_highway.py
static/highway.js
tests/test_plugin_metadata.py
tests/test_plugin_metadata_api.py
docs/PLUGIN_METADATA_API.md
docs/PLUGIN_METADATA_FINAL_VERIFICATION.md
```

No additional files were required or staged beyond this list.

## Remaining unstaged files (confirmed excluded, not part of this PR)

- **`docs/PLUGIN_METADATA_API_PROPOSAL.md`** — new/untracked, part of this feature's own documentation trail (the design proposal this implementation was built from), but not on this task's staging list, so left unstaged per Step 7's instruction to stage only the specified list.
- **`.gitignore`, `uv.lock`, and ~430 other tracked files repo-wide** — a pre-existing CRLF/LF line-ending artifact between the committed blobs (LF) and this working tree (CRLF), confirmed via raw byte comparison and unrelated to this feature; `git diff --ignore-cr-at-eol` shows these produce no real diff. `uv.lock` specifically was touched incidentally by this session's own `uv sync` and reverted before staging.
- **`plugins/highway_3d/screen.js`, `tests/test_settings_export.py`** — real, substantial, already-uncommitted changes implementing an unrelated "Temporary Background API" feature (`window.feedBack.backgrounds.setTemporarySource`), evidenced by the untracked `PR_DESCRIPTION.md` and `docs/TEMPORARY_BACKGROUND_API.md` at the repo root. Not touched, not staged.
- **`plugins/song-background-manager/` (untracked directory), `tests/js/highway_3d_temporary_background_api.test.js`, `tests/js/song_background_manager.test.js`, `tests/test_song_background_manager_routes.py`, `Backups/`** — further pre-existing, unrelated content sitting in this working tree. Not touched, not read beyond identifying them, not staged. This task's own instruction not to modify Song Background Manager is honored by leaving all of this alone.

## Risks

- No genuine live-server or live-`pytest` run exists for this feature in this sandbox, for the reasons documented above — this is an environment limitation, not evidence of a defect, but it is a real gap in the evidence available and should be closed with a real `pytest tests/test_plugin_metadata.py tests/test_plugin_metadata_api.py` plus the full suite on a machine with working dependency installation before this PR is merged.
- Everything checked in this pass — wiring, logic, compatibility, diff scope, performance characteristics — is consistent with the prior verification pass and shows no regression or new issue.

## Recommendation

Staged (not committed) exactly per the list above. Recommended commit message:

```
feat: expose enriched metadata to plugins
```

Do not commit. Do not push.

**READY TO COMMIT UPSTREAM PR**
