# Spec: Hybrid LLM-Assisted Web Automation (NL → Script → Execute → Self-Repair)

**Goal:** Build a production-ready system that lets you describe browser workflows in high-level, natural-language steps (e.g., `Login({user},{password})`, `Click("Login")`, `Fill("Destination", value)`), uses an LLM to synthesize a **Playwright** script, executes the script **fast** on repeat runs, and if execution fails, falls back to an **LLM-guided recovery** that repairs the script and persists the fix.

**Primary outcome:** A containerized service (Cloud Run-ready) with:

* A simple API to submit a **Workflow Spec** (NL steps + site URL + secrets).
* A **Hybrid Engine**:

  1. **Plan** (LLM generates Playwright script),
  2. **Run** (execute Playwright),
  3. **Repair** on failure (LLM self-heals script),
  4. **Persist** updated script in a versioned store.
* **Tests** that validate: (a) LLM → script, (b) successful execution, (c) induced failure → LLM repair → re-execution passes.

---

## 1) Tech Choices

* **Language:** Python 3.11+ (or 3.12) for the orchestrator (fast to glue LLM + Playwright + storage).
* **Browser automation:** Playwright **Python** bindings (Chromium runner by default; allow Firefox/WebKit toggles).
* **Optional anti-bot:** run with “stealth” flags, proxy support, and pluggable CAPTCHA service (opt-in).
* **LLM provider:** abstracted interface; implement first with OpenAI/Anthropic (env-selectable).
* **Persistence:** Git-backed script store (local FS + git) or a DB table with file blobs; include semantic versioning (`workflow_id@vN`).
* **Runtime:** Dockerized, stateless; stores scripts in a mounted volume or cloud bucket (GCS) + revision metadata in Postgres.
* **Observability:** Structured logs (JSON), run IDs, step metrics, Playwright traces/screenshots on failure.

---

## 2) Directory Layout

```
/app
  /engine
    llm_planner.py          # NL → Script synthesis (and repair)
    executor.py             # Playwright runner (exec + trace + artifacts)
    repair.py               # Failure triage + fallback-to-LLM repair flow
    script_store.py         # get/put/list versions; Git or DB-backed
    prompts.py              # Prompt templates for plan/repair
    models.py               # Pydantic models for inputs/outputs
    config.py               # Env/config parsing (providers, proxies)
  /api
    server.py               # FastAPI (submit workflow, run, status)
  /tests
    test_hybrid_ok.py
    test_hybrid_fail_then_repair.py
    test_api_contracts.py
    fixtures.py             # play server/mocks, fake LLM, fake site
  Dockerfile
  pyproject.toml
  README.md
```

---

## 3) Core Data Models (Pydantic)

```python
# models.py
from pydantic import BaseModel, HttpUrl, Field
from typing import List, Dict, Optional, Literal
import datetime as dt

class Step(BaseModel):
    # High-level action with human-friendly targeting
    # action in {"Navigate","Login","Click","Fill","WaitFor","Select","Upload","Download","Screenshot","AssertText",...}
    action: Literal["Navigate","Login","Click","Fill","WaitFor","Select","Upload","Download","Screenshot","AssertText"]
    target: Optional[str] = None         # e.g., "Login", "Destination", CSS/XPath/role or natural label
    value: Optional[str] = None          # e.g., password or input text
    options: Dict[str, str] = {}         # e.g., timeout, containsText, role, exact, index

class WorkflowSpec(BaseModel):
    workflow_id: str                      # stable key
    url: HttpUrl
    steps: List[Step]
    credentials: Dict[str, str] = {}      # e.g., {"username": "...", "password": "..."} (resolved via secrets before run)
    browser: Literal["chromium","firefox","webkit"] = "chromium"
    headless: bool = True
    locale: Optional[str] = "en-US"
    user_agent: Optional[str] = None
    proxy: Optional[str] = None           # http://user:pass@host:port
    artifacts: bool = True                # save traces/screenshots on failure

class ScriptVersion(BaseModel):
    workflow_id: str
    version: int
    created_at: dt.datetime
    engine: Literal["playwright_py"] = "playwright_py"
    code: str                             # runnable Python file content
    notes: Optional[str] = None

class RunRequest(BaseModel):
    spec: WorkflowSpec
    reuse_existing: bool = True           # run latest script if exists, else create
    allow_repair: bool = True             # fallback to repair on failure
    max_repair_attempts: int = 1

class RunResult(BaseModel):
    run_id: str
    status: Literal["success","failed","repaired_success","repaired_failed"]
    script_version_used: int
    artifacts_path: Optional[str] = None
    logs: List[str] = []
    error: Optional[str] = None
```

