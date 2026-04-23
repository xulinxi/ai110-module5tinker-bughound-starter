# BugHound Mini Model Card (Reflection)

Filled in after running BugHound in both Heuristic and Gemini modes.

---

## 1) What is this system?

**Name:** BugHound

**Purpose:** Analyze a Python snippet, propose a fix, and run reliability checks before suggesting whether the fix should be auto-applied.

**Intended users:** Students learning agentic workflows and AI reliability concepts. Secondary: engineers who want a low-stakes sandbox for studying guardrail design.

---

## 2) How does it work?

BugHound runs a five-step agentic loop implemented in `bughound_agent.BugHoundAgent.run()`:

1. **PLAN** — Log the intent. No model call.
2. **ANALYZE** (`analyze()`) — Produce a list of issues. If an LLM client is available, call it with a JSON-only system prompt. Otherwise, or on any of four fallback conditions (no client, API exception, unparseable JSON, or an empty/content-free issue list after validation), run the deterministic `_heuristic_analyze()` — substring/regex checks for `print(`, bare `except:`, and `TODO`.
3. **ACT** (`propose_fix()`) — If there are no issues, return the original code unchanged. Otherwise, either call the LLM to rewrite the code or run `_heuristic_fix()` (replace `print(` with `logging.info(`, replace bare `except:` with `except Exception as e:`, prepend `import logging`). On LLM exception or empty output, fall back to the heuristic fixer.
4. **TEST** (`reliability.risk_assessor.assess_risk()`) — Always heuristic. Score starts at 100; deducts for issue severity, line-count shrinkage, apparent return-statement removal, bare-except removal. Short-circuits to high risk if the fix is empty or is not valid Python (new guardrail). Decides `level` ∈ {low, medium, high} from thresholds 75/40.
5. **REFLECT** — Log whether the computed `should_autofix` allows automatic application.

The LLM and heuristic pathways are independent at each step: the analyzer may fall back while the fixer proceeds via LLM, or vice versa. The risk assessor is always heuristic and serves as the final gate regardless of which path produced `fixed_code`.

---

## 3) Inputs and outputs

**Inputs tested in this session:**

| Snippet | Shape | Purpose |
|---|---|---|
| `sample_code/cleanish.py` | Short function with `logging` already imported | Baseline — code that should be left alone |
| `sample_code/print_spam.py` | Function with multiple `print` calls | Low-severity lint signal |
| `sample_code/mixed_issues.py` | Function with `TODO`, `print`, bare `except:` | Stacked issues across severities |
| `sample_code/flaky_try_except.py` | File I/O inside bare `try/except` | Single high-severity issue |
| Ad-hoc `# TODO: implement later\ndef noop():\n    return None` | Minimal TODO-only snippet | Isolate the Medium-severity gate |

**Output shape:** `run()` returns `{issues, fixed_code, risk, logs}`.
- `issues`: list of `{type, severity, msg}` dicts.
- `fixed_code`: string; may equal `code_snippet` when no issues were found.
- `risk`: `{score: 0-100, level: low|medium|high, reasons: [str], should_autofix: bool}`.
- `logs`: ordered list of `{step, message}` entries, one per workflow stage plus fallback notes.

**Observed results:**
- Heuristic mode emits from a fixed taxonomy (Code Quality / Reliability / Maintainability) with deterministic severities. Gemini mode emits richer labels like `Error Handling | High` and natural-language messages.
- Risk reports varied widely for the same input depending on mode: `mixed_issues.py` in heuristic mode gave score 35 (3 issues, no structural deductions); in Gemini mode it gave score 10 (2 issues plus structural penalty from the restructured fix).

---

## 4) Reliability and safety rules

Three rules currently applied by `assess_risk`, with the tradeoffs of each:

**a) Severity-based deduction (`risk_assessor.py:36-47`).**
Checks `issue["severity"]` and subtracts 40 / 20 / 5 for High / Medium / Low.
*Why it matters:* caps how confident the agent can be when the issue list itself flags danger.
*False positive:* an LLM that inflates severity for stylistic nits punishes the score for a benign fix.
*False negative:* an LLM that understates severity (calls a real bug "Low") sails through the gate.

**b) `"return"` substring check (`risk_assessor.py:56-58`).**
Penalizes -30 when the word `return` appears in the original but not in the fix.
*Why it matters:* losing a return value is almost always a behavior change.
*False positive:* a removed docstring containing the word "return" triggers the rule even though the actual `return` statement is intact.
*False negative:* a real `return` removed while the word survives in a comment — the rule misses it entirely.

**c) `has_medium_or_high` auto-fix gate (`risk_assessor.py`, added this session).**
`should_autofix` now requires both `level == "low"` **and** no Medium/High severity issues in the list.
*Why it matters:* closes the loophole where a single Medium issue (-20) left the score at 80 — still "low" — and allowed auto-apply without review.
*False positive:* a Medium-severity finding on a clearly correct fix blocks auto-apply when a human would have approved it.
*False negative:* all-Low issue lists can still auto-apply even if there are many of them — the gate doesn't count quantity.

