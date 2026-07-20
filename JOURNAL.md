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