---

## 4) LLM Prompts (plan + repair)

**4.1 Plan Prompt (NL → Script)**

* **System:**
  “You are a senior automation engineer. Output only **valid Python** using Playwright. The code must be self-contained and runnable as `python -m script` with a `run(spec: dict)` entry point. Use semantic locators first (roles, labels, placeholders, text), then robust fallbacks. Respect timeouts, retries, and write clear logs.”

* **User template:**

  ```
  TARGET_URL: {url}
  BROWSER: {browser} HEADLESS: {headless} LOCALE: {locale} USER_AGENT: {ua}
  PROXY: {proxy}
  WORKFLOW_STEPS (high-level):
  {steps_json}

  Requirements:
  - Implement `async def run(spec: dict) -> dict` that performs steps.
  - Use Playwright’s auto-wait and role/label-based queries (get_by_role/get_by_label).
  - For Login step: find username & password fields robustly (by label/placeholder/aria) and submit.
  - For Click/Fill: use resilient selectors; include try/except with alternate selectors if necessary.
  - For pop-ups/modals: detect common “new feature/what’s new” dialogs and close them before proceeding.
  - Return a result dict with keys: "status","notes","artifacts" (paths if any).
  - DO NOT include secret values in logs; read from `spec['credentials']`.
  - Avoid sleeps; use explicit waits (e.g., `locator.wait_for()` or `page.wait_for_url()`).
  - Ensure the script exits cleanly and closes the browser context.
  Output only code, no explanations.
  ```

**4.2 Repair Prompt (Failure context → Script patch)**

* **System:**
  “You are a senior automation engineer specializing in self-healing tests. Given failing logs, DOM snapshots (if any), and the current script, produce a corrected Python Playwright script, preserving the `run(spec)` contract. Prefer replacing only brittle selectors; add alternative strategies. Keep secrets out of logs. Output only code.”

* **User template:**

  ```
  CURRENT_SCRIPT (Python):
  {current_code}

  FAILURE_LOGS:
  {logs}

  OPTIONAL_HTML_SNIPPETS_OR_ACCESSIBILITY_SNAPSHOT:
  {snippets}

  TASK:
  - Diagnose root cause (e.g., changed aria-label, new modal, delayed loading).
  - Update selectors or add defensive checks (e.g., close "What's new" modal).
  - Keep structure and function signatures.
  - Improve resilience (use role/label/text with regex/contains, timeout increases where needed).
  - Output the full, updated Python file content only.
  ```

---

## 5) Execution Flow (Hybrid Loop)

```python
# orchestrator sketch
def run_workflow(run_req: RunRequest) -> RunResult:
    spec = run_req.spec
    script_ver = script_store.get_latest(spec.workflow_id)

    if run_req.reuse_existing and script_ver:
        code = script_ver.code
    else:
        code = llm_planner.plan(spec)            # PLAN
        script_ver = script_store.save_new(spec.workflow_id, code, notes="initial synth")

    # RUN
    result = executor.execute(code, spec)

    if result.status == "success":
        return finalize("success", script_ver.version, result)

    if not run_req.allow_repair:
        return finalize("failed", script_ver.version, result)

    # REPAIR LOOP
    attempts = 0
    while attempts < run_req.max_repair_attempts:
        attempts += 1
        repair_code = llm_planner.repair(current_code=code, logs=result.logs, dom_snippets=executor.last_dom())
        new_ver = script_store.save_new(spec.workflow_id, repair_code, notes=f"repair attempt {attempts}")

        run2 = executor.execute(repair_code, spec)

        if run2.status == "success":
            return finalize("repaired_success", new_ver.version, run2)

        code = repair_code
        result = run2

    return finalize("repaired_failed", new_ver.version, result)
```

**Key behaviors:**

* **Semantic selectors first, fallbacks second**.
* **Auto-close common modals** (look for “close”, “skip”, “got it”, “next time” buttons by role/name).
* **Artifacts on failure**: save Playwright trace, page screenshot, HTML snapshot, console/network logs.
* **No secrets in logs**: credentials loaded from `spec.credentials` and redacted in any error message.

---

## 6) Executor Requirements (Playwright Runner)

* Launch Playwright with:

  * `headless=true/false` from spec,
  * custom **user agent**, **locale**, **timezone**,
  * optional **proxy**,
  * strict **download directory** (mounted writable path),
  * **HAR** recording optional (config flag).
