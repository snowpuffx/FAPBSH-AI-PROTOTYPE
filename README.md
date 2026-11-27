# FAPBSH-AI-PROTOTYPE
# FAPBSH AI IDE: analysis and verification upgrade

## How to run

```bash
python -m venv venv
venv\Scripts\activate            # Windows  (macOS/Linux: source venv/bin/activate)
pip install -r requirements.txt   # PySide6 + bandit, semgrep, pytest (the last three are optional)
python main.py
```

Open a `.py` file, press **Analyze** (or Run → Analyze Security, Ctrl+Shift+A).
Try `tests/samples/demo_app.py`, which contains all five CWEs.

* **Findings** cards drive everything else. Click a card or a Problems row to show the trace in the editor.
* **Recommendation** shows the original and corrected code. **Apply** is one undo step (Ctrl+Z), **Copy** puts the corrected file on the clipboard, and **Ignore** hides the finding for this session.
* **Verification** runs automatically the first time you open the tab for a finding, and again after Apply. **Re-run** repeats it.

## How to run the tests

```bash
python -m pytest -q tests
```

The suite has 44 tests: engine, classifier, recommender and verification.
The engine tests do not need Qt. Some verification tests call bandit and semgrep when they are installed.

## Architecture

| Layer | Package | Qt? |
|---|---|---|
| Taint analysis (`ast`), CWE-798 detector, models | `fapbsh/analysis/` | no |
| `Classifier` interface + `RuleBasedClassifier` | `fapbsh/classification/` | no |
| Explanation + fix templates + unified diff | `fapbsh/explain/` | no |
| Five checks, verdict, isolated harness, Semgrep rules | `fapbsh/verification/` | no |
| Workers (QThread), right panel, Problems list | `ui/analysis/` | yes |

To add a model-backed classifier later, subclass `fapbsh.classification.Classifier`, implement
`classify(evidence) -> Classification | None`, and pass it to `analyze_source(..., classifier=...)`
(called in `ui/analysis/workers.py`). The UI only ever sees `Finding` objects.

## Changes to existing files

| File | Change |
|---|---|
| `main.py` | Applies `GRAYSCALE_THEME` instead of `DARK_THEME`. `DARK_THEME` is still in `theme.py`. |
| `ui/theme.py` | Added `GRAYSCALE_THEME`, `GRAY` and `EDITOR_COLORS`. |
| `ui/main_window.py` | Added a Run menu (Run F5, Stop Shift+F5, Analyze Ctrl+Shift+A). Analyze and Run buttons now sit at the right of the menu bar, replacing the old toolbar row. The custom header row is hidden. The right panel lives in a splitter. Problems is now a clickable list. The run console moved into **Output**, and the unused empty Output box was removed. The status bar shows "N findings". Added analysis, Apply, Copy and Ignore handlers. Worker threads are joined on close. |
| `ui/editor/code_editor.py` | Colours now come from the theme. Added `set_trace` / `clear_trace` (line highlights, numbered badges, arrows, wavy sink underline), `go_to_line`, and `apply_text` (one undo step). Font falls back to monospace when Consolas is missing. |
| `ui/terminal/terminal_widget.py`, `ui/welcome_widget.py` | Inline colours switched to grayscale. No behaviour changes. |
| `requirements.txt` | Added bandit, semgrep and pytest. |

Nothing was deleted or renamed.

## Verdict rules

* **Not resolved**: syntax fails, or the data-flow re-check or the security test cases still show the vulnerability.
* **Partially resolved**: data-flow and payloads pass, but Bandit/Semgrep or the functional tests fail.
* **Inconclusive**: nothing failed, but neither independent check (tools, payload tests) could run.
* **Resolved**: every available check passed and none failed.

A missing tool is reported as **Unavailable**, never as Passed. Classifier confidence is not evidence.

## Known limitations

* **Taint scope.** Tracking covers one file. It follows calls between functions in that file through small summaries, plus `self.method()` calls. It does not follow imports from other modules, and it does not track attributes on objects other than `self.x`.
* **Parameters of uncalled functions.** These are not treated as sources unless the function is a Flask-style route (`@app.route` and similar), where they count as URL parameters. Example: `lookup(domain)` in `tests/samples/cwe78_vulnerable.py` uses `shell=True` but is not reported, because nothing in the file calls it with tainted data.
* **Branches and loops.** These are merged conservatively. A sanitizer applied on only one branch leaves the value tainted.
* **Containment checks.** These are recognised only in `if` form: `commonpath(...) != base` / `startswith` / `is_relative_to` on a `realpath`/`abspath`/`resolve` result.
* **CWE-79 scope.** Covers Flask/Django-style code only: `make_response`, `render_template_string`, `Markup`, `HttpResponse`, `mark_safe`, and a route that returns a tainted string containing HTML. Jinja template files are not analysed.
* **CWE-798.** Detection is by name heuristic, entropy and known key formats (AWS, GitHub, Slack, `sk-` and Google keys, private keys). A secret in a variable with a neutral name and an unknown format is missed.
* **`os.environ` and `os.getenv` count as sources**, as requested. Findings from them get lower confidence and one lower severity level.
* **Fix templates are deterministic and cover common shapes.** For a command string that is built somewhere the template cannot see, the CWE-78 fix falls back to `shlex.split()` plus `shell=False`. For a command that uses pipes or redirection, it keeps the shell and applies `shlex.quote()`. If no template fits, the Recommendation tab says so and Apply is disabled. The import a fix needs is added after the last import line.
* **Security test cases.** These run the file in a separate `python -I` process with a timeout. Shell, SQL, file and HTML sinks are replaced by recorders, so no command, query or file read is performed. Other side effects in the code under test, such as network calls, are not blocked. Only events on the finding's own lines are judged. When the payloads cannot exploit the original code in the harness either, the check reports Unavailable instead of Passed.
* **Disagreement between checks.** In `demo_app.py`, the CWE-79 fix escapes `name`, but Semgrep also treats `rows` as tainted because it comes from a query that used `name`. The verdict is therefore Partially resolved. That is a real disagreement between checks, not a crash.
* **Functional tests.** These run pytest (or unittest) on a copy of the project folder when tests exist, and otherwise smoke-import the file. A test suite that already fails on the original code is reported as Unavailable.
* **Terminal tab.** Unchanged. It still offers only `cmd` / PowerShell, as before.
