---

# 🐛 BugBox – Product Requirements Document (MVP v1.0)

---

## 1. Vision

BugBox is an **LLM-powered testing sidekick**. Developers write probe tests with explicit expectations, BugBox generates adversarial, metamorphic, and negative inputs, and shrinks failures into minimal counterexamples.

Unlike “wrapper-y” AI apps, BugBox is a **full-stack developer tool**:

* A Python package with a decorator + CLI for seamless test integration.
* A landing site with **sandbox (Pyodide)**, **static eval gallery**, and **developer docs (Nextra)**.
* A Cloudflare worker to serve eval artifacts.

The ethos: playful “trickster sidekick” vibes, but serious engineering under the hood.

---

## 2. Goals

* Ship a usable **Python package** (`bugbox`) installable from PyPI.
* Provide **3 input generation modes**: adversarial, metamorphic, negative.
* Implement **LLM-based shrinking** with caching and replay.
* Provide **assertion helpers** (`bugbox.expect`) to simplify probe tests.
* Offer **profiles** for developer ergonomics:

    * `quick` → fast local loop (MVP).
    * `ci` → replay-only deterministic mode (MVP).
    * `thorough` → deeper exploration (stretch).
* Launch **bugbox.dev** with:

    * Landing page (hero, quickstart).
    * Pyodide sandbox (pure functions only).
    * Static eval gallery (curated OSS demos).
    * **Developer docs** built with Nextra.
* Securely expose a **Cloudflare worker API** for eval snapshots.
* Seed with **3 OSS demo integrations** (`dateutil`, `python-slugify`, `emoji`).

---

## 3. Non-Goals

* Multi-language support (Python only at MVP).
* Rich developer telemetry/analytics.
* Persistent accounts or cloud storage.
* Complex rubric-based validation (LLM shrinker replaces this).
* Forking/re-writing upstream OSS tests.
* **Execution time budgets** in backend harness (deferred).
* **User-submitted evals** in gallery (friction, trust risk).
* **Native pytest plugin integration** (CLI-first design chosen instead).

---

## 4. Success Criteria

* ✅ PyPI package published.
* ✅ CLI + decorator working with cached replays.
* ✅ Assertion helpers (`bugbox.expect`) available and documented.
* ✅ Site live with sandbox, docs, and eval gallery.
* ✅ Cloudflare worker serving static eval JSON.
* ✅ 3 OSS demos showing real-world counterexamples.
* ✅ Blogpost/demo tweet with failure cards as screenshots.

---

## 5. Architecture

### Backend (`bugbox/`)

* Python package (`bugbox/`).
* Components:

    * **Decorator (`@bugbox`)** → marks probe tests with expectations.
    * **Assertion helpers** (`bugbox.expect`) → ergonomic predicates for probe assertions.
    * **Input generation** via LLM with JSON schema validation.
    * **Shrinking** via LLM, deterministically re-verified.
    * **Caching + replay** for CI (`--bugbox-offline`).
    * **Failure Cards** persisted for each confirmed failure.
    * **Profiles**: quick, ci, thorough.
* CLI (Typer): `bugbox run --profile=quick`.

### Frontend (`frontend/`)

* Next.js + Tailwind landing site.
* **Docs**: `/docs` powered by **Nextra** for developer-facing documentation.
* Pages:

    * `/` → hero, quickstart, PyPI link.
    * `/playground` → Pyodide sandbox (pure funcs only).
    * `/evals` → static gallery of curated OSS counterexamples.
    * `/docs/*` → developer docs.
* **No user telemetry**: gallery comes from static JSON you export.

### Worker (`backend/`)

* Cloudflare Worker exposing static endpoints:

    * `/api/health` → status check.
    * `/api/evals` → static JSON with eval snapshots.

---

## 6. Input Generation & Execution Controls

* **Defaults**

    * Adversarial → 6 cases.
    * Metamorphic → 3 pairs.
    * Negative → 5 cases.

* **Overrides**

    * `k` parameter (1 ≤ k ≤ 32).
    * Warn if shallow (<3 for adversarial/negative).

