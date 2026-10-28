# Payables QA

**A Python + Playwright test automation framework for an invoice approval app, with UI tests, API tests, CI, and a real bug it caught.**

![tests](https://github.com/hannoneg/payables-qa/actions/workflows/tests.yml/badge.svg)

I spent three years doing manual UAT on a federal financial system. This repo is what that work looks like automated: the same kinds of business rules (funding, duplicate invoices, segregation of duties), checked by code on every push instead of by hand before every release.

![Payables Portal](docs/app.png)

## What's here

| | |
|---|---|
| `app/` | **Payables Portal**, the system under test. A small Flask app: analysts submit vendor invoices, approvers approve or reject them. Browser UI plus a JSON API over the same rules. |
| `tests/ui/` | Playwright browser tests: login, invoice submission, approvals, role permissions. |
| `tests/api/` | HTTP-level tests of the same rules. Fast, so most edge cases live here. |
| `tests/pages/` | Page objects. Tests speak business language, and only page objects know selectors. |
| `.github/workflows/` | CI: smoke tests first (fail fast), full suite next, results posted to the run summary, screenshots and traces uploaded on failure. Also runs on a weekday schedule. |
| `BUGS.md` | Defect log, including **BUG-001**, found by this suite. |

**62 tests · ~35 seconds · 60 pass + 2 expected failures documenting an open bug.**

## Run it

```bash
python -m venv .venv && source .venv/bin/activate     # Windows: .venv\Scripts\activate
pip install -r requirements.txt
python -m playwright install chromium

pytest                 # everything
pytest -m smoke        # 6 critical-path tests, ~5s
pytest -m api          # no browser
HEADED=1 pytest -m ui  # watch the browser drive itself (mac)
$env:HEADED="1"; pytest -m ui
```

Want to click around yourself? `python -m app.payables` and open http://localhost:5000
(analyst: `ana` / `analyst-pass`, approver: `pat` / `approver-pass`).

When a UI test fails, `test-results/` gets a full-page screenshot and a Playwright trace.
Open the trace with `python -m playwright show-trace <file>.trace.zip` to replay every click, network call, and DOM snapshot.

## What the suite covers

- **Happy paths:** submit, approve, reject, list, and filter.
- **Validation and boundaries:** zero, negative, non-numeric, too many decimals, exactly $1,000,000 (allowed) vs. $1,000,000.01 (not), future dates, wrong date formats, inactive and unknown vendors, required fields.
- **Money correctness:** amounts round-trip exactly (`0.10` stays `0.10`), because they're stored as integer cents, not floats.
- **Business controls:** duplicate invoice prevention, a corrected invoice is allowed after a rejection, and no double approvals.
- **Security and access control:**
  - Analysts can't approve, even by crafting the POST themselves (hiding a button isn't access control).
  - **Segregation of duties:** an approver can't approve their own submission.
  - Every API endpoint rejects missing or bad tokens.
  - Login errors don't reveal whether the username or the password was wrong.
  - The post-login redirect can't be abused to send users to another site.

## How known bugs are handled

BUG-001 (the duplicate check is case-sensitive, so `brf-1001` slips past `BRF-1001`) has tests marked `xfail(strict=True)`. The suite stays green while the bug is open, but **the moment someone fixes it, those tests "unexpectedly pass" and the build goes red**, forcing the marker to be removed. The bug test then becomes a permanent regression guard. Details and a suggested fix are in [BUGS.md](BUGS.md).

## Design choices

- **Clean state per test.** An autouse fixture resets the database before each test, so there's no run-order dependence and no flaky shared state.
- **Real server, random port.** The app runs in a background thread for the session, so UI and API tests hit real HTTP, not mocks.
- **`data-testid` selectors.** Tests don't break when someone changes a CSS class or button label.
- **Playwright's built-in waits** (`expect(...)`) everywhere. No `sleep()` calls.
- **One validation function** shared by UI and API, and both layers are tested, so they can't silently drift apart.
- **The test reset endpoint only exists when `testing=True`**, so it can't ship to production.

## Roadmap

- [ ] Parallel runs with `pytest-xdist`
- [ ] Accessibility checks with axe-core
- [ ] Cross-browser matrix (Firefox, WebKit) in CI
- [ ] Load test of the API with Locust
- [ ] Dockerfile so CI and local runs are identical
- [ ] Unit test layer for `validate_invoice()` (complete the testing pyramid)