* Create a **fresh browser context** per run; **storage state** optional per workflow (toggle).
* Enable **tracing** (`trace:on-first-retry` or `trace:on` for test env).
* Provide helpers for robust actions:

  * `await page.get_by_role("button", name=re.compile("login|sign in", re.I)).click()`
  * Utility to find inputs by label/placeholder/aria and `fill`.
  * `close_common_modals(page)` scanning for known modal roles/names.
* When any step fails:

  * capture **screenshot**, **HTML**, **accessibility snapshot** (optional), and **console logs**.
  * return structured error with step context.

---

## 7) Script Store (Versioning)

* `save_new(workflow_id, code, notes) -> ScriptVersion`

  * increments semantic version (`v1`, `v2`, ...),
  * writes file `scripts/{workflow_id}/v{N}.py`,
  * commits to Git with message (or inserts DB row),
  * returns metadata.
* `get_latest(workflow_id) -> ScriptVersion|None`
* `list_versions(workflow_id) -> List[ScriptVersion]`

---

## 8) API Endpoints (FastAPI)

* `POST /run` — body: `RunRequest` → returns `RunResult`
* `GET /workflows/{id}/versions` — list script versions
* `GET /runs/{run_id}` — fetch status/logs
* `POST /workflows/{id}/scripts/{version}/run` — rerun a specific version
* `POST /workflows/{id}/invalidate` — optional: mark script obsolete (forces re-plan next time)

Auth for API is out of scope; assume service-to-service token header.

---

## 9) Security & Compliance

* Credentials injected at runtime (env/secret manager). Never log raw values.
* Artifacts sanitized (mask inputs visually where possible).
* Respect robots/ToS per your policy. Pluggable **rate limiter** and **proxy**.

---

## 10) Cloud Run / Container

* One container, stateless.
* Mount `/data` for `scripts/`, `artifacts/`.
* Health endpoints: `/healthz`, `/readyz`.
* Concurrency: 1–4 (tune based on memory); Chromium is heavy—expose `MAX_PARALLEL` env and a queue.

---

## 11) Minimal Contracts (Code Interfaces)

```python
# llm_planner.py
class LLMPlanner:
    def plan(self, spec: WorkflowSpec) -> str: ...
    def repair(self, current_code: str, logs: list[str], dom_snippets: dict|None) -> str: ...

# executor.py
class Executor:
    def execute(self, code: str, spec: WorkflowSpec) -> RunResult: ...
    def last_dom(self) -> dict|None: ...  # { "html": "...", "acc_tree": {...} }
```

---

## 12) Basic Example Workflow Spec

```json
{
  "workflow_id": "login_and_search",
  "url": "https://example.com/login",
  "steps": [
    {"action":"Navigate"},
    {"action":"Login", "target": "default", "value": null},
    {"action":"Fill", "target": "Destination", "value": "Montreal"},
    {"action":"Click", "target": "Search"},
    {"action":"AssertText", "target": "Results"}
  ],
  "credentials": {"username":"{{SECRET_USER}}","password":"{{SECRET_PASS}}"},
  "browser":"chromium","headless":true
}
```

*Notes:*

* `Login` means: find username/password inputs by label/placeholder/aria; submit form.
* `Navigate` means: `page.goto(spec.url)` and wait for network idle/document ready.
* `AssertText("Results")` means: wait for an element containing text “Results”.

---

## 13) Tests

### 13.1 Testing Strategy

* **Unit tests** for planner and repair (with a **FakeLLM** returning deterministic code).
* **Integration tests** with:

  * a **local demo site** (fixtures) that includes:

    1. a normal login form,
    2. a “What’s new” modal that sometimes appears (config flag),
    3. a renamed login button to simulate UI drift.
* **Induced failures**: first version intentionally uses a selector that will break; verify repair cycle updates the script and the next run passes.

### 13.2 Fixtures

* **Fake site** (FastAPI/Flask + Jinja) at `http://localhost:8088`:

  * `/login` → username+password fields:

    * v1 labels: “Email”, “Password”
    * v2 labels (toggle): “E-mail address”, “Passcode”
  * “What’s new” modal appears 30% of time or when query `?modal=1`
  * “Login” button sometimes “Sign in”
  * `/dashboard` on success; contains text “Results” after a search form submit

* **FakeLLM**:

  * `plan()` returns a script with a **brittle** selector for login button (`text="Login"`).
  * `repair()` replaces it with robust role/name regex.

