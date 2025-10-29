# Specification Lifecycle Business Rules

## Business Context

The specification lifecycle governs how features progress from idea to implementation through structured documentation phases. It ensures consistent identification, organization, and state tracking of all specifications across the project.

**Domain**: Software Development Workflow
**Purpose**: Manage specification identity, directory structure, and document state transitions
**Actors**: Developers, Project Managers, Automated Tooling

---

## Core Concepts

### Specification Identity

A specification is uniquely identified by:
- **ID**: 3-digit zero-padded number (001, 002, 042)
- **Name**: URL-friendly sanitized feature name (lowercase, hyphens, alphanumeric only)
- **Directory**: `docs/specs/{ID}-{name}/`

**Constraints**:
- IDs are sequential but gap-tolerant (001, 002, 005 is valid)
- IDs never reused, even if specification deleted
- Names must match regex pattern: `^[a-z0-9]+(-[a-z0-9]+)*$`
- Directory name format: `^\d{3}-[a-z0-9]+(-[a-z0-9]+)*$`

### Document Types

Each specification directory contains:
- **product-requirements.md** (PRD): Business requirements and user needs
- **solution-design.md** (SDD): Technical architecture and design decisions
- **implementation-plan.md** (PLAN): Execution tasks and validation steps

Optional quality gates:
- **definition-of-ready.md**: Prerequisites before document creation
- **definition-of-done.md**: Completion criteria after document creation
- **task-definition-of-done.md**: Implementation task validation

---

## Business Rules

### BR-SPEC-001: Specification ID Assignment

**Rule**: Specification IDs are 3-digit zero-padded sequential numbers assigned from the highest existing ID + 1

**Given** existing specifications with IDs: 001, 002, 004
**When** creating a new specification
**Then** assign ID: 005

**Given** no existing specifications
**When** creating the first specification
**Then** assign ID: 001

**Constraint**: Gaps in sequence are allowed and intentional (deleted specs leave gaps)
**Validation**: ID must match regex pattern `^\d{3}$`
**Implementation**: Scan `docs/specs/` directory, extract numbers from `^\d{3}-` pattern, return `max(numbers) + 1`

**Examples**:
- Existing: [] → New: 001
- Existing: [001] → New: 002
- Existing: [001, 002, 003] → New: 004
- Existing: [001, 005, 010] → New: 011 (gap-tolerant)

---

### BR-SPEC-002: Feature Name Sanitization

**Rule**: Feature names are transformed to URL-friendly directory names through lowercase conversion and special character replacement

**Given** feature name: "Multi-Tenancy & RBAC"
**When** sanitizing for directory name
**Then** produce: "multi-tenancy-rbac"

**Given** feature name: "User Auth & API!!"
**When** sanitizing for directory name
**Then** produce: "user-auth-api"

**Given** feature name: "Search___Optimization"
**When** sanitizing for directory name
**Then** produce: "search-optimization"

**Transformation Rules**:
1. Convert to lowercase: `name.lower()`
2. Replace non-alphanumeric sequences with single hyphen: `re.sub(r'[^a-z0-9]+', '-', name)`
3. Strip leading/trailing hyphens: `name.strip('-')`

**Constraint**: Result must be non-empty after sanitization
**Validation**: Result must match `^[a-z0-9]+(-[a-z0-9]+)*$`

**Edge Cases**:
- "---Special---" → "special"
- "123-Numbers" → "123-numbers"
- "CamelCase" → "camelcase"
- "   Spaces   " → "spaces"

---

### BR-SPEC-003: Directory Creation and Template Population

**Rule**: Specification directories are created with ID-prefixed names and optionally populated with document templates

**Given** spec ID: 001, sanitized name: "user-authentication"
**When** creating specification directory
**Then** create: `docs/specs/001-user-authentication/`

**Given** specification directory exists
**When** template requested via `--add product-requirements`
**Then** copy `plugins/start/templates/product-requirements.md` to `docs/specs/{ID}-{name}/product-requirements.md`

**Given** template requested but file does not exist in `plugins/start/templates/`
**When** attempting template copy
**Then** fail with error: "Template {template}.md not found"

**Constraint**: Parent directory `docs/specs/` must exist or be created
**Validation**: Directory creation must succeed before template copy

**Template Discovery**:
- Templates are read from plugin directory: `plugins/start/templates/`
- Available templates: `product-requirements.md`, `solution-design.md`, `implementation-plan.md`, `definition-of-ready.md`, `definition-of-done.md`, `task-definition-of-done.md`
- Templates are copied verbatim, preserving all `[NEEDS CLARIFICATION]` placeholders

---

### BR-SPEC-004: Specification Continuation

**Rule**: Existing specifications can be continued by adding new templates without creating new directories

