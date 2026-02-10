<!-- f650b8da-da7b-4687-9d4b-b4cbd4a674a1 97bc1748-b6b0-4583-87c7-ba864fd411c0 -->
# Avocado Utils Module Migration and Enhancement Plan

## Pre-Migration Questions

**IMPORTANT**: These questions must be asked ONE AT A TIME. Wait for user response after each question before proceeding to the next.

### Question 1: Target Module Identification

**Ask the user**: "Which module do you want to work on?"


**Wait for their answer before proceeding to Question 2.**

---

### Question 2: Migration Strategy Selection

**Ask the user**: "Which migration strategy would you like?"

**Present these options**:
- A) Complete Migration - Enhance in Avocado (tests + docs) THEN migrate to autils
- B) Avocado Enhancement Only - Improve tests and docs, keep in Avocado only
- C) autils Migration Only - Migrate existing module (assumes good coverage already)

**Wait for their answer (A, B, or C) before proceeding to Question 3.**

---

### Question 3: Test Coverage Focus

**Ask the user**: "Which test types should I focus on?"

**Present these options**:
- a) Unit tests primarily (functional tests if time permits)
- b) Both unit and functional tests equally

**Wait for their answer before proceeding to Question 4.**

---

### Question 4: Maintainer Information

**Ask the user**: "Who is submitting/maintaining this module? (This will be used in the metadata file)"

**Present these options**:
- a) Harvey (Harvey James Lynden, hlynden@redhat.com, GitHub: harvey0100)
- b) Jan (Jan Richter, jarichte@redhat.com, GitHub: richtja)
- c) Cleber (Cleber Rosa, crosa@redhat.com, GitHub: clebergnu)
- d) Other (I will need to ask for full name, email, and GitHub username)

**If they select "d) Other"**, ask these follow-up questions one at a time:
1. "What is your full name?"
2. "What is your email address?"
3. "What is your GitHub username?"

**Once all 4 questions are answered, store the information and proceed with the migration workflow.**

## Part 1: Avocado Enhancement (Options A or B)

**Quick Phase Overview:**
| Phase | Focus | Key Deliverable |
|-------|-------|-----------------|
| 1 | Setup & Analysis | Branch created, module analyzed |
| 2 | Unit Tests | Tests passing, no redundancy |
| 3 | Functional Tests | Real-world scenarios tested |
| 4 | Docstrings | Module + function docstrings (pylint-compliant) |
| **4.5** | **Pylint & Deprecation** | **Remove from ignore, add deprecation, add to workflow** |
| 5 | QA & Verification | All static checks passing |

### Phase 1: Setup and Analysis

**1.0 Python Environment Setup**

Ensure Python environment is active with required packages:

```bash
# Activate pyenv (if using pyenv)
eval "$(pyenv init -)"
pyenv shell <version>  # e.g., 3.13, 3.14, etc.

# Or just ensure your desired Python is active
python --version

# Install required packages (if not already installed)
python -m pip install PyYAML pylint==3.0.0 pytest jsonschema black==22.3.0 isort==5.10.1 pyenchant
```

Required packages (with versions from static-checks/requirements.txt):
- `PyYAML` - Schema validation for metadata files
- `pylint==3.0.0` - Code quality and style checking
- `pytest` - Test running and reporting
- `jsonschema` - JSON schema validation
- `black==22.3.0` - Code formatting
- `isort==5.10.1` - Import sorting
- `pyenchant` - Spell checking

**Note**: Use whatever Python version is compatible with Avocado (Python 3.8+ typically)

**1.1 Branch Management**

- Navigate to avocado repository
- Check current branch: `git branch --show-current`
- Create/checkout module-specific branch: `git checkout -b [MODULE_NAME]`

**1.2 Module Analysis**

- Read source module: `avocado/avocado/utils/[MODULE_NAME].py`
- Check for existing unit tests: `avocado/selftests/unit/utils/[MODULE_NAME].py`
- Check for functional tests: `avocado/selftests/functional/utils/[MODULE_NAME].py`
- Identify all public functions using `grep "^def " [MODULE_NAME].py`
- Map untested functions

**1.3 Category Determination for Future Migration**

