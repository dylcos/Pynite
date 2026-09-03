# Pynite Contribution Tracker

> **This file is local bookkeeping only.** It lives on the `dev/contributions-tracker`
> branch and should **not** be merged into `main` or included in any PR sent upstream to
> `JWock82/Pynite`. Each proposal below gets implemented on its own branch cut from
> `main`, so this file never leaks into a real diff. Treat it as a personal project
> board that survives across sessions.

Owner: dylcos (fork of [JWock82/Pynite](https://github.com/JWock82/Pynite))
Created: 2026-09-02

## How to use this file

1. Before starting a proposal, re-read its **Collision risk** note and check the linked
   upstream issue for new comments — statuses can go stale between sessions.
2. `git checkout main && git pull && git checkout -b <branch>` for the proposal's own
   branch. Never build on top of `dev/contributions-tracker`.
3. Update the **Status** and **Notes** columns as work happens. Add a dated line under
   **Log** for anything future-you needs to know (a maintainer reply, a design decision,
   a reason something got dropped).
4. Statuses: `Not started` · `Coordinating` (waiting on an issue reply before starting) ·
   `Blocked` (waiting on something needed to move forward, e.g. a reproduction) ·
   `In progress` · `In review` (PR open upstream) · `Merged` · `Dropped`.
5. **Stale issues:** some tracked issues turn out to already be resolved — in code but
   never referenced back to the issue (like #190), or by someone else who just hasn't
   opened a PR yet (like #294). When that happens: mark it `Dropped`, note the evidence
   (commit hash, code reference) in its **Log**, and move on rather than re-investigating
   it in a future session. Don't treat a stale issue as a contribution opportunity for us.

## Repo conventions to hold every proposal to

From `CONTRIBUTING.md`, distilled:

- Comment heavily; PEP8-ish but **100-char lines**, not 79.
- Small, incremental PRs — break large features into a series of simple ones.
- Every feature PR needs an example `.py` in `Examples/` and tests in `Testing/`.
- Toolchain is mid-migration: `uv` for packaging (done), `ruff`/`prek` for linting
  (config sitting in unmerged PR #299 as of this writing — CI is still flake8-based on
  `main`), no type checker in CI yet.
- PRs tied to an active repo "Project" get priority — check the Projects board yourself
  before starting anything big; it wouldn't load for me when I checked.

## Index

| # | Proposal | Bucket | Status | Upstream issue |
|---|----------|--------|--------|-----------------|
| 1 | Fix Shear Wall - Advanced screenshot crash | Bug / quick win | Blocked — pending reproduction | [#324](https://github.com/JWock82/Pynite/issues/324) |
| 2 | Add pyrefly/ty type checking to CI + prek | Ongoing effort | Not started | [#327](https://github.com/JWock82/Pynite/issues/327) |
| 3 | Linear eigenvalue buckling analysis | New feature | Not started | [#309](https://github.com/JWock82/Pynite/issues/309) |
| 4 | Run `Examples/` as part of the test suite | Quick win | Dropped — stale, addressed elsewhere | [#294](https://github.com/JWock82/Pynite/issues/294) |
| 5 | Close out local-axes visualization | Quick win | Dropped — stale, comment posted | [#190](https://github.com/JWock82/Pynite/issues/190) |
| 6 | Cross-drive (`E:`) path crash | Quick win (unconfirmed) | Not started | [#285](https://github.com/JWock82/Pynite/issues/285) |
| 7 | conda-forge release | Infra, no code | Not started | [#200](https://github.com/JWock82/Pynite/issues/200) |
| 8 | Performance: matrix assembly / discretize / merge_duplicate_nodes | Performance | Coordinating | [#269](https://github.com/JWock82/Pynite/issues/269), [#251](https://github.com/JWock82/Pynite/issues/251) |
| 9 | Timoshenko beam elements | New feature | Coordinating | [#250](https://github.com/JWock82/Pynite/issues/250) |
| 10 | pydantic-based model serialization | New feature (needs buy-in) | Coordinating | [#274](https://github.com/JWock82/Pynite/issues/274) |

Suggested order: **1 → 2 → (comment on 8 and 9) → 3**. Item 1 is small and builds trust
fast; item 2 is unclaimed and low-risk; items 8 and 9 need a maintainer/contributor reply
before any code is written; item 3 is the biggest legitimate feature opening. Items 4 and
5 turned out stale — see their sections and the stale-issues policy above.

Item 1 is currently blocked — couldn't reproduce the reported crash (see its section
below), so it's paused pending a reproduction attempt on different hardware. **2** is the
next thing to actually pick up in the meantime.

---

## 1. Fix Shear Wall - Advanced screenshot crash

- **Bucket:** Bug / quick win
- **Status:** Blocked — could not reproduce; paused until reproduced elsewhere
- **Upstream issue:** [#324](https://github.com/JWock82/Pynite/issues/324)
- **Branch (when started):** `fix/shear-wall-screenshot-crash`

**Original theory (from reading the code only, not verified):**
`ShearWall.screenshots()` calls `Renderer.screenshot(..., interact=True)`. `screenshot()`
calls `render_model(interact=True)`, which — when `interact=True` — blocks on
`interactor.Start()` and then calls `window.Finalize()` **before returning**
(`Pynite/Visualization.py:308-361`). Back in `screenshot()`, `w2if.SetInput(window)` then
runs on an already-finalized window (`Pynite/Visualization.py:381-384`), theorized to
cause the reported `TypeError`.

**This theory did not survive reproduction.** Ran the real `Examples/Shear Wall -
Advanced.py` end-to-end (`uv sync --extra all --all-groups` → `vtk==9.6.2` per
`uv.lock`) under `xvfb-run`, with `vtkRenderWindowInteractor` subclassed so `Start()`
auto-terminates instead of blocking forever (no real user/display to close the window
in this environment). This exercises the exact call the issue's traceback names
(`ShearWall.py:825`, `interact=True, reset_camera=True`). Result: **completed cleanly,
both `screenshot()` calls produced valid non-trivial PNGs (750×750), no `TypeError`.**
So `Finalize()`-before-`SetInput` does not throw on this VTK/platform combination —
the original theory is disproven, not confirmed.

**What's still real, reproduction aside:** `self.window` is created once in
`Renderer.__init__` (`Visualization.py:71`) and reused across every `render_model()`/
`screenshot()` call on that instance. `ShearWall.screenshots()` calls `screenshot()`
twice on one `Renderer` (`interact=True` then `interact=False`), and `render_model()`
unconditionally finalizes that shared window whenever `interact=True` — finalizing a
window that's about to be reused is fragile lifecycle management regardless of whether
it crashes here. The pyvista backend (`Rendering.py`) doesn't have this problem because
`pv.Plotter.screenshot()` owns the window lifecycle internally; the vtk backend hand-rolls
it. Also notable: **no existing test exercises `interact=True` at all** — every call in
`Testing/test_Visualization.py` uses `interact=False`, so this path has zero coverage,
which is plausibly why the regression (from commit `22459e6`, switching the example from
`pyvista` to `vtk`) shipped unnoticed.

**Revised plan, once reproduced:**
1. Ask the reporter (jonbiemond) for OS + `vtk.vtkVersion.GetVTKVersion()` — not done yet.
2. TDD: add a test mirroring `ShearWall.screenshots()`'s exact pattern (one `Renderer`,
   `screenshot(interact=True)` then `screenshot(interact=False)`) asserting both produce
   valid images — this is genuinely missing coverage either way.
3. Fix the window-reuse lifecycle: add `finalize: bool = True` to `render_model()`;
   `screenshot()` passes `finalize=False`, captures the image, finalizes only when the
   caller won't render again.
4. Verify against the real example the same way it was reproduced (Xvfb + auto-closing
   interactor), plus the new test.
5. PR should be honest that it fixes a verified structural issue, not a confirmed
   reproduction of the reporter's exact error — ask them to confirm once it's up.

**Collision risk:** none found.

**Log:**
- 2026-09-02: Root cause theorized from reading the code (not yet verified).
- 2026-09-03: Attempted reproduction — failed. Original theory disproven. Real structural
  issue (window reuse + premature finalize) still stands, but paused per user request
  until they can attempt reproduction on different hardware. Do not resume until either
  it reproduces somewhere, or the reporter provides environment details on the issue.

---

## 2. Add pyrefly/ty type checking to CI + prek

- **Bucket:** Ongoing effort
- **Status:** Not started
- **Upstream issue:** [#327](https://github.com/JWock82/Pynite/issues/327) (umbrella: uv + ruff + pyrefly toolchain)
- **Branch (when started):** `chore/typecheck-pyrefly`

**Status of the other 3 workstreams in #327** (so this doesn't get duplicated):
- Packaging (uv) → done, merged.
- Linting (ruff/prek) → PR #299 is open, maintainer-approved ("this one can be merged"),
  **not yet merged**. `.flake8` and flake8-based CI are still live on `main`. Don't touch
  this workstream — just build on top of it once it lands.
- CI/CD unification → likely folds in with whichever of the above lands last.
- **Type checking → unclaimed.** This is the opening.

**Plan:** introduce `pyrefly` (or `ty`) in lenient/non-blocking mode, wire into CI and
the `prek` config the same way PR #332 documented for ruff. Should be rebased on top of
#299 once it merges, since it'll want `.pre-commit-config.yaml` to already exist.

**Collision risk:** low, but blocked on #299 merging first for the pre-commit hook
integration — the CI-only piece could start independently.

**Log:**
- 2026-09-02: Identified as the unclaimed workstream in #327. Not yet started.

---

## 3. Linear eigenvalue buckling analysis

- **Bucket:** New feature
- **Status:** Not started
- **Upstream issue:** [#309](https://github.com/JWock82/Pynite/issues/309)
- **Branch (when started):** `feature/buckling-analysis`

**What's requested:** linear eigenvalue buckling — geometric stiffness matrix assembly,
generalized eigenvalue solve, critical load factor, buckling mode identification. The
issue's own author proposes a shortcut ("just add the buckling formula to axial
reactions") that is **not sufficient** — real per-element geometric stiffness (Kg)
assembly and a generalized eigenproblem (`scipy.linalg.eigh(K, Kg)` or similar) are
needed for this to be structurally meaningful.

**Plan:** scope as a multi-PR series per CONTRIBUTING's "keep it simple" guidance —
e.g. (a) Kg assembly per `Member3D`, (b) global Kg assembly + generalized eigensolve on
`FEModel3D`, (c) results API + example + tests. Needs an `Examples/` file per
convention.

**Collision risk:** none found. Unclaimed, unassigned, no linked branch.

**Log:**
- 2026-09-02: Scoped during initial review. Not yet started.

---

## 4. Run `Examples/` as part of the test suite

- **Bucket:** Quick win / ongoing effort
- **Status:** Dropped — stale, already addressed by the original reporter
- **Upstream issue:** [#294](https://github.com/JWock82/Pynite/issues/294)

**What was requested:** run everything in `Examples/` during tests to catch API breakage
(not to validate numerical correctness).

**Why dropped:** jonbiemond (the original requester) already wrote it themselves —
commit [`51ce212`](https://github.com/jonbiemond/Pynite/commit/51ce2127cde98831197edcb73552394362bc5fe8)
on their own fork, message "Run examples as part of test suite — resolves issue
JWock82#294". Adds `Testing/test_examples.py`: `autouse=True` fixtures mocking
`Renderer.render_model` (both backends), `matplotlib.pyplot.show`, and
`Reporting.create_report` so examples run headless, then a
`@pytest.mark.parametrize`d test that discovers and `exec()`s every `Examples/*.py`.
**No PR opened against upstream yet** (checked JWock82/Pynite's open PR list — not
there), so this isn't merged or even in review, just clearly claimed. Building our own
version now would be redundant and would compete with the person who already did the
work and referenced the issue directly.

**If revisited:** only if jonbiemond's fork goes dormant for a long stretch with no PR —
check their fork/the issue again before ever restarting this from scratch.

**Log:**
- 2026-09-02: Identified as unclaimed. Started scoping.
- 2026-09-03: User found jonbiemond already implemented and referenced the issue in
  their own fork's commit. Dropped — not ours to build.

---

## 5. Close out local-axes visualization

- **Bucket:** Quick win / housekeeping
- **Status:** Dropped — stale, comment posted by user
- **Upstream issue:** [#190](https://github.com/JWock82/Pynite/issues/190)

**Finding:** this was already implemented in code (not referenced back to the issue).
`VisLocalCsys` (`Pynite/Visualization.py:2698`) draws colored X/Y/Z arrows at member
midpoints via the public `renderer.member_csys = True` toggle (landed in `1ad9fb6`,
auto-sizing refined in `e728613`), documented in `docs/source/rendering.rst`. Generated
a demo figure with the real API (`renderer.member_csys = True` on a 3-member frame,
proper `vtkCamera` isometric framing, auto-computed `annotation_size`) confirming it
works exactly as the issue asks.

**Resolution:** user posted a comment on the issue themselves pointing to the existing
implementation. Nothing further for us to do here — leave it to the maintainer to close.

**Log:**
- 2026-09-02: Confirmed already implemented in code.
- 2026-09-03: Generated a demo figure proving the feature. User commented on the issue
  directly. Dropped — done from our side.

---

## 6. Cross-drive (`E:`) path crash

- **Bucket:** Quick win, unconfirmed
- **Status:** Not started
- **Upstream issue:** [#285](https://github.com/JWock82/Pynite/issues/285)
- **Branch (when started):** TBD pending investigation

**Finding:** `ValueError: path is on mount 'C:', start on mount 'E:'` is the classic
message from `Path.relative_to()`/`os.path.relpath()` across Windows drives. I grepped
all of `Pynite/` for `relpath`, `relative_to`, `commonpath` and found **nothing** — the
only `abspath`-style usage is in `Reporting.py`, which isn't in the reported example's
call path. This strongly suggests the bug is in a **dependency** (pyvista/vtk/trame),
not Pynite's own code.

**Plan:** do not start coding. Ask the reporter (or reproduce locally on a Windows VM)
for the **full traceback** before doing anything — this may only be fixable with a
workaround (e.g. forcing an absolute-path config on the render/trame call), not a real
code fix in Pynite.

**Collision risk:** none, but low confidence this is actionable at all without more
info.

**Log:**
- 2026-09-02: Investigated; root cause not found in Pynite's own code. Needs more
  info before starting.

---

## 7. conda-forge release

- **Bucket:** Infra, no code
- **Status:** Not started
- **Upstream issue:** [#200](https://github.com/JWock82/Pynite/issues/200)

Packaging-only (conda-forge feedstock + recipe), no library code changes. Good option
if you want a contribution that doesn't touch the FEA internals at all. Not investigated
in depth yet.

**Log:**
- 2026-09-02: Listed as an option, not investigated further.

---

## 8. Performance: matrix assembly / discretize / merge_duplicate_nodes

- **Bucket:** Performance
- **Status:** Coordinating — do not start coding against these functions
- **Upstream issues:** [#269](https://github.com/JWock82/Pynite/issues/269) (umbrella), [#251](https://github.com/JWock82/Pynite/issues/251) (repro model, no response yet)

**Finding:** confirmed `FEModel3D.merge_duplicate_nodes` (`FEModel3D.py:953`) is a real
O(n²) nested loop doing pairwise distance checks — a legitimate KD-tree target. But
**this exact function, plus `PhysMember.discretize`, `Analysis._partition`
(csr_matrix conversion), and caching in `Quad3D`/`Member3D`, are already being built on
an active fork branch** (`Apex-Engineers-Inc/Pynite`, tied to #269) with reported
1.78x-3.5x speedups on parts of the test suite. Sending a competing PR against the same
functions wastes effort and guarantees merge conflicts.

**Plan:**
1. Comment on #269 offering to help test/review once a PR is opened. Low-risk, useful.
2. If independent performance work is wanted now, profile something **not** mentioned
   in #269 — e.g. `Rendering.py`/`Visualization.py` actor creation (one VTK actor per
   element currently, no batching) or `Reporting.py`. Unconfirmed as real bottlenecks —
   profile before proposing a fix, don't guess.

**Collision risk:** high on the functions named in #269. Low on anything else,
untested.

**Log:**
- 2026-09-02: Confirmed collision with active WIP branch. Holding — comment on #269
  before doing anything here.

---

## 9. Timoshenko beam elements

- **Bucket:** New feature
- **Status:** Coordinating — confirm status before starting
- **Upstream issue:** [#250](https://github.com/JWock82/Pynite/issues/250)

Labeled `in progress` but has **no linked branch, PR, or assignee** — the label may be
stale rather than reflecting real ownership. Don't assume it's free just because
nothing's linked.

**Plan:** comment on the issue asking whether anyone (maintainer included) is actively
working on it before scoping implementation.

**Log:**
- 2026-09-02: Flagged ambiguous status. Not started, no comment posted yet.

---

## 10. pydantic-based model serialization

- **Bucket:** New feature, needs buy-in
- **Status:** Coordinating
- **Upstream issue:** [#274](https://github.com/JWock82/Pynite/issues/274)

Framed by its own author as a "big idea" — a serialization format change like this
needs maintainer agreement on the approach before any code, not a cold-start PR.

**Plan:** not being pursued unless/until there's a design discussion on the issue with
maintainer input.

**Log:**
- 2026-09-02: Listed, deliberately not scoped further pending maintainer discussion.
