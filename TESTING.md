# Test-first verification

No billed endpoint was called during development or verification. Publication is separate from these test results.
A later, user-requested live smoke test is recorded at the end of this file.

## Red/green sequence

1. Added `tests/test_client.py` before `scripts/client.py` existed. Running
   `python3 -B -m unittest discover -s tests` failed with a missing-helper
   `FileNotFoundError` in each of the four service repositories.
2. Implemented the helper. The original nine safety tests passed in each repo.
3. Added six regression/workflow tests. The shared suite exposed two failures:
   result download could start after a just-expired polling deadline, and a
   preflight HTTP error was mislabeled `poll_error` instead of `blocked`.
4. Fixed both branches. Added the fenced-example syntax test before creating
   `STARTER_RELAY.md`; it failed because that required guide did not exist.
5. Created the guide and reran the full suite. Final results are reported below
   only after execution; no live submission/result is claimed by offline tests.

## Coverage

- Credential file and environment fallback, null/empty/missing values, CODDY_HOME.
- Starter subscription/status guard, exact non-premium operation, malformed and
  insufficient service quota, independent chat decision, double-page OCR pro cost.
- HTTP failures and ambiguous submissions with no retry or secret disclosure.
- Durable provider ID before polling; duplicate state creation refused.
- Failed statuses and deadline exhaustion never fetch results; unknown statuses keep polling.
- SHA-256 artifact provenance and untrusted-data labeling.
- Python and Bash fenced examples parsed without execution.

Official documentation and documented read-only quota endpoints were inspected;
only schema structure and public operation classification informed the fixtures.
SpeechCore remaining quota is unverified, so upload remains blocked. No end-to-end
paid test was performed. Artifact formats and submission contracts follow provider
documentation; the tests exercise mocked responses, not a claim of live success.

A further regression test exposed a CLI catch-all that hid sanitized block
reasons; it was fixed. A mocked workflow test covers all five image operations
as asynchronous jobs, including multipart processing rather than direct PNG POSTs.

## Unknown polling status

A live run on 2026-10-17 (four agents, real API) found that every OCR `run extract`
ended `failed` with "returned an unknown status", although the job had finished:
a later `GET /v1/ocr/jobs/{id}` said `completed`, and `resume` fetched the text.
`poll()` treated any status outside its fixed in-progress list as a failure.

Six tests were added first and failed: an unknown intermediate status now keeps
polling, an unknown status until the deadline ends `timed_out` naming it (only a
plain word is echoed), explicit failure statuses still stop at once, a response
without a status is a resumable `poll_error`, and a mocked OCR run passes through
an unknown status to `completed`. Mutations of the failure set, the status check
and the sanitising each fail their own test. **48 tests passed**, both normally
and with `LC_ALL=C PYTHONUTF8=0`, with no EncodingWarnings.

## Transcription models

The shared helper now knows three speech-to-text models for the `whisper`
service: `gigaam-v3` (the default), `whisper-1` and `whisper-podlodka-turbo`,
chosen with `--model`. Each has its own public price ID; the guard checks the
chosen model's row and its block names the model. `--model` is refused for the
other services and for unknown models (exit 2). Five shared tests were added
first; four failed and the fifth guards the new flag. Moving the default from
`whisper-1` to `gigaam-v3` updated the tests pinned to it. Mutations of the
default, the model check and the per-model price ID each fail their own test.
A live read-only `check` (2026-10-17) blocked each model at the quota step, so
all three price rows are present and non-premium. **53 tests passed**, both
normally and with `LC_ALL=C PYTHONUTF8=0`.

## Final results

- `python3 -B -m unittest discover -s tests`: **42 tests passed** in this repo
  (22 original plus 20 added in the notes below).
  The identical shared suite passed in all four service repos (88 total).
  Two Search-only `execute()` tests were later added here (24 total): request
  method/URL per operation, saved artifact, and `reconciliation_required` for
  error-shaped or wrong-shape Search responses.
  A later failing-first regression (25 total) requires `request()` to close
  the `HTTPError` response it catches instead of leaking it until GC.
