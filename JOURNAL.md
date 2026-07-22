# Journal — PathReview Contribution

## Week 7: Issue selection

**Issue link:** https://github.com/ascherj/pathreview/issues/150

**Issue title:** Tech detector counts vendored and build-output files, skewing language detection

**Tier:** [x] Tier 1  [ ] Tier 2  [ ] Tier 3

**Problem summary:**
The tech detector in agent/tools/tech_detector.py classifies a repo's primary language by counting files across the whole tree, including node_modules/ and build/ directories. That means a Python-first project that vendors a few JavaScript dependencies ends up flagged as JavaScript, which is wrong. A successful fix excludes vendored and build-output directories from the file count so the primary language matches the actual source code the developer wrote. Two failing tests already exist for this behavior (test_node_modules_excluded and test_build_directory_excluded), and they should pass after the fix.

**"Is this right for me?" checklist reasoning:**
- Scope is small: one file to modify, existing failing tests to make pass, maybe add a few more test cases for related patterns (dist/, vendor/, .min.js). Realistic for a 2-3 week module.
- Domain fits my background: backend Python bug with clear inputs and outputs.
- Reproduction is documented in the issue with exact code. No ambiguity about what "correct" looks like.
- Product angle for portfolio: the fix is really about defining what counts as a project's actual code vs boilerplate, which is a data quality question. Good story for interviews.
- Verified nobody else has an active claim on the issue in the comments.

**Branch name:** fix/150-exclude-vendored-files-from-tech-detector

**Setup confirmation:** [x] App runs locally at localhost:5173

**Cohort ledger:** [ ] Issue added to cohort ledger

## Week 7: Reproduction

**What I reproduced:**
The tech detector reports JavaScript as the primary language for a Python project that vendors a few JS files. It should report Python. Confirmed and reliably reproducible.

**Where the bug lives:**
- agent/tools/tech_detector.py, `_should_skip_file` (around lines 143-164). The skip patterns all start with a slash, like `/node_modules/`. A top level path like `node_modules/x/index.js` has no leading slash, so the pattern is not a substring and the file is not skipped. The vendored JS gets counted.
- There is a second smaller bug. `_detect_tech` picks the primary language with `sorted(languages)[0]` on a set (around lines 126-129). The comment says "most common" but it just takes the alphabetically first language. That is why JavaScript wins over Python once both are in the set.

**How to reproduce:**
1. Run the two existing failing tests:
   `PYTHONIOENCODING=utf-8 .venv/Scripts/python -m pytest tests/unit/test_tech_detector.py -v -m unit`
   test_node_modules_excluded and test_build_directory_excluded both fail. Each expects primary_language Python but gets JavaScript.
2. Or run it directly against the tool:
   `.venv/Scripts/python -c "from agent.tools.tech_detector import TechDetector; print(TechDetector().execute({'files': ['src/main.py','utils.py','node_modules/x/index.js','node_modules/y/lib.js']}).data)"`
   Output shows primary_language JavaScript for a repo that is mostly Python.

**Result:** 2 failed, 25 passed. The two failures are the vendored and build directory cases. Bug is real and I know where it lives.