**Given** existing spec ID: 042, directory: `docs/specs/042-multi-tenancy/`
**When** running `spec.py 042 --add solution-design`
**Then** add `solution-design.md` to existing directory `docs/specs/042-multi-tenancy/`

**Given** spec ID: 042 does not exist
**When** running `spec.py 042 --add solution-design`
**Then** fail with error: "Spec 042 not found"

**Constraint**: Continuation requires exact 3-digit ID match (`^\d{3}$`)
**Validation**: Directory must exist with pattern `{ID}-*` before template addition

---

### BR-SPEC-005: Specification Metadata Reading (TOML Output)

**Rule**: Specification metadata is output in TOML format for tool consumption

**Given** spec ID: 001, directory: `docs/specs/001-user-authentication/`
**When** running `spec.py 001 --read`
**Then** output TOML:
```toml
id = "001"
name = "user-authentication"
dir = "docs/specs/001-user-authentication"

[spec]
prd = "docs/specs/001-user-authentication/product-requirements.md"
sdd = "docs/specs/001-user-authentication/solution-design.md"
plan = "docs/specs/001-user-authentication/implementation-plan.md"

[gates]
definition_of_ready = "docs/specs/001-user-authentication/definition-of-ready.md"
definition_of_done = "docs/specs/001-user-authentication/definition-of-done.md"
task_definition_of_done = "docs/specs/001-user-authentication/task-definition-of-done.md"

files = [
  "definition-of-done.md",
  "definition-of-ready.md",
  "implementation-plan.md",
  "product-requirements.md",
  "solution-design.md",
  "task-definition-of-done.md"
]
```

**TOML Structure**:
- **Root level**: `id`, `name`, `dir` (always present)
- **[spec] section**: Only includes existing document files (prd, sdd, plan)
- **[gates] section**: Only present if at least one quality gate file exists
- **files array**: Sorted list of all files in directory

**Constraint**: Missing files are omitted from output (not listed as null/empty)

---

## Specification State Machine

### States

A specification progresses through these states based on document existence and completeness:

1. **Created**: Directory exists, no documents
2. **PRD In Progress**: PRD exists with `[NEEDS CLARIFICATION]` markers
3. **PRD Complete**: PRD exists, no placeholders remaining
4. **SDD In Progress**: SDD exists with `[NEEDS CLARIFICATION]` markers
5. **SDD Complete**: SDD exists, no placeholders remaining
6. **PLAN In Progress**: PLAN exists with `[NEEDS CLARIFICATION]` markers
7. **PLAN Complete**: PLAN exists, no placeholders remaining
8. **Implemented**: PLAN complete and implementation tasks finished

### State Transitions

```
Created
  ↓ (add PRD template)
PRD In Progress
  ↓ (complete all PRD sections)
PRD Complete
  ↓ (add SDD template)
SDD In Progress
  ↓ (complete all SDD sections)
SDD Complete
  ↓ (add PLAN template)
PLAN In Progress
  ↓ (complete all PLAN sections)
PLAN Complete
  ↓ (execute implementation)
Implemented
```

### State Detection Logic

**Given** specification directory exists
**When** determining current state
**Then** apply these checks in order:

1. **No documents exist** → State: Created
2. **PRD exists + has `[NEEDS CLARIFICATION]` markers** → State: PRD In Progress
3. **PRD exists + no placeholders + SDD missing** → State: PRD Complete
4. **SDD exists + has `[NEEDS CLARIFICATION]` markers** → State: SDD In Progress
5. **SDD exists + no placeholders + PLAN missing** → State: SDD Complete
6. **PLAN exists + has `[NEEDS CLARIFICATION]` markers** → State: PLAN In Progress
7. **PLAN exists + no placeholders** → State: PLAN Complete

**Detection Commands**:
```bash
# Check if PRD complete
test -f docs/specs/{ID}/product-requirements.md && \
  grep -c "\[NEEDS CLARIFICATION" docs/specs/{ID}/product-requirements.md
# Exit code 0 and count = 0 means complete

# Check if SDD complete
test -f docs/specs/{ID}/solution-design.md && \
  grep -c "\[NEEDS CLARIFICATION" docs/specs/{ID}/solution-design.md
# Exit code 0 and count = 0 means complete

# Check if PLAN complete
test -f docs/specs/{ID}/implementation-plan.md && \
  grep -c "\[NEEDS CLARIFICATION" docs/specs/{ID}/implementation-plan.md
# Exit code 0 and count = 0 means complete
```

---

## Workflows

### Workflow 1: Create New Specification

**Actor**: Developer
**Trigger**: New feature requirement identified

**Steps**:
1. Developer runs: `spec.py "Multi-Tenancy & RBAC"`
2. System extracts highest ID from `docs/specs/`: max([001, 002, 004]) = 004
3. System calculates next ID: 005
4. System sanitizes name: "multi-tenancy-rbac"
5. System creates directory: `docs/specs/005-multi-tenancy-rbac/`
6. System outputs confirmation:
   ```
   Created spec directory: docs/specs/005-multi-tenancy-rbac
   Spec ID: 005
   Specification directory created successfully
   ```