Analyze module functionality to determine autils category:

- `file` - File system operations, path handling
- `archive` - Archive/compression operations
- `devel` - Development tools, debugging, output formatting
- `network` - Network-related operations

**If module doesn't fit existing categories**: Analyze the module's purpose and create a NEW specific category name (NOT "utils"). Examples:

- String manipulation → `string/` or fit in existing based on primary use
- Data structures → `data/` or determine more specific purpose
- Process management → `process/`
- System operations → `system/`

The category name should be descriptive and meaningful, not generic.

### Phase 2: Unit Test Development

**2.1 Create/Enhance Unit Tests**

If no test file exists, create: `avocado/selftests/unit/utils/[MODULE_NAME].py`

For each function, create comprehensive tests covering:

- **Basic functionality**: Standard use cases
- **Edge cases**: Empty inputs, None values, boundary conditions
- **Type variations**: Different valid input types (str, int, bytes, etc.)
- **Error handling**: Invalid inputs, exception testing
- **Round-trip testing**: For conversion functions
- **Unicode/encoding**: Special characters, various encodings

**Test Quality Over Quantity - IMPORTANT:**

Focus on meaningful coverage and regression detection value, not test count. Based on project maintainer feedback:

- **Avoid redundant tests**: Before creating multiple similar tests, ask "Does this test a different code path in the module?"
- **Example of redundancy**: Testing `wait_for()` with different return types (int, str, list, dict) that are all truthy tests the same truthiness check - use 1-2 representative cases instead of 5+
- **Coverage impact**: Tests should meaningfully increase module coverage, not just exercise the same code paths repeatedly. AI-generated tests added only 0.01% coverage in some cases due to redundancy
- **Regression detection value**: Each test should help pinpoint specific bugs. If 10 tests would all fail together for the same regression, you probably only need 2-3 of them
- **Real-world perspective**: From a method's perspective, testing it with a file existence check vs. an environment variable check vs. a socket check are often the same test (just different predicates) - consolidate these

**Red flags for redundant tests (from actual PR reviews):**

- **Overlapping coverage**: If test B exercises the same code path as test A but with trivially different inputs, consolidate them
  - Example: `test_run_with_environment_variable` and `test_system_output_integration` both test environment variable propagation - combine into one comprehensive test
- **Loose assertions**: Using `assertGreater()` when exact values can be verified suggests the test may be poorly understood
  - Example: Testing large output with `assertGreater(len(result.stdout), 100000)` when you know exactly how many bytes should be produced
  - **Fix**: Use `assertEqual(len(result.stdout), expected_size)` for precise validation
- **"Integration" tests that don't integrate**: Creating a test with "integration" in the name that just repeats a unit test with different data

**Key principle**: A well-designed set of 15 focused tests is more valuable than 30 repetitive tests that add minimal coverage. When in doubt, prioritize:
1. **Coverage**: Does this test execute code paths not covered by existing tests?
2. **Precision**: Does this test use exact assertions that will pinpoint regressions?
3. **Uniqueness**: Is this testing behavior fundamentally different from other tests?

**2.2 Test Execution and Validation - MANDATORY**

**CRITICAL**: You MUST run unit tests to verify they pass.

- **PRIMARY**: Run tests: `python -m avocado run selftests/unit/utils/[MODULE_NAME].py`
- **REQUIRED**: Verify ALL tests pass (check RESULTS line for PASS count)
- Debug failures by testing functions interactively or use pytest for detailed output: `python -m pytest selftests/unit/utils/[MODULE_NAME].py -v`
- Adjust expectations based on actual behavior
- Ensure all tests pass with avocado run

**DO NOT SKIP THIS STEP** - All tests must pass before proceeding.

**2.3 Update Test Count Tracking**

- Count new test methods added
- Update `avocado/selftests/check.py` TEST_SIZE["unit"] value
- Critical: Build system will fail without this update

### Phase 3: Functional Test Development

**3.1 Identify/Create Functional Tests**

- Check if functional tests exist: `avocado/selftests/functional/utils/[MODULE_NAME].py`
- Create functional tests if none exist
- Enhance existing functional tests with:
  - Integration scenarios
  - Real-world usage patterns
  - Cross-module interactions
  - Performance validation