### 13.3 Test: Happy path (no repair)

```python
# test_hybrid_ok.py
def test_hybrid_generates_and_runs_successfully(fake_site, fakellm, engine):
    spec = make_spec(url=fake_site.login_url, steps=[...], creds=...)
    run_req = RunRequest(spec=spec, reuse_existing=False, allow_repair=True)
    res = engine.run_workflow(run_req)
    assert res.status == "success"
    assert res.script_version_used == 1
```

### 13.4 Test: Failure → Repair → Success

```python
# test_hybrid_fail_then_repair.py
def test_brittle_selector_repaired(fake_site, fakellm_brittle_then_fix, engine):
    fake_site.set_variant("rename_login_button")  # break initial selector
    spec = make_spec(url=fake_site.login_url, steps=[...], creds=...)

    # First run: fails
    res1 = engine.run_workflow(RunRequest(spec=spec, reuse_existing=False, allow_repair=True, max_repair_attempts=1))
    assert res1.status in ("failed", "repaired_success", "repaired_failed")
    # If immediate repair attempted within run_workflow, expect repaired_success
    if res1.status == "repaired_success":
        assert res1.script_version_used == 2
        return

    # If engine returns failed (no auto-repair), explicitly trigger repair
    res2 = engine.run_workflow(RunRequest(spec=spec, reuse_existing=True, allow_repair=True, max_repair_attempts=1))
    assert res2.status == "repaired_success"
    assert res2.script_version_used == 2
```

### 13.5 Test: Modal handling

```python
# test_modal_close_flow.py
def test_handles_whats_new_modal(fake_site, fakellm_modal_aware, engine):
    fake_site.enable_modal()
    spec = make_spec(url=fake_site.login_url, steps=[...], creds=...)
    res = engine.run_workflow(RunRequest(spec=spec, reuse_existing=False))
    assert res.status == "success"
    # Verify logs mention modal closed / or artifact shows it
```

### 13.6 Test: No Secrets in Logs

```python
# test_sensitive_redaction.py
def test_redacts_credentials_in_logs(fake_site, fakellm, engine):
    spec = make_spec(..., creds={"username":"u","password":"p"})
    res = engine.run_workflow(RunRequest(spec=spec, reuse_existing=False))
    for line in res.logs:
        assert "u" not in line and "p" not in line
```

---

## 14) Implementation Notes & Guardrails

* **Selector strategy order:**

  1. Accessibility roles & names (`get_by_role("button", name=/login|sign in/i)`),
  2. Labels/placeholders (`get_by_label("Password")` / `get_by_placeholder("Email")`),
  3. Text contains / regex on visible text,
  4. Last resort: stable CSS (data-test ids) if present.
* **Auto-close modals:** maintain a small library of patterns:

  * buttons: `Close`, `Skip`, `Dismiss`, `Got it`, `Not now`, `No thanks`.
  * role=dialog → find dismissible elements.
* **Time-bounded waits:** never infinite; configurable default (e.g., 10–15s). Prefer `locator.wait_for(state="visible")` etc.
* **Retries:** allow 1 internal retry per critical action before failing a step.
* **Artifacts:** on any failure, save:

  * `artifacts/{run_id}/screenshot.png`
  * `artifacts/{run_id}/page.html`
  * `artifacts/{run_id}/trace.zip`
  * console/network logs
* **Performance:** reuse browser binary across runs (Playwright manages its driver cache). Keep memory low by **closing contexts** eagerly.

---

## 15) Example Generated Script Contract (Skeleton)

