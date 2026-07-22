## Solution plan

**Issue:** Tech detector counts vendored and build-output files, skewing language detection — https://github.com/ascherj/pathreview/issues/150

### Understand

The tool should detect what language a project mostly uses by analyzing the codebase and ignoring any borrowed code. Right now it counts the borrowed code too, which causes mislabeling.

Expected: a Python project that vendors a few JS files is reported as Python.
Actual: it is reported as JavaScript.

Root cause is two bugs working together in agent/tools/tech_detector.py:
1. `_should_skip_file` (around lines 143-164) is supposed to drop vendored and build folders like node_modules/, vendor/, dist/, build/. But every skip pattern starts with a slash, like `/node_modules/`. A top level path like `node_modules/x/index.js` has no leading slash, so the pattern is not a substring and the file is never skipped. The borrowed JS gets counted.
2. `_detect_tech` (around lines 126-129) picks the primary language with `sorted(languages)[0]` on a set. The comment says "most common" but it just takes the alphabetically first language. JavaScript sorts before Python, so JavaScript wins whenever both are present.

### Map

Files involved:
- agent/tools/tech_detector.py — the fix. `_should_skip_file` (path matching) and `_detect_tech` (primary language choice).
- tests/unit/test_tech_detector.py — the two failing tests live here (test_node_modules_excluded, test_build_directory_excluded). I will add a few more cases.
- agent/tools/base.py — reference only. Defines BaseTool and ToolResult. No change.
- agent/orchestrator.py (line 105) — reference only. Calls the tool by name. No change.

### Plan

1. Fix the skip logic in `_should_skip_file` so it catches vendored and build folders whether or not the path has a leading slash. Match on path segments instead of a raw substring. For example split the path on `/` and check if any segment is node_modules, vendor, dist, build, .git, __pycache__, .venv, venv. This handles `node_modules/x.js`, `src/node_modules/x.js`, and `/node_modules/x.js` the same way.
2. Fix the primary language choice in `_detect_tech` so it actually reflects the source files. Count how many files map to each language (after filtering), then pick the language with the highest count. Break ties in a stable way so results do not flip randomly.
3. Make the two existing failing tests pass. Run the suite and confirm test_node_modules_excluded and test_build_directory_excluded go green.
4. Add a few more test cases for related patterns the issue mentions: dist/, vendor/ already have one, plus nested vendored dirs and a mixed repo where Python has more real files than JS so primary should be Python by count.
5. Run the full unit test file and the linters (make lint, make format, make typecheck) so nothing else breaks.

### Inputs & outputs

Input: a dict with a `files` key that is a list of file path strings, for example `["src/main.py", "node_modules/x/index.js"]`. Paths can be relative or absolute.

Output: a ToolResult with a data dict containing:
- primary_language: the language of the actual source code, not the vendored code.
- all_languages: sorted list of detected languages, vendored and build files excluded.
- frameworks: sorted list of detected frameworks.

The change should only affect which files get counted and how primary_language is chosen. The shape of the output stays the same.

### Risks & unknowns

- I do not know for sure what format the real caller passes in (leading slash or not, forward or back slashes). The segment split approach should handle both, but I want to confirm how orchestrator.py builds the file list.
- Windows paths can use backslashes. If any caller passes `node_modules\x.js` the split on `/` would miss it. I may normalize separators first.
- Changing primary language from alphabetical to count based could shift results for other repos. Some existing tests may rely on the old behavior. I will read every test before changing this and adjust only what is actually wrong.
- The skip list is hard coded. There may be other vendored folder names in real repos (for example bower_components, target for Rust or Java builds). Out of scope for this issue but worth a note.

### Edge cases

- A folder named node_modules that is nested, like `packages/app/node_modules/x.js`. Should be skipped.
- A file that just contains the word node_modules in its name but is not in that folder, like `src/node_modules_helper.py`. Should NOT be skipped.
- Empty file list or missing files key. Already handled, returns Unknown. Keep it working.
- A repo with only vendored files and no real source. Primary should be Unknown or reflect that there is no real source, not the vendored language.
- Mixed repo where two languages have the same file count. Tie break should be stable and predictable.
- Absolute paths and Windows style backslashes.