**IMPORTANT - Apply same quality standards as unit tests:**

Functional tests are even more prone to redundancy since they often test similar scenarios with different data. Review the "Test Quality Over Quantity" guidelines in Phase 2.1 before writing functional tests. Specifically:

- **Don't duplicate unit test coverage**: If a unit test already validates environment variable passing, don't create a functional test that does the same thing with a different variable name
- **True integration**: Functional tests should test integration between components, not just "the same test but with real files instead of mocks"
- **Distinct scenarios**: Each functional test should represent a meaningfully different real-world use case
- **Example**: Instead of `test_run_with_environment_variable` (unit) AND `test_system_output_integration` (functional) that both test env vars, pick ONE location and make it comprehensive

**3.2 Functional Test Execution - MANDATORY**

**CRITICAL**: You MUST run functional tests to verify they pass before proceeding.

- **PRIMARY**: Run functional tests with avocado: `python -m avocado run selftests/functional/utils/[MODULE_NAME].py`
- **REQUIRED**: Verify ALL tests pass (check RESULTS line for PASS count)
- Debug failures using pytest if needed: `python -m pytest selftests/functional/utils/[MODULE_NAME].py -v`
- Verify tests pass in realistic scenarios
- Update functional test count in `check.py` if applicable

**DO NOT SKIP THIS STEP** - Functional tests validate real-world integration scenarios.

### Phase 4: RST Docstring Enhancement

**4.1 Add Module Docstring**

**REQUIRED**: Add a module-level docstring at the top of the file (after license comments, before imports):

```python
# ... license comments ...

"""Brief description of the module's purpose."""

import hashlib
import os
```

This is required by pylint (`C0114: Missing module docstring`).

**4.2 Apply autils Docstring Format**

Enhance all public functions with comprehensive RST docstrings following autils format.

**CRITICAL - Pylint Format**: The brief description MUST be on the SAME line as the opening `"""`:

```python
def function_name(param1, param2, param3=None):
    """Brief one-line description ending with period.

    Optional detailed explanation of what the function does, how it works,
    and any important behavior or algorithms. This can span multiple
    paragraphs if needed.

    :param param1: Description of first parameter ending with period.
    :type param1: expected_type
    :param param2: Description of second parameter ending with period.
    :type param2: expected_type
    :param param3: Optional parameter description ending with period.
    :type param3: expected_type or None
    :return: Description of return value ending with period.
    :rtype: return_type
    :raises ValueError: When parameter validation fails.
    :raises TypeError: When parameter types are incorrect.

    :see: http://reference-url-if-applicable

    Example::

        >>> function_name("param1", "param2")
        'expected_output'
        >>> function_name("param1", "param2", "param3")
        'different_output'
    """
```

**⚠️ WRONG FORMAT (will fail pylint C0199):**
```python
def function_name():
    """
    Brief description here.  # <-- WRONG: First line is empty!
    """
```

**✅ CORRECT FORMAT:**
```python
def function_name():
    """Brief description here.  # <-- CORRECT: Content on same line as opening quotes
    
    More details...
    """
```

**4.3 Docstring Standards**

- **Brief description**: On SAME line as `"""`, ending with period, then blank line
- **Detailed description**: Optional, use when function behavior needs explanation
- **Parameters**: Every parameter must have `:param:` and `:type:` directives
- **Return value**: Use `:return:` (singular) and `:rtype:` for non-void functions
- **Exceptions**: Document all exceptions with `:raises ExceptionType:` description
- **References**: Optional `:see:` directive for related documentation/URLs
- **Examples**: Optional `Example::` section for complex functions or non-obvious usage

### Phase 4.5: Pylint & Deprecation Setup (REQUIRED FOR ALL OPTIONS)

**⚠️ DO NOT SKIP THIS PHASE** - These steps are required for Options A, B, AND C.

**4.5.1 Remove from Pylint Ignore List (DO THIS FIRST)**

Remove the module from the pylint ignore list in `.pylintrc_utils`:

1. Open `.pylintrc_utils` in the repository root
2. Find the `ignore=` line (around line 10)
3. Remove `[MODULE_NAME].py,` from the comma-separated list
4. This enables pylint checking for the module