- `python3 -B scripts/client.py --help`: passed, with no network call.
- `git diff --check`: passed.
- Fenced Python examples parsed with `ast.parse`; Bash examples passed `bash -n`.
- Live **read-only** CLI preflights returned `ready` for Search, OCR, and Images.
  SpeechCore exited nonzero with `blocked` and the unverified-quota reason.
  No account values were saved in this repository and no provider job was created.

Additional test-first fixes ensured early failures carry the full handoff fields
and interrupted HTTP response bodies become sanitized request errors rather than
uncaught exceptions. Both regressions were observed failing before their fixes.

Independent review found that an async error body could be saved as a completed
artifact. Added failing regressions, then validated PNG MIME/signature and JSON
MIME/result fields before artifact creation. Invalid or empty results now preserve
the provider ID with `poll_error` and no success artifact. PNG signature checks
are not a full image-decoding or visual-quality audit. Generated bytecode caches
are excluded by `.gitignore`. The full 88-test regression passed after these fixes.

Ported from neuraldeep-images: `multipart()` now keeps only an alphanumeric
file extension (`\.[A-Za-z0-9]{1,10}`) in the `filename=` header, so local
names with quotes, `;` or line breaks cannot inject header text. The
regression `test_multipart_filename_cannot_inject_headers` failed first
(4 injection cases); the suite is now **27 tests**.

Ported from neuraldeep-ocr: a reused `--state` path now blocks with "State
file already exists" instead of the generic CLI message, makes no HTTP call,
and leaves the existing file untouched. The updated duplicate-state test and the
new `test_existing_state_path_is_refused_with_clear_reason` failed first; the
suite is now **28 tests**.

Ported from neuraldeep-ocr: polling defaults now come from `POLL_INTERVAL = 2`
and `DEFAULT_TIMEOUT = 120` (used by `poll()` and `--timeout`, which gained help
text). `test_polling_defaults_come_from_shared_constants` failed first.
`scripts/client.py` is now identical across all four repos except `SERVICE`;
the suite is now **29 tests**.

## UTF-8 file reads

Added `test_text_files_are_read_as_utf8_on_non_utf8_locales` first. It runs the
helper in a C locale, whose default text encoding is ASCII, and failed: an auth
file with a non-ASCII field was reported as having no credential, and a UTF-8
`--payload` or state file raised `UnicodeDecodeError`. The helper and tests now
read and write text as UTF-8 explicitly. **30 tests passed**, both normally and
with `LC_ALL=C PYTHONUTF8=0`, with no `EncodingWarning` under
`-X warn_default_encoding`.

Ported from neuraldeep-images (beb9533): the 429 case in the HTTP-error test and seven tests: origin allowlist, no
key on public prices, refused redirects, timeout then resume from saved state,
refusal to resume non-resumable state, multipart layout, and submission routing.
They pin existing behavior, so they passed on first run; each was checked by a
targeted mutation of `client.py` in a scratch copy, which it caught. The images
CI workflow (`.github/workflows/tests.yml`: offline suite and `--help` on Python
3.10 and 3.x, with no credential in the environment) was copied unchanged; the
suite is now **37 tests**.

Ported from neuraldeep-whisper-1 (9fb4a33): `resume` of a hand-edited Search
state (status `submitted`, `timed_out` or `poll_error`) crashed with an uncaught
`UnboundLocalError` traceback, because polling was skipped and no result existed.
A `SYNCHRONOUS` service set now blocks `resume` for Search up front with
"Synchronous search has no resumable job endpoint" and a normal JSON envelope.
`test_resume_of_edited_resumable_state_is_blocked_not_crashed` failed first for
all three statuses; the suite is now **38 tests**.