```python
# generated script example (Playwright, Python)
import re, json, asyncio
from playwright.async_api import async_playwright

async def run(spec: dict) -> dict:
    browser_name = spec.get("browser","chromium")
    headless = spec.get("headless", True)
    ua = spec.get("user_agent")
    proxy = spec.get("proxy")
    url = spec["url"]
    creds = spec.get("credentials", {})
    steps = spec["steps"]

    result = {"status":"unknown","notes":[],"artifacts":[]}
    async with async_playwright() as p:
        browser_type = getattr(p, browser_name)
        launch_args = {"headless": headless}
        if proxy: launch_args["proxy"] = {"server": proxy}
        browser = await browser_type.launch(**launch_args)
        context = await browser.new_context(locale=spec.get("locale","en-US"), user_agent=ua)
        page = await context.new_page()

        try:
            await page.goto(url, wait_until="domcontentloaded")
            await maybe_close_modals(page)

            for step in steps:
                action = step["action"]
                target = step.get("target")
                value  = step.get("value")

                if action == "Login":
                    await do_login(page, creds)
                elif action == "Click":
                    await click_by_name(page, target)
                elif action == "Fill":
                    await fill_by_label_or_placeholder(page, target, value)
                elif action == "AssertText":
                    await page.get_by_text(re.compile(target, re.I)).wait_for(timeout=15000)
                elif action == "Navigate":
                    pass
                # ... others ...
                await maybe_close_modals(page)

            result["status"] = "success"
            return result
        except Exception as e:
            # take artifacts here (screenshot/html) but do NOT include secrets
            result["status"] = "failed"
            result["notes"].append(f"ERROR: {type(e).__name__}: {str(e)[:300]}")
            # paths to artifacts added by executor wrapper
            return result
        finally:
            await context.close()
            await browser.close()

async def click_by_name(page, name):
    # role-based then text-based fallback
    btn = page.get_by_role("button", name=re.compile(name, re.I))
    try:
        await btn.click(timeout=8000)
        return
    except:
        await page.get_by_text(re.compile(name, re.I)).click(timeout=8000)

async def fill_by_label_or_placeholder(page, label, value):
    try:
        await page.get_by_label(re.compile(label, re.I)).fill(value, timeout=8000)
        return
    except:
        await page.get_by_placeholder(re.compile(label, re.I)).fill(value, timeout=8000)

async def do_login(page, creds):
    user = creds.get("username","")
    pwd  = creds.get("password","")
    # find username field
    u = page.get_by_label(re.compile("email|username|user name|e-mail", re.I))
    p = page.get_by_label(re.compile("password|passcode", re.I))
    try:
        await u.fill(user, timeout=8000)
    except:
        await page.get_by_placeholder(re.compile("email|username", re.I)).fill(user, timeout=8000)
    try:
        await p.fill(pwd, timeout=8000)
    except:
        await page.get_by_placeholder(re.compile("password|passcode", re.I)).fill(pwd, timeout=8000)
    # click login/sign in
    await click_by_name(page, "login|sign in")

async def maybe_close_modals(page):
    # Try a few common modal dismiss patterns
    for pat in ["close","skip","dismiss","got it","not now","no thanks","continue"]:
        try:
            await page.get_by_role("button", name=re.compile(pat, re.I)).click(timeout=1000)
            # attempt multiple dismisses
        except:
            pass
```

---

## 16) Acceptance Criteria

* **API**: `POST /run` accepts `RunRequest` and returns `RunResult`.
* **First-time workflow** with no saved script:

  * Uses LLM to produce Playwright code and executes it.
* **Subsequent runs** default to reusing last working script (fast path).
* **On failure**, **if `allow_repair`**:

  * System sends current script + failure logs (+ optional DOM snapshot) to LLM.
  * Receives updated script, saves `v+1`, retries, and succeeds on test site.
* **Artifacts** produced on failure and path returned.
* **Tests**:

  * Happy path passes.
  * Induced selector drift triggers repair path and passes on second run.
  * Modal test passes with modal enabled.
  * Logs do not contain raw secrets.

---

## 17) Optional Enhancements (post-MVP)

* **Accessibility snapshot feed** to LLM for more context during repair.
* **Heuristic self-healing** before LLM (e.g., try role→text→placeholder fallbacks automatically).
* **Playwright trace viewer links** in RunResult.
* **Script linting** step to ensure imports/awaits are correct before execute.
* **Storage state caching** (stay logged-in per workflow) if policy allows.
* **Proxy pool + rate limiting** per domain.
* **MCP integration** later for tighter NL-to-UI mapping.

---

## 18) How to Hand This to a Coding Agent

**Instruction:**
“Implement the above spec exactly. Produce a Dockerized FastAPI service with modules and tests as specified. Use Playwright Python. Provide a `Makefile` or `justfile` with: `setup`, `test`, `run-local`. Default to headless Chromium. Include a simple fake site fixture for tests. Ensure `pytest -q` passes locally. Expose `POST /run`. Return `RunResult` JSON. Keep secrets out of logs. On failure, persist artifacts to `/data/artifacts/{run_id}` and scripts to `/data/scripts/{workflow_id}/v{N}.py` with a simple version index. Implement a `FakeLLM` for tests and leave a real LLM provider class behind a feature flag.”

---

This spec gives your agent everything it needs to build the hybrid system end-to-end, including contracts, prompts, flow, storage, and tests that explicitly validate **plan → run → repair**.