**d) Syntax validity short-circuit (`risk_assessor.py`, added this session).**
Calls `ast.parse(fixed_code)` early; on `SyntaxError`, returns score 0, level high, `should_autofix=False`.
*Why it matters:* previously the risk assessor would happily score invalid Python and potentially recommend auto-apply.
*False positive:* Python 2 code or other dialects would fail `ast.parse` under a Python 3 interpreter — but BugHound targets Python 3 snippets, so this is acceptable.
*False negative:* syntactically valid but semantically broken code (e.g., `return` replaced with `pass`) still passes the gate — only syntax is checked, not semantics.

---

## 5) Observed failure modes

**a) LLM rewrite over-edits and trips substring rules (Gemini mode on `mixed_issues.py`).**
The LLM restructured `compute_ratio` and produced a shorter fix that did not contain the word `return`. The substring-based return check fired (-30), the short-fix rule fired (-20), and combined with severity deductions the score collapsed to 10 — HIGH risk. The rewrite itself may have been semantically acceptable; the assessor was penalizing surface features, not meaning. This is a **false-positive structural penalty** driven by coarse detectors.

**b) Identity "fix" treated as auto-applicable.**
When the analyzer finds no issues, `propose_fix()` short-circuits and returns the original code unchanged. The risk assessor sees `fixed_code == original_code`, score 100, level LOW, `should_autofix=True`. The UI displays "Auto-fix? YES" on a file that will have zero effect if applied. Harmless in outcome but **misleading in labeling** — the agent is promising to apply something, when there is nothing to apply.

**c) Heuristic analyzer false positive inside docstrings.**
`_heuristic_analyze()` uses `"print(" in code` — a raw substring check. A snippet with `print(` inside a docstring example (e.g., `>>> print(x)` in a doctest) gets flagged as a Code Quality issue, and `_heuristic_fix()` happily runs `replace("print(", "logging.info(")` inside the docstring. The heuristic does not distinguish code from literal string content.

---

## 6) Heuristic vs Gemini comparison

On `mixed_issues.py`:

| Aspect | Heuristic | Gemini |
|---|---|---|
| Issue taxonomy | `Code Quality`, `Reliability`, `Maintainability` — fixed | `Code Quality`, `Error Handling` — free-form |
| Message style | Deterministic boilerplate ("Found print statements...") | Context-aware, explains *why* it matters |
| Severity assignments | Hard-coded (TODO → Medium, bare except → High) | Varies per run; inflates TODO to Medium (mentions missing input validation) |
| Number of issues | 3 (print, TODO, bare except) | 2 (grouped the TODO + validation concern; surfaced the bare-except as "catches SystemExit") |
| Proposed fix | Textual substitution: `print(` → `logging.info(`, adds `import logging`, replaces `except:` with `except Exception as e:` | Restructures function, sometimes removes the `return` path, often much shorter |
| Score impact | 100 − 5 − 40 − 20 = 35 (HIGH) | 100 − 20 − 40 − 30 − 20 = 10 (HIGH) — structural rules fired |
| Decision | Auto-fix NO (high) | Auto-fix NO (high) |

**Agreement with intuition:** the risk gate's decision (NO) was right in both modes, but for different reasons. In heuristic mode, severity alone drove the score down. In Gemini mode, the substring/short-fix rules dominated — the gate was right by accident, penalizing an acceptable rewrite. If the LLM had produced a *minimal* fix that preserved line count and `return`, the same issue list would have scored higher and the gate would have rested entirely on the `has_medium_or_high` rule added this session.

---

## 7) Human-in-the-loop decision

**Scenario:** a proposed fix changes the function signature — argument list, number of arguments, return type annotation, or function name. Any signature change is almost always a behavior change that callers need to know about, and the risk assessor currently doesn't notice it.

**Trigger:** after the syntax check passes, extract the `def <name>(<args>)` line(s) from both `original_code` and `fixed_code` (regex `^\s*def\s+\w+\s*\(.*?\)` on each). If the set of signatures differs, force `level = "high"`, append the reason, and set `should_autofix = False`.

**Where it lives:** `risk_assessor.py`, adjacent to the new syntax-validity short-circuit. The assessor is the right place because the signal is path-agnostic — LLM and heuristic fixers should both be gated.

**Message to the user:** *"Proposed fix changes the function signature — callers may break. Review manually before applying."* Shown in the Reasons panel alongside the existing deduction reasons.

---

## 8) Improvement idea

**Minimal-diff policy:** add `difflib.SequenceMatcher(None, original_code, fixed_code).ratio()` to `assess_risk`. If the ratio is below 0.4 (meaning the LLM replaced more than 60% of the code) **and** no High-severity issues were reported, force `level = "medium"` and block auto-fix.

**Why it's small and targeted:** one additional `import difflib` and ~4 lines of logic. It addresses the over-editing failure mode observed in §5(a) — the LLM restructuring `compute_ratio` dropped the diff ratio well below 0.4 even though only a Medium issue was reported. A minimal-diff policy would have caught that specific scenario without depending on the brittle substring rules.

**Why gate on "no High severity":** if the analyzer genuinely found a High-severity bug, a large rewrite may be justified (e.g., replacing a bare `except:` correctly may require restructuring). The gate only fires when the issue list doesn't justify the scope of change.

**Verification:** one offline test using a stub client that returns a radically-different fix for a Low/Medium-severity input; assert `should_autofix is False`.