**Why first?** Running static checks after this will reveal any pylint issues in the module that need fixing (like missing module docstring).

**4.5.2 Add Deprecation Notice to Module**

Add the deprecation warning at the END of the module file (after all functions):

```python
# pylint: disable=wrong-import-position
from avocado.utils.deprecation import log_deprecation

log_deprecation.warning("[MODULE_NAME]")
```

**Note**: Use the simple form `log_deprecation.warning("[MODULE_NAME]")` - see `path.py`, `output.py`, `data_structures.py` for examples.

**4.5.3 Add to Migration Announcement Workflow**

Add the module to `.github/workflows/autils_migration_announcement.yml`:

```yaml
paths:
  - '**/ar.py'
  - '**/path.py'
  # ... existing modules ...
  - '**/[MODULE_NAME].py'  # Add this line (alphabetical order preferred)
```

This triggers an automatic comment on PRs that modify the module, informing contributors about the autils migration.

### Phase 5: Quality Assurance

**5.1 Run Module Tests and Static Checks - MANDATORY GATE**

**CRITICAL**: Module tests and static checks MUST PASS before proceeding to Part 2 (autils migration)

- Navigate to avocado repository root

**Run module-specific tests:**
```bash
# Unit tests for the module
python -m avocado run selftests/unit/utils/[MODULE_NAME].py

# Functional tests for the module
python -m avocado run selftests/functional/utils/[MODULE_NAME].py
```

**Run static checks:**
```bash
# Static checks (pylint, black, isort, spell check, etc.)
PYTHONPATH=. python selftests/check.py --select static-checks
```

**Note**: You do NOT need to run all unit/functional tests in the repository - only the tests for the module you're working on. Static checks must still be run as they validate code style across modified files.

**Handling Spelling Errors:**

If the spell checker reports false positives (technical terms, command names, etc.):
- Add the flagged words to `spell.ignore` file in the repository root
- Words should be added one per line at the end of the file
- Common false positives: command names (e.g., `pkill`), technical terms (e.g., `substring`), possessives (e.g., `pkill's`)
- Example: If spell checker flags "pkill" and "subprocess", add them to spell.ignore
- Re-run static checks after updating spell.ignore to verify the errors are resolved

**Optional: Debugging Tools**

If tests fail, use these for detailed output:
- Using pytest for detailed debugging: `python -m pytest selftests/unit/utils/[MODULE_NAME].py -v`
- Auto-format with black: `python -m black avocado/utils/[MODULE_NAME].py selftests/unit/utils/[MODULE_NAME].py selftests/functional/utils/[MODULE_NAME].py`
- **Fix spelling errors**: Add false positives to `spell.ignore` file (one word per line)

**5.2 Part 1 Completion Checklist**

**⚠️ ALL items below are REQUIRED** - even for Option B (Avocado Enhancement Only):

| Step | Task | Command/File |
|------|------|--------------|
| □ | Unit tests passing | `python -m avocado run selftests/unit/utils/[MODULE_NAME].py` |
| □ | Functional tests passing | `python -m avocado run selftests/functional/utils/[MODULE_NAME].py` |
| □ | Test counts updated | `selftests/check.py` TEST_SIZE dictionary |
| □ | Module docstring added | Top of module file (required for pylint) |
| □ | Function docstrings (RST format) | Content on SAME line as `"""` |
| □ | **Removed from pylint ignore** | `.pylintrc_utils` ignore= line |
| □ | **Deprecation notice added** | End of module: `log_deprecation.warning("[MODULE_NAME]")` |
| □ | **Added to workflow** | `.github/workflows/autils_migration_announcement.yml` |
| □ | Static checks passing | `PYTHONPATH=. python selftests/check.py --select static-checks` |
| □ | Branch ready | `git status` shows clean working tree |

**DO NOT SKIP THE BOLDED ITEMS** - They are easy to miss but required for all options.

**For Option B users**: You are DONE after this checklist. No Part 2 needed.
**For Option A users**: Proceed to Part 2 after all items complete.

## Part 2: autils Migration (Options A or C)