**Postcondition**: Specification in "Created" state

---

### Workflow 2: Add Template to New Specification

**Actor**: Developer
**Trigger**: Specification directory created, need to add PRD

**Steps**:
1. Developer runs: `spec.py "User Auth" --add product-requirements`
2. System creates directory: `docs/specs/006-user-auth/`
3. System copies template: `plugins/start/templates/product-requirements.md` → `docs/specs/006-user-auth/product-requirements.md`
4. System outputs:
   ```
   Created spec directory: docs/specs/006-user-auth
   Spec ID: 006
   Generated template: product-requirements.md
   Specification directory created successfully
   ```

**Postcondition**: Specification in "PRD In Progress" state

---

### Workflow 3: Continue Existing Specification

**Actor**: Developer
**Trigger**: PRD completed, need to add SDD

**Steps**:
1. Developer runs: `spec.py 005 --add solution-design`
2. System finds directory matching `005-*`: `docs/specs/005-multi-tenancy-rbac/`
3. System copies template: `plugins/start/templates/solution-design.md` → `docs/specs/005-multi-tenancy-rbac/solution-design.md`
4. System outputs:
   ```
   Adding template to existing spec: docs/specs/005-multi-tenancy-rbac
   Generated template: solution-design.md
   ```

**Postcondition**: Specification transitions from "PRD Complete" to "SDD In Progress" state

---

### Workflow 4: Read Specification Metadata

**Actor**: Automated tooling (e.g., /start:specify command)
**Trigger**: Need to determine specification state and files

**Steps**:
1. Tool runs: `spec.py 005 --read`
2. System finds directory: `docs/specs/005-multi-tenancy-rbac/`
3. System checks for document files:
   - PRD: exists
   - SDD: exists
   - PLAN: missing
4. System checks for quality gate files:
   - DOR: exists
   - DOD: exists
   - TASK-DOD: missing
5. System outputs TOML with existing files only
6. Tool parses TOML to determine state: "SDD Complete" (SDD exists, PLAN missing)
7. Tool suggests continuation: "SDD found. Continue to Step 4 (Implementation Plan)?"

**Postcondition**: Tool has accurate specification state

---

## Edge Cases

### Edge Case 1: Empty Directory Name After Sanitization

**Given** feature name: "!@#$%^&*()"
**When** sanitizing name
**Then** result after sanitization: ""
**Expected**: System should reject with error: "Feature name must contain alphanumeric characters"

**Mitigation**: Validate sanitized name is non-empty before directory creation

---

### Edge Case 2: Directory Already Exists

**Given** directory exists: `docs/specs/003-search-optimization/`
**When** running: `spec.py "Search Optimization"`
**Then** directory creation with `mkdir(exist_ok=True)` succeeds
**Expected**: System should not fail, but may overwrite if template added

**Mitigation**: Check for existing directory and warn before overwriting

---

### Edge Case 3: Non-Sequential ID Request

**Given** existing specs: [001, 002, 003]
**When** running: `spec.py 010 --add product-requirements`
**Then** system interprets "010" as existing ID to continue
**Expected**: If directory doesn't exist, fail with "Spec 010 not found"

**Behavior**: Exact 3-digit IDs are treated as continuation requests, not new creation

---

### Edge Case 4: Template Missing from Plugin

**Given** template requested: `--add custom-template`
**When** template file not found: `plugins/start/templates/custom-template.md`
**Then** system outputs: "Warning: Template custom-template.md not found"
**Expected**: Directory created but template not copied

**Current Behavior**: Warning issued, continues execution
**Alternative**: Could fail fast with error exit code

---

### Edge Case 5: Large ID Numbers

**Given** existing specs with ID: 998
**When** creating new specification
**Then** next ID: 999
**Expected**: System continues normally

**Given** existing specs with ID: 999
**When** creating new specification
**Then** next ID would be: 1000 (4 digits)
**Expected**: Breaks 3-digit constraint

**Constraint Violation**: Maximum 999 specifications per project
**Mitigation**: Document limitation or extend to support 4+ digit IDs

---

## Validation Rules

### VR-SPEC-001: ID Format Validation

**Rule**: Specification IDs must be exactly 3 digits, zero-padded

**Valid**: 001, 042, 100, 999
**Invalid**: 1, 42, 0001, 1000, abc, 01a

**Regex**: `^\d{3}$`

---

### VR-SPEC-002: Directory Name Format Validation

**Rule**: Directory names must follow pattern: `{3-digit-ID}-{sanitized-name}`

**Valid**:
- `001-user-authentication`
- `042-multi-tenancy-rbac`
- `100-search-optimization`