* **Execution Policy**

    * Stop on first failure (default).
    * `continue_on_fail=True` optional.
    * Shrink + cache counterexamples immediately.

* **Profiles**

    * `quick` → small counts, fast loop (MVP).
    * `ci` → replay-only deterministic (MVP).
    * `thorough` → deeper exploration (stretch).

* **Deferred**

    * Time budgets (`timeout_ms`, `max_duration_ms`) — future consideration only.

---

## 7. Assertion Helpers (`bugbox.expect`)

To reduce boilerplate, BugBox ships a small **expectation DSL** for writing probe tests.

### Example Usage

```python
from bugbox import bugbox
from bugbox.expect import expect

@bugbox(target=parse_date, mode="adversarial")
def test_parse_date(a, out, err):
    expect(out, err).ok_or(ValueError)

@bugbox(target=is_admin, mode="adversarial")
def test_is_admin(user, out, err):
    expect(out, err).boolean()

@bugbox(target=normalize, mode="metamorphic")
def test_normalize_idempotent(text, out, err):
    expect(out, err).idempotent(normalize)
```

### Helper Set (MVP)

* `ok_or(exc_type)` → result is valid OR raises exception.
* `raises(exc_type)` → must raise a given error.
* `boolean()` → must return a boolean (if no error).
* `idempotent(fn)` → applying `fn(fn(x)) == fn(x)`.
* `predicate(fn)` → pass a custom lambda condition.

---

## 8. Pytest Integration Model

* **Not a native pytest plugin** → BugBox does not register as a pytest plugin.
* Instead:

    * Users still write pytest-style test functions with `@bugbox`.
    * BugBox CLI (`bugbox run`) discovers and executes those probe tests.
    * CI runs BugBox separately (`bugbox run --profile=ci --bugbox-offline`) from the normal `pytest` suite.
* **Regression Promotion** turns BugBox failures into plain pytest tests → these become part of the deterministic suite.
* **Rationale**: separation avoids nondeterminism in CI, preserves trust, and clarifies exploratory vs deterministic layers.

---

## 9. Failure Cards & Regression Promotion

* **Failure Cards**

    * Produced for each confirmed failure.
    * Contains: shrunk input, observed vs expected, summary, root cause, suggested fixes, drop-in pytest test.
    * Saved in `.bugbox/cache/.../report.json`.
    * Console prints card in rich format.

* **Regression Promotion**

    * `bugbox promote <case_id> --out tests/test_regressions.py`.
    * Converts Failure Card into a permanent pytest test.
    * Adoption carrot: developers keep value even if they stop running BugBox.

---

## 10. Safety & Security

### Local usage

* User provides own API key.
* Injection risks limited to their key.

### Sandbox (public)

* **Pure functions only.**
* **Sanitization** on all prompts/outputs.
* **Strict JSON schema validation**.
* **Hard caps** on tokens, retries.
* **Low-quota API key** (ours) for public runs.
* **Ephemeral only**: no sandbox results flow into evals.

---

## 11. Evals / Gallery

* **Author-driven only** (no user telemetry).
* You run BugBox on curated OSS libs.
* Verified Failure Cards exported via:

    * `bugbox eval export` → `frontend/public/evals.json`.
    * `bugbox eval report` → optional Markdown for blog/README.
* Landing page shows these snapshots to build trust.

---

## 12. Developer Docs (Nextra)

### Structure

* `/docs` route with sidebar navigation (powered by Nextra).
* MDX-based content for copy-pastable code examples.
* Badges for “MVP” vs “Stretch.”

### Table of Contents

1. **Getting Started** – install, decorator usage, running BugBox.
2. **Test Patterns** – adversarial, metamorphic, negative.
3. **Assertion Helpers** – `ok_or`, `raises`, `boolean`, `idempotent`, `predicate`.
4. **Failure Cards** – what they look like, caching, replay.
5. **Regression Promotion** – export failures as deterministic pytest tests.
6. **Profiles** – quick, ci, thorough.
7. **CI Integration** – replay-only mode, nightly exploratory jobs, promotion workflow.
8. **Safety & Limits** – API keys, sandbox constraints, sanitization.