Crossreview follow-up: `test_resume_of_edited_resumable_state_is_blocked_not_crashed`
now also asserts the saved state file is byte-for-byte unchanged. Without it, a
guard moved inside `execute()`'s `try:` block passed all 38 tests while rewriting
the hand-edited state to `blocked`; that mutant now fails for all three statuses.
The suite stays at **38 tests**.

Ported from neuraldeep-whisper-1 (36d8bc4): the shared helper now knows the
`whisper` service (`transcribe`, public price ID `whisper-1`). Like SpeechCore
uploads, it stays blocked because whisper-1 has no verified remaining-quota
endpoint. `test_whisper_is_blocked_without_verified_quota` failed first (the
guard reported an unrecognized schema); `scripts/client.py` is now identical to
whisper-1's apart from `SERVICE`, and Search behaviour is unchanged. The suite
is now **39 tests**.

Added AGENTS.md with instructions for coding agents (Codex, Coddy and others):
the shared-helper rule, the offline test commands, safety rules and the PR
workflow. No code or test changed; the suite stays at **39 tests**.

Ported from neuraldeep-speechcore (a9a8073) via neuraldeep-whisper-1 (eda8436):
`resume` of a malformed state (not an object, non-string `status`, or non-list
`errors`) now blocks with "State is not resumable" instead of crashing.
`test_resume_of_malformed_state_is_blocked_not_crashed` (patched to SERVICE
"speechcore", since Search refuses `resume` before reading state) failed first in
all five subtests. Two whisper tests were copied verbatim from whisper-1 to cover
the `SYNCHRONOUS` and `submission()` whisper changes; they pinned existing code, so
each was checked by removing its change in a scratch copy, which it caught.
`scripts/client.py` is identical to whisper-1's apart from `SERVICE`. The suite
is now **42 tests**.

## Live smoke test (2026-10-17)

At the user's request, the helper at `2bd7aa2` was run once against the live API.
Read-only `check web`, `check tg` and `check crawl` all returned `ready`. Then one
minimal billed request each (`limit` 1, a neutral public query, `https://example.com`
for crawl) returned `completed`. Each artifact's SHA-256 matched its envelope, the
state equalled the envelope, and the state and result files were created mode 0600.
Each request took exactly one unit from its subscription bucket (web and tg from
`search`, crawl from `crawl`). For web and crawl that held even though the response
reported `cost: 15`. Live refusals behaved as tested: a second `run` with the same
state path blocked with "State file already exists" before any request, and
`resume` blocked with "Synchronous search has no resumable job endpoint".
The Telegram response has no `success` or `cost` field and adds `search_type` and
`sources`, so SKILL.md's illustrative shapes were updated. No account values,
queries' results or runtime files were saved in this repository.

## Agent rule surfaces

Added `tests/test_agent_rules.py` (10 tests) before any rule file existed; all
10 failed because `.codex/hooks/attach_rules.py`, the rules and `CLAUDE.md` were
missing. Then added Cursor rules (`.cursor/rules/*.mdc`), their Claude Code
mirrors with identical bodies (`.claude/rules/*.md`), the Codex hook that attaches
the Cursor rules (`.codex/hooks.json`, `.codex/hooks/attach_rules.py`),
`CLAUDE.md` as a symlink to `AGENTS.md`, and the Delivery workflow and Rules Sync
sections in `AGENTS.md`. The tests check the pairing and scope mapping, that
every glob matches a file, and the hook itself: always-on rules at session start,
the client rule once per session for patch add, update, delete and rename and for
`Edit` and `Write`, and fail-open on bad input. Eight mutations (a widened or
narrowed scope, `paths` differing from `globs`, body drift, a missing mirror, a
dead glob, `.codex/rules.md`, a copied `CLAUDE.md`) each failed a test. The test
file and `.codex/hooks/` are identical across the neuraldeep-* repositories.
**58 tests passed**, both normally and with `LC_ALL=C PYTHONUTF8=0`, with no
`EncodingWarning` under `-X warn_default_encoding`.