**IMPORTANT**: Do not start Part 2 until ALL Phase 5 requirements are met:
- Module unit tests passing
- Module functional tests passing
- Static checks passing (`PYTHONPATH=. python selftests/check.py --select static-checks`)
- Test counts updated in `selftests/check.py` TEST_SIZE dictionary

### Phase 6: Migration Setup

**6.1 Branch Management in autils**

- Navigate to autils repository
- Create/checkout branch: `git checkout -b [MODULE_NAME]`

**6.2 Determine Category**

Based on Phase 1 analysis, place module in:

- Existing category: `archive/`, `devel/`, `file/`, `network/`
- New `utils/` category: For general utilities that don't fit above

### Phase 7: File Migration

**7.1 Create Directory Structure**

```bash
mkdir -p autils/autils/[CATEGORY]
mkdir -p autils/metadata/[CATEGORY]
mkdir -p autils/tests/unit/modules/[CATEGORY]
mkdir -p autils/tests/functional/modules/[CATEGORY]
```

**7.2 Copy Files**

- Copy module: `cp avocado/avocado/utils/[MODULE_NAME].py autils/autils/[CATEGORY]/[MODULE_NAME].py`
- Copy unit tests: `cp avocado/selftests/unit/utils/[MODULE_NAME].py autils/tests/unit/modules/[CATEGORY]/[MODULE_NAME].py`
- Copy functional tests (if they exist): `cp avocado/selftests/functional/utils/[MODULE_NAME].py autils/tests/functional/modules/[CATEGORY]/[MODULE_NAME].py`

**7.3 Update Import Statements**

In both test files (unit and functional), change:

- From: `from avocado.utils import [MODULE_NAME]`
- To: `from autils.[CATEGORY] import [MODULE_NAME]`

Update all test imports throughout both test files.

**IMPORTANT - Import Best Practices:**

When migrating test files, consolidate all imports to the top of the file following PEP 8:

1. **Standard library imports** (e.g., `import os`, `import time`, `import tempfile`, `import threading`, `import socket`)
2. **Third-party imports** (e.g., `from unittest import mock`)
3. **Local/application imports** (e.g., `from autils.[CATEGORY] import [MODULE_NAME]`)

**DO NOT** repeat imports inside individual test methods. If the original Avocado tests have imports scattered throughout functions (like `import tempfile` or `import threading` repeated in multiple test methods), consolidate them at the top of the file during migration.

Example of CORRECT import structure:
```python
import os
import socket
import tempfile
import threading
import time
import unittest
from unittest import mock

from autils.devel import wait


class MyTestClass(unittest.TestCase):
    def test_something(self):
        # Use tempfile, threading, etc. directly - already imported at top
        tmpdir = tempfile.mkdtemp()
        # ... test code ...
```

### Phase 8: Metadata and Documentation

**8.1 Create Metadata File**

Create `autils/metadata/[CATEGORY]/[MODULE_NAME].yml` using the maintainer information provided in the Pre-Migration Questions (Question 4):

```yaml
name: [MODULE_NAME]
description: [Brief description]

categories:
  - [Primary category - Files/Development/OS/Network/Misc]
  - [Secondary category if applicable]
maintainers:
  - name: [Use full name from Question 4]
    email: [Use email from Question 4]
    github_usr_name: [Use GitHub username from Question 4]
supported_platforms:
  - CentOS Stream 9
  - Fedora 36
  - Fedora 37
tests:
  - tests/unit/modules/[CATEGORY]/[MODULE_NAME].py
  - tests/functional/modules/[CATEGORY]/[MODULE_NAME].py  # Include if functional tests exist
remote: false
```

**Maintainer Information Mapping:**
- If **Harvey** selected: name: Harvey James Lynden, email: hlynden@redhat.com, github_usr_name: harvey0100
- If **Jan** selected: name: Jan Richter, email: jarichte@redhat.com, github_usr_name: richtja
- If **Cleber** selected: name: Cleber Rosa, email: crosa@redhat.com, github_usr_name: clebergnu
- If **Other** selected: Use the custom name, email, and GitHub username provided

**8.2 Update Documentation**

Add to `autils/docs/source/utils.rst` in alphabetical order:

```rst
[MODULE_NAME]
===========
.. automodule:: autils.[CATEGORY].[MODULE_NAME]
```

### Phase 9: Validation

**9.1 Schema Validation**

```bash
cd autils && python3 tests/validate_schema.py
```

**9.2 Import Testing**

```bash
python3 -c "import sys; sys.path.insert(0, 'autils'); from autils.[CATEGORY] import [MODULE_NAME]; print('Import successful')"
```

**9.3 Run Migrated Tests**

```bash
# Run unit tests
cd autils && PYTHONPATH=. python -m avocado run tests/unit/modules/[CATEGORY]/[MODULE_NAME].py

# Run functional tests (if they exist)
cd autils && PYTHONPATH=. python -m avocado run tests/functional/modules/[CATEGORY]/[MODULE_NAME].py
```

Note: pytest can be used for debugging:
```bash
cd autils && PYTHONPATH=. python -m pytest tests/unit/modules/[CATEGORY]/[MODULE_NAME].py -v
cd autils && PYTHONPATH=. python -m pytest tests/functional/modules/[CATEGORY]/[MODULE_NAME].py -v
```

**9.4 Basic Functionality Test**

Test key functions work in new location.

### Phase 10: Final Verification

**10.1 Complete Structure Check**

Verify all files exist:

- `autils/autils/[CATEGORY]/[MODULE_NAME].py`
- `autils/metadata/[CATEGORY]/[MODULE_NAME].yml`
- `autils/tests/unit/modules/[CATEGORY]/[MODULE_NAME].py`
- `autils/tests/functional/modules/[CATEGORY]/[MODULE_NAME].py` (if functional tests exist)
- Updated `autils/docs/source/utils.rst`

**10.2 Part 2 Completion Summary**

- Module migrated to autils/[CATEGORY]/[MODULE_NAME]
- All imports updated and working
- Schema validation passed
- All tests passing in new location
- Documentation updated
- Branch ready for review: `autils/[MODULE_NAME]`

## Success Criteria

### Part 1 (Avocado Enhancement) - Required for Options A & B

**Tests:**
- [ ] Unit test coverage >90% (verify with `check_module_coverage.py`)
- [ ] Unit tests EXECUTED and passing
- [ ] Functional tests created/enhanced
- [ ] Functional tests EXECUTED and passing
- [ ] Test counts updated in `selftests/check.py`

**Documentation:**
- [ ] Module docstring at top of file
- [ ] RST docstrings on all functions (content on SAME line as `"""`)

**Pylint & Deprecation (EASY TO MISS):**
- [ ] Module REMOVED from `.pylintrc_utils` ignore list
- [ ] Deprecation notice added at END of module
- [ ] Module added to `.github/workflows/autils_migration_announcement.yml`

**Final Verification:**
- [ ] Static checks passing: `PYTHONPATH=. python selftests/check.py --select static-checks`
- [ ] No functionality regressions

**For Option A**: Proceed to Part 2 after all complete.
**For Option B**: You are DONE after Part 1.

### Part 2 (autils Migration)

- Correct category structure created
- Files properly placed and imports updated
- Schema validation passes
- All unit tests pass
- Documentation updated
- Import path works correctly

## Key Notes

1. **Python Environment**: Use compatible Python version (3.8+) with all required packages installed
2. **Test Runners**: ALWAYS use `avocado run` for primary test execution; pytest only for debugging
3. **Module-Specific Testing**: Run unit/functional tests only for the module being worked on, plus static checks for the whole repo
4. **No Unit Tests?** Create comprehensive unit test suite from scratch
5. **Functional Tests**: Always attempt to improve/create functional tests
6. **Category Selection**: If no existing category fits, create `utils/` subfolder
7. **autils Test Structure**: Unit tests go in `tests/unit/modules/[CATEGORY]/`, functional tests in `tests/functional/modules/[CATEGORY]/`
8. **autils Docstring Format**: Follow examples from existing autils modules
9. **Branch Isolation**: Each repository works on module-specific branch
10. **Gate Between Parts**: Part 1 must be 100% complete (module tests + static checks passing) before starting Part 2

### To-dos

**Part 1: Avocado Enhancement (Options A & B)**