---

## 13. CLI Mocks

Key commands:

* `bugbox run` → executes probe tests, prints/shrinks failures.
* `bugbox cards list` → browse cached failures.
* `bugbox replay` → re-run a case deterministically.
* `bugbox promote` → convert failure card into pytest test.
* `bugbox eval export` → produce static JSON gallery.
* `bugbox eval report` → generate Markdown/HTML summary.
* `bugbox sandbox` → run a single target function quickly.

Example run:

```bash
$ bugbox run --profile=quick
[BugBox] Adversarial: parse_date (k=4)
✗ case #3 failed
  Input: "2025-13-40"
  Expectation: ok_or(ValueError)
  Observed: datetime.date(2026, 1, 9)

→ shrinking… minimal counterexample: "2025-13-01"
→ saved card: .bugbox/cache/55c9f2e1/report.json
```

---

## 14. CI Integration

### Default CI (Deterministic Replay)

* Blocking CI jobs must run **replay-only** to guarantee determinism.
* Command:

  ```bash
  bugbox run --profile=ci --bugbox-offline
  ```
* Ensures cached counterexamples are replayed exactly, with no new generations.
* Suitable for PR validation and main branch merges.

### Exploratory CI (Optional, Non-Blocking)

* Separate job (nightly/scheduled) may run BugBox in exploratory mode:

  ```bash
  bugbox run --profile=quick
  ```
* This job should be marked `continue-on-error: true` in GitHub Actions so failures do not block merges.
* Any new counterexamples are saved as **Failure Cards** and uploaded as build artifacts.

### Promotion Workflow (Human-in-the-Loop)

1. Developer downloads artifacts from exploratory job.
2. Runs locally to verify deterministically:

   ```bash
   bugbox replay <target> --case <id>
   ```
3. If valid, promotes to regression test:

   ```bash
   bugbox promote <target> --case <id> --out tests/test_regressions_bugbox.py
   ```
4. Commits the regression test (and optionally the cached card) to the repo.
5. From then on, the failure is covered in **deterministic CI**.

### Keys & Spend

* Replay jobs do not require an API key (offline).
* Exploratory jobs require an API key (e.g. from GitHub secrets).
* To control cost:

    * Use `--profile=quick` in CI.
    * Cap `k` in nightly runs.
    * Restrict exploratory jobs to once per day.

---

## 15. Deliverables

* Python package on PyPI.
* Example repo with 3 OSS demos.
* bugbox.dev site with:

    * Landing page.
    * Playground (Pyodide).
    * Eval gallery.
    * Developer docs (Nextra).
* Launch blogpost/demo tweet with counterexample screenshots.

---

## 16. Timeline (4 Weeks)

* **Week 1**: Backend scaffold + decorator stub + CLI skeleton.
* **Week 2**: Input modes, shrinking, caching, profiles, assertion helpers, failure cards.
* **Week 3**: Frontend scaffold + Pyodide sandbox + Worker API + Docs setup (Nextra).
* **Week 4**: OSS demos, docs content, CI guidance, polish, PyPI + site launch.

---

## 17. Decisions & Rationale

* BugBox is **CLI-first**, not a pytest plugin, to ensure deterministic CI.
* Developers write **probe/expectation-style tests**.
* **Assertion helpers** reduce friction.
* **Failure Cards** + **Regression Promotion** = durable adoption carrot.
* **Profiles** simplify DX.
* **Caching & replay** ensures reproducibility.
* **Default CI is replay-only** (deterministic). Exploratory CI is optional, non-blocking, and human-in-the-loop.
* **Time budgets deferred** for simplicity.
* **No user-submitted evals** → avoids friction/telemetry concerns.
* **No fine-tuning** → avoids scope creep.
* **Eval dashboard is creator-only** → site shows static eval snapshots.
* **Developer docs (Nextra)** → critical for adoption & hiring signal.