**Invalid**:
- `1-user-auth` (ID not zero-padded)
- `001-User_Auth` (uppercase, underscore)
- `001-` (missing name)
- `-user-auth` (missing ID)

**Regex**: `^\d{3}-[a-z0-9]+(-[a-z0-9]+)*$`

---

### VR-SPEC-003: Template File Existence

**Rule**: Template files must exist in plugin templates directory before copying

**Check**: `(plugins/start/templates/{template}.md).exists()`
**Action on failure**: Warn user, skip template copy

---

### VR-SPEC-004: Document Completeness Detection

**Rule**: Documents are complete when they exist and contain no `[NEEDS CLARIFICATION]` markers

**Check**:
```bash
test -f {document} && grep -c "\[NEEDS CLARIFICATION" {document}
```
**Complete when**: File exists AND grep count = 0

---

## Implementation Notes

### Technology Stack
- **Language**: Python 3
- **Location**: `plugins/start/scripts/spec.py`
- **Working Directory**: Current directory where command executed
- **Template Source**: `plugins/start/templates/` (relative to script location)
- **Specification Target**: `docs/specs/` (relative to current directory)

### Key Implementation Details

**ID Calculation**:
```python
def get_next_spec_id() -> str:
    max_id = 0
    if SPECS_DIR.exists():
        for dir_path in SPECS_DIR.iterdir():
            if dir_path.is_dir():
                match = re.match(r'^(\d{3})-', dir_path.name)
                if match:
                    num = int(match.group(1))
                    if num > max_id:
                        max_id = num
    return f"{max_id + 1:03d}"
```

**Name Sanitization**:
```python
def sanitize_name(name: str) -> str:
    name = name.lower()
    name = re.sub(r'[^a-z0-9]+', '-', name)
    name = name.strip('-')
    return name
```

**State Detection** (conceptual, not implemented in script):
```python
def detect_state(spec_dir: Path) -> str:
    prd = spec_dir / "product-requirements.md"
    sdd = spec_dir / "solution-design.md"
    plan = spec_dir / "implementation-plan.md"

    if not any([prd.exists(), sdd.exists(), plan.exists()]):
        return "Created"

    if prd.exists():
        if has_placeholders(prd):
            return "PRD In Progress"
        elif not sdd.exists():
            return "PRD Complete"

    if sdd.exists():
        if has_placeholders(sdd):
            return "SDD In Progress"
        elif not plan.exists():
            return "SDD Complete"

    if plan.exists():
        if has_placeholders(plan):
            return "PLAN In Progress"
        else:
            return "PLAN Complete"

    return "Unknown"

def has_placeholders(file: Path) -> bool:
    content = file.read_text()
    return "[NEEDS CLARIFICATION" in content
```

---

## Configuration

### Directory Structure
```
project-root/
├── docs/
│   └── specs/
│       ├── 001-user-authentication/
│       │   ├── product-requirements.md
│       │   ├── solution-design.md
│       │   └── implementation-plan.md
│       ├── 002-multi-tenancy/
│       │   └── product-requirements.md
│       └── 005-search-optimization/
│           ├── product-requirements.md
│           ├── solution-design.md
│           ├── implementation-plan.md
│           └── definition-of-done.md
└── plugins/
    └── start/
        ├── scripts/
        │   └── spec.py
        └── templates/
            ├── product-requirements.md
            ├── solution-design.md
            ├── implementation-plan.md
            ├── definition-of-ready.md
            ├── definition-of-done.md
            └── task-definition-of-done.md
```

### Path Resolution
- **Script Location**: `plugins/start/scripts/spec.py`
- **Plugin Root**: `plugins/start/` (calculated as `script_dir.parent`)
- **Template Source**: `{plugin_root}/templates/`
- **Specification Target**: `docs/specs/` (relative to current working directory)

---

## FAQ

**Q: What happens if I delete specification 003?**
A: The gap remains. Next specification will still be 006 if existing specs are [001, 002, 004, 005]. IDs are never reused.

**Q: Can I rename a specification directory?**
A: Manual renaming is not recommended as tooling expects the `{ID}-{name}` pattern. Change the name portion only, preserve the ID prefix.

**Q: How do I know which document to work on next?**
A: Run `spec.py {ID} --read` and check the `[spec]` section. Missing documents or documents with `[NEEDS CLARIFICATION]` markers need attention.

**Q: Can I skip PRD and go directly to SDD?**
A: Technically yes (just run `spec.py {ID} --add solution-design`), but workflow best practices recommend completing PRD before SDD.

**Q: What if I have more than 999 specifications?**
A: Current system supports maximum 999 specs (001-999). Extension to 4-digit IDs would require code modification and migration.

**Q: Can I customize templates?**
A: Yes, edit files in `plugins/start/templates/`. Changes apply to all new specifications created after modification.