- [ ] Get user confirmation on target module, migration strategy, and test focus
- [ ] Set up git branch in avocado repository: `git checkout -b [MODULE_NAME]`
- [ ] Analyze source module, existing tests, and determine autils category
- [ ] Create or enhance unit tests (focus on unique code paths, avoid redundancy)
- [ ] Create or enhance functional tests (real-world integration scenarios)
- [ ] Add module docstring at top of file (required for pylint)
- [ ] Add RST docstrings to all functions (content on SAME line as `"""`)
- [ ] **Remove module from `.pylintrc_utils` ignore list**
- [ ] **Add deprecation notice at end of module**
- [ ] **Add module to `.github/workflows/autils_migration_announcement.yml`**
- [ ] Update test counts in `selftests/check.py` TEST_SIZE dictionary
- [ ] Run and verify: unit tests, functional tests, static checks all pass

**Part 2: autils Migration (Options A & C only)**

- [ ] Set up git branch in autils repository for migration
- [ ] Create directory structure and copy module files to autils with updated imports
- [ ] Create metadata YAML file and update documentation
- [ ] Validate schema, test imports, run all migrated tests

## Quick Reference Commands

### Environment Setup
```bash
# Activate desired Python version (if using pyenv)
eval "$(pyenv init -)"
pyenv shell <version>  # e.g., 3.13, 3.14, etc.

# Or verify your current Python
python --version

# Ensure required packages are installed (check versions match requirements)
python -m pip list | grep -E "(PyYAML|pylint|pytest|jsonschema|black|isort)"

# Install specific versions if needed
python -m pip install black==22.3.0 isort==5.10.1 pylint==3.0.0
```

### Test Execution (Avocado Repository)
```bash
# PRIMARY: Run module-specific tests with avocado - MANDATORY
python -m avocado run selftests/unit/utils/[MODULE_NAME].py
python -m avocado run selftests/functional/utils/[MODULE_NAME].py

# REQUIRED: Run static checks (pylint, black, isort, spell check)
PYTHONPATH=. python selftests/check.py --select static-checks

# DEBUGGING ONLY: Using pytest for detailed output
python -m pytest selftests/unit/utils/[MODULE_NAME].py -v
python -m pytest selftests/functional/utils/[MODULE_NAME].py -v
```

### Code Quality Checks (Avocado Repository) - MANDATORY
```bash
# REQUIRED: Run static checks
PYTHONPATH=. python selftests/check.py --select static-checks

# Auto-format with black (for quick fixes before running static checks):
python -m black avocado/utils/[MODULE_NAME].py selftests/unit/utils/[MODULE_NAME].py selftests/functional/utils/[MODULE_NAME].py

# Check for linter errors in IDE or use read_lints tool

# If spell checker reports false positives:
# Add flagged words to spell.ignore (one per line at end of file)
# Example: pkill, substring, pkill's
# Then re-run: PYTHONPATH=. python selftests/check.py --select static-checks
```

### Test Execution (autils Repository)
```bash
cd autils

# PRIMARY: Run unit tests with avocado
PYTHONPATH=. python -m avocado run tests/unit/modules/[CATEGORY]/[MODULE_NAME].py

# PRIMARY: Run functional tests with avocado (if they exist)
PYTHONPATH=. python -m avocado run tests/functional/modules/[CATEGORY]/[MODULE_NAME].py

# DEBUGGING ONLY: Using pytest (not primary runner)
PYTHONPATH=. python -m pytest tests/unit/modules/[CATEGORY]/[MODULE_NAME].py -v
PYTHONPATH=. python -m pytest tests/functional/modules/[CATEGORY]/[MODULE_NAME].py -v

# REQUIRED: Run schema validation
python tests/validate_schema.py

# Test import
python -c "import sys; sys.path.insert(0, '.'); from autils.[CATEGORY] import [MODULE_NAME]; print('Import successful')"
```

### Git Commands
```bash
# Check current branch
git branch --show-current

# Create new branch
git checkout -b [MODULE_NAME]

# View changes
git status
git diff

# Stage and commit (when ready)
git add [files]
git commit -m "Add comprehensive tests and docs for [MODULE_NAME] module"
```
