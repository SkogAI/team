# Quality Gates Business Rules

## Business Context

Quality gates enforce readiness and completeness criteria at critical decision points in the specification and implementation lifecycle. They prevent premature progression, ensure minimum information thresholds are met, and validate that deliverables meet defined standards before moving forward.

**Domain**: Quality Assurance and Workflow Governance
**Purpose**: Enforce prerequisites (DOR), validate completeness (DOD), verify implementation quality (TASK-DOD)
**Actors**: Developers, Automated Validation Systems, Project Leads

---

## Core Concepts

### Quality Gate Types

The system defines **3 primary quality gate types**:

1. **Definition of Ready (DOR)**: Prerequisites that MUST be met BEFORE creating documents
   - Validates: Sufficient information exists to start work
   - Enforced: Before PRD, SDD, or PLAN creation
   - Purpose: Prevent starting too early, ensure research complete

2. **Definition of Done (DOD)**: Completion criteria that MUST be met AFTER creating documents
   - Validates: Document completeness, logical flow, coverage, consistency
   - Enforced: After PRD, SDD, or PLAN completion
   - Purpose: Ensure quality standards met before progression

3. **Task Definition of Done (TASK-DOD)**: Implementation quality criteria for individual tasks
   - Validates: Build success, test coverage, code quality, TDD adherence
   - Enforced: After each implementation task in PLAN
   - Purpose: Ensure code meets standards before marking task complete

---

### Validation Levels

Quality gates operate at **3 enforcement levels**:

#### Strict
- **Behavior**: Block on ANY failure (critical or non-critical)
- **When to Use**: Critical systems, production releases, high-risk features
- **Threshold**: 100% of all checks must pass
- **Example**: Financial transactions, security features, data migrations

#### Balanced (Default)
- **Behavior**: Block on critical failures, warn on non-critical
- **When to Use**: Standard development, most features
- **Threshold**: Critical items 100%, overall items ≥85%
- **Example**: Regular feature development, standard workflows

#### Advisory
- **Behavior**: Never block, only provide warnings
- **When to Use**: Exploration, prototypes, emergency fixes
- **Threshold**: No enforcement, informational only
- **Example**: Proof-of-concepts, research spikes, hotfixes

**Configuration**:
```yaml
validation_level: balanced  # strict | balanced | advisory
```

---

### Check Types

Quality gates consist of **2 check categories**:

#### Automated Checks
- **Execution**: Run programmatically via shell commands or code analysis
- **Speed**: Fast (seconds)
- **Objectivity**: Deterministic, repeatable results
- **Examples**: File existence, grep for patterns, command exit codes, metric calculations

#### Manual Verification
- **Execution**: Developer judgment after automated checks pass
- **Speed**: Slower (requires human review)
- **Subjectivity**: Context-aware, experience-based
- **Examples**: "Is the problem statement clear?", "Would a new team member understand this?"

**Rule**: Automated checks run first. Manual verification only presented if automated checks pass.

---

### Threshold Model

Quality gates enforce **dual threshold scoring**:

#### Critical Threshold
- **Value**: Always 100%
- **Items**: Must-have prerequisites or requirements
- **Failure Impact**: Blocks progression (cannot proceed)
- **Examples**: "PRD file exists", "Build succeeds", "All tests pass"

#### Overall Threshold
- **Value**: Configurable (default 85%, range 70-95%)
- **Items**: All checks combined (critical + non-critical)
- **Failure Impact**: Blocks if below threshold (balanced/strict modes)
- **Examples**: Coverage target, documentation completeness, research depth

**Scoring Formula**:
```python
critical_score = (critical_passed / critical_total) * 100
overall_score = (total_passed / total_checks) * 100

if critical_score < 100:
    BLOCK("Critical prerequisites incomplete")
elif overall_score < threshold:
    BLOCK("Overall readiness below threshold")
else:
    PROCEED()
```

---

## Business Rules

### BR-QG-001: Definition of Ready Enforcement

**Rule**: DOR validation MUST be performed and passed before creating PRD, SDD, or PLAN documents

**Given** developer wants to create SDD
**When** DOR validation runs for "Before Creating SDD"
**Then** execute automated checks:
1. PRD file exists
2. PRD has no `[NEEDS CLARIFICATION]` markers
3. All PRD sections non-empty
4. PRD validation checklist complete
5. Architecture approach documented
6. Technical dependencies identified

**Given** critical score < 100%
**When** calculating enforcement decision
**Then** BLOCK with failure message listing incomplete critical items

**Given** critical score = 100%, overall score = 75%, threshold = 85%
**When** calculating enforcement decision
**Then** BLOCK with failure message listing non-critical items below threshold

**Given** critical score = 100%, overall score = 90%, threshold = 85%
**When** calculating enforcement decision
**Then** PROCEED to manual verification

**Failure Message Format**:
```
❌ Definition of Ready: BLOCKED

Document: SDD
Overall: 5/7 checks passed (71%)
Critical: 3/4 checks passed (75%)

⛔ Critical Failures:
  • PRD has 3 [NEEDS CLARIFICATION] markers (lines 45, 67, 89)
    Impact: Cannot write SDD without complete PRD
    Fix: Complete PRD sections at lines 45, 67, 89

⚠️ Non-Critical Failures:
  • Competitive analysis not found
    Impact: May miss important context
    Fix: Create docs/research/competitors.md

🔧 Next Actions:
  1. Edit docs/specs/001/product-requirements.md lines 45, 67, 89
  2. Remove [NEEDS CLARIFICATION] markers
  3. (Optional) Create competitor analysis
  4. Re-run: /start:specify [continue]

Cannot proceed until critical items resolved.
```

---

### BR-QG-002: Definition of Done Validation

**Rule**: DOD validation MUST be performed and passed after completing PRD, SDD, or PLAN documents

**Given** developer completes PRD
**When** DOD validation runs for "PRD Completion"
**Then** execute automated checks:
1. All required sections present (compare headers to template)
2. No `[NEEDS CLARIFICATION]` markers remaining
3. Validation checklist complete (all items `[x]`)
4. Minimum word count met (≥500 words for PRD)
5. All MoSCoW categories addressed

**Additional validations** (if enabled):
- **SCQA Validation**: Situation-Complication-Question-Answer logical flow
- **MECE Validation**: Mutually Exclusive, Collectively Exhaustive coverage
- **Consistency Validation**: Cross-section and cross-document alignment

**Given** validation passes automated checks + manual verification
**When** marking document complete
**Then** allow progression to next document (PRD → SDD → PLAN)

**Given** validation fails
**When** developer attempts to proceed
**Then** BLOCK with specific failures and remediation guidance

---

### BR-QG-003: SCQA Validation (Logical Flow)

**Rule**: Documents must follow Situation-Complication-Question-Answer logical flow to ensure coherent narrative

**SCQA Framework**:
1. **Situation**: Context and background (what is the current state?)
2. **Complication**: Problem statement (what's wrong with the current state?)
3. **Question**: Implied question (what should we do about it?)
4. **Answer**: Solution or approach (how will we address the problem?)

**PRD SCQA Validation**:
```python
# Automated checks
sections = extract_sections(prd)
required_order = ["Context/Background", "Problem", "Solution"]
validate_order(sections, required_order)

# Check section content length
assert len(sections["Context"]) >= 100  # words
assert len(sections["Problem"]) > 0
assert len(sections["Solution"]) > 0

# Manual verification
- Does Context flow naturally into Problem?
- Does Problem make reader ask "What should we do?"
- Does Solution answer that question?
- Is there clear cause-and-effect chain?
```

**SDD SCQA Validation**:
```python
# Automated checks
assert "Constraints" or "Requirements" section exists
assert "Technical Challenges" section exists
assert "Architecture Decisions" section exists

# Verify decisions have rationale
decisions = extract_decisions(sdd)
for decision in decisions:
    if not has_rationale(decision):
        FAIL(f"Decision '{decision}' missing rationale")

# Manual verification
- Is it clear WHY each architecture decision was made?
- Are trade-offs explained (what was considered and rejected)?
```

**Failure Example**:
```
⚠️ SCQA Failures:
  • Architecture decision "Use PostgreSQL" missing rationale
    Impact: Team may not understand why this choice was made
    Fix: Add rationale to SDD Section 5.2
```

---

### BR-QG-004: MECE Validation (Coverage Completeness)

**Rule**: Documents must be Mutually Exclusive (no overlap) and Collectively Exhaustive (nothing missing)

**Mutually Exclusive Checks** (no duplication/overlap):

**PRD MECE**:
```python
# Check for duplicate user stories
stories = extract_user_stories(prd)
for i, story1 in enumerate(stories):
    for story2 in stories[i+1:]:
        similarity = fuzzy_match(story1, story2)
        if similarity > 0.80:
            FAIL(f"Stories '{story1}' and '{story2}' are {similarity*100}% similar")

# Check for contradicting requirements
requirements = parse_requirements(prd)
for req1 in requirements:
    for req2 in requirements:
        if contradicts(req1, req2):
            FAIL(f"Requirement '{req1}' contradicts '{req2}'")

# Check feature redundancy
features = extract_features(prd)
for i, feat1 in enumerate(features):
    for feat2 in features[i+1:]:
        tfidf_similarity = calculate_tfidf(feat1, feat2)
        if tfidf_similarity > 0.70:
            FAIL(f"Features '{feat1}' and '{feat2}' may be redundant")
```

**SDD MECE**:
```python
# Check component responsibility overlap
components = extract_components(sdd)
for i, comp1 in enumerate(components):
    for comp2 in components[i+1:]:
        similarity = calculate_similarity(comp1.description, comp2.description)
        if similarity > 0.70:
            FAIL(f"Components '{comp1.name}' and '{comp2.name}' have overlapping responsibilities")

# Check for duplicate APIs
endpoints = extract_endpoints(sdd)
for endpoint in endpoints:
    if endpoints.count(endpoint) > 1:
        FAIL(f"Endpoint '{endpoint}' defined multiple times")
```

**Collectively Exhaustive Checks** (completeness):

**PRD MECE**:
```python
# Every persona must have at least one user journey
personas = extract_personas(prd)
journeys = extract_journeys(prd)
for persona in personas:
    if not has_journey(persona, journeys):
        FAIL(f"Persona '{persona}' has no user journey")

# Every feature must have acceptance criteria
features = extract_features(prd)
for feature in features:
    if not has_acceptance_criteria(feature):
        FAIL(f"Feature '{feature}' missing acceptance criteria")

# Every metric must have tracking method
metrics = extract_metrics(prd)
for metric in metrics:
    if not has_tracking_method(metric):
        FAIL(f"Metric '{metric}' has no tracking method")
```

**SDD MECE**:
```python
# Every PRD requirement must be addressed in SDD
prd_requirements = extract_requirements("docs/specs/ID/product-requirements.md")
sdd_coverage = extract_requirement_mappings("docs/specs/ID/solution-design.md")

for req in prd_requirements:
    if req not in sdd_coverage:
        FAIL(f"PRD requirement '{req}' not addressed in SDD")

# All interfaces between components must be defined
components = extract_components(sdd)
for comp1 in components:
    for comp2 in components:
        if comp1.depends_on(comp2):
            interface = find_interface(comp1, comp2, sdd)
            if not interface:
                FAIL(f"Interface between '{comp1}' and '{comp2}' not defined")
```

**Failure Example**:
```
⚠️ MECE Failures:
  • Components "AuthService" and "UserService" have 78% description overlap
    Impact: Unclear responsibility boundaries
    Fix: Clarify distinct responsibilities or merge components

  • PRD requirement "User authentication" not covered in SDD
    Impact: Feature will be missing from implementation
    Fix: Add authentication component to SDD Section 4
```

---

### BR-QG-005: Consistency Validation (Cross-Document)

**Rule**: Documents must be internally consistent (within document) and externally consistent (across documents)

**Internal Consistency** (PRD):
```python
# Personas mentioned in journeys must be defined
journeys = extract_journeys(prd)
personas = extract_personas(prd)
for journey in journeys:
    persona_refs = extract_persona_references(journey)
    for persona_ref in persona_refs:
        if persona_ref not in personas:
            FAIL(f"Journey references undefined persona '{persona_ref}'")

# Metrics must align with goals
goals = extract_goals(prd)
metrics = extract_metrics(prd)
for goal in goals:
    if not has_corresponding_metric(goal, metrics):
        FAIL(f"Goal '{goal}' has no corresponding metric")

# Features must support stated objectives
features = extract_features(prd)
objectives = extract_objectives(prd)
for feature in features:
    if not references_objective(feature, objectives):
        FAIL(f"Feature '{feature}' doesn't reference any objective")
```

**Cross-Document Consistency** (SDD ↔ PRD):
```python
# Every PRD feature must map to SDD component
prd_features = extract_features("docs/specs/ID/product-requirements.md")
sdd_components = extract_components("docs/specs/ID/solution-design.md")

for feature in prd_features:
    if not has_component_mapping(feature, sdd_components):
        FAIL(f"PRD feature '{feature}' not implemented in SDD")

# Technical constraints in SDD must align with PRD assumptions
prd_constraints = extract_constraints(prd)
sdd_constraints = extract_constraints(sdd)
for prd_constraint in prd_constraints:
    if contradicts(prd_constraint, sdd_constraints):
        FAIL(f"SDD constraint contradicts PRD assumption: '{prd_constraint}'")

# Success metrics from PRD must have implementation plan in SDD
prd_metrics = extract_metrics(prd)
sdd_tracking = extract_tracking_implementations(sdd)
for metric in prd_metrics:
    if metric not in sdd_tracking:
        FAIL(f"PRD metric '{metric}' has no tracking implementation in SDD")
```

**Cross-Document Consistency** (PLAN ↔ SDD):
```python
# Every SDD component must have implementation tasks in PLAN
sdd_components = extract_components("docs/specs/ID/solution-design.md")
plan_tasks = extract_tasks("docs/specs/ID/implementation-plan.md")

for component in sdd_components:
    if not has_implementation_tasks(component, plan_tasks):
        FAIL(f"SDD component '{component}' has no implementation tasks in PLAN")

# PLAN task references must point to existing SDD sections
plan_tasks = extract_tasks(plan)
for task in plan_tasks:
    if task.references_sdd:
        sdd_ref = parse_sdd_reference(task)
        if not sdd_ref_exists(sdd_ref, sdd):
            FAIL(f"Task '{task.id}' references non-existent SDD section '{sdd_ref}'")
```

**Failure Example**:
```
⚠️ Consistency Failures:
  • PRD feature "Multi-tenant authentication" not implemented in SDD
    Impact: Feature missing from technical design
    Fix: Add component to SDD Section 4

  • Task T1.3.1 references [ref: SDD/Section 6.5] but SDD only has 5 sections
    Impact: Broken reference, unclear implementation guidance
    Fix: Update reference to correct section or add Section 6.5 to SDD
```

---

### BR-QG-006: Task Definition of Done Enforcement

**Rule**: TASK-DOD validation MUST be performed after each implementation task completes

**Given** developer completes implementation task T1.3.1
**When** TASK-DOD validation runs
**Then** execute automated checks:
1. Project builds without errors
2. All tests pass
3. Coverage meets threshold (e.g., ≥80%)
4. Linting passes
5. Code is formatted

**Optional checks** (if configured):
- TDD cycle validation (RED → GREEN → REFACTOR)
- Integration tests pass
- E2E tests pass
- Security scan passes
- Performance benchmarks pass

**Build Success Check**:
```bash
# Run build command from configuration
npm run build
# Expected: exit code 0
```

**Test Execution Check**:
```bash
# Run test command from configuration
npm test
# Expected: exit code 0, all tests passing
```

**Coverage Check**:
```bash
# Run coverage command
npm run test:coverage
# Parse output, extract percentage
# Expected: coverage ≥ threshold (e.g., 80%)
```

**Linting Check**:
```bash
# Run lint command
npm run lint
# Expected: exit code 0, no lint errors
```

**Formatting Check**:
```bash
# Run format check
npm run format:check
# Expected: exit code 0, no formatting changes needed
```

**Given** any critical check fails
**When** determining task completion
**Then** BLOCK with specific failures:
```
❌ Task Definition of Done: BLOCKED

Task: T1.3.1 Implement authentication service
Overall: 4/6 checks passed (67%)
Critical: 2/3 checks passed (67%)

⛔ Critical Failures:
  • Tests failing (3 failures)
    Impact: Feature is broken
    Fix: Run npm test and fix failures

⚠️ Non-Critical Failures:
  • Coverage below threshold (72% < 80%)
    Impact: Insufficient test coverage
    Fix: Add tests for uncovered code paths

🔧 Next Actions:
  1. Fix failing tests
  2. Add tests to reach 80% coverage
  3. Re-run validation

Cannot mark task complete until critical items resolved.
```

---

### BR-QG-007: TDD Cycle Validation (Optional)

**Rule**: If TDD enforcement enabled, validate RED → GREEN → REFACTOR cycle was followed

**TDD Cycle Phases**:

1. **RED (Prime Phase)**: Write failing tests first
   - Test files exist for component
   - Tests are initially failing (RED state)
   - Test failures are meaningful (not syntax errors)

2. **GREEN (Implement Phase)**: Make tests pass with minimal code
   - All tests now pass (GREEN state)
   - No skipped or ignored tests
   - Implementation is simplest solution that works

3. **REFACTOR (Validate Phase)**: Improve code while keeping tests passing
   - Tests still pass after refactoring
   - Code coverage maintained or improved
   - No new lint warnings introduced
   - Code is more readable than before

**Automated Validation**:
```python
# Check RED phase documented
test_file = find_test_file(component)
impl_file = find_implementation_file(component)

# Test file should be created before implementation file
assert test_file.created_at < impl_file.created_at, "Test file must be created before implementation"

# Verify initial test run failed
initial_test_result = get_test_result_at(test_file.created_at)
assert initial_test_result.status == "FAILED", "Tests must fail initially (RED phase)"

# Verify current test run succeeds
current_test_result = get_test_result_at(impl_file.modified_at)
assert current_test_result.status == "PASSED", "Tests must pass after implementation (GREEN phase)"
```

**Manual Verification**:
- Was test written BEFORE implementation?
- Does test fail for right reason (not syntax error)?
- Did you write minimal code to make test pass?
- Is code more readable after refactoring?

**Failure Example**:
```
⚠️ TDD Failures:
  • Implementation file timestamp (14:32:05) before test file (14:35:12)
    Impact: TDD cycle violated (should be test-first)
    Fix: Follow RED-GREEN-REFACTOR: write test first, then implementation
```

---

### BR-QG-008: Threshold Configuration

**Rule**: Quality gate thresholds are configurable per project but critical items always require 100%

**Configuration Structure**:
```yaml
thresholds:
  critical: 100              # Always 100% (not configurable)

  dor_overall: 85            # Definition of Ready overall threshold
  dod_overall: 85            # Definition of Done overall threshold
  task_dod_overall: 85       # Task DOD overall threshold

  coverage_target: 80        # Code coverage target percentage

validation_level: balanced   # strict | balanced | advisory

features:
  scqa: true                 # Enable SCQA validation
  scqa_scope: all            # all | prd-sdd | prd-only
  mece: true                 # Enable MECE validation
  mece_scope: all            # all | prd-sdd | prd-only
  consistency: automated     # automated | manual | disabled
  tdd: false                 # Enable TDD cycle validation
```

**Threshold Ranges**:
- **Critical**: Always 100% (immutable)
- **Overall**: 70-95% (recommended 85%)
- **Coverage**: 60-95% (recommended 80%)

**Project Maturity Guidelines**:
- **New projects**: 70% overall (more exploration, less certainty)
- **Standard projects**: 85% overall (balanced)
- **Critical systems**: 95% overall (high certainty required)

**Given** project sets overall threshold: 90%
**When** validation runs with score: 88%
**Then** BLOCK (below threshold)

**Given** project sets overall threshold: 85%
**When** validation runs with score: 88%
**Then** PROCEED (above threshold)

---

### BR-QG-009: Validation Execution Flow

**Rule**: Quality gate validation follows strict execution order with early exit on failures

**Execution Sequence**:
1. Run all automated checks (parallel where possible)
2. Calculate scores (critical and overall)
3. Apply enforcement decision based on validation level
4. If automated passes, present manual verification checklist
5. Collect manual verification responses
6. Apply final decision (pass only if both automated + manual complete)

**Flow Diagram**:
```
Start
  ↓
Run Automated Checks
  ↓
Calculate Scores
  ↓
Critical < 100%? ────Yes────→ BLOCK (Critical failures)
  ↓ No
  ↓
Overall < Threshold? ────Yes────→ Check Validation Level
  ↓ No                              ↓
  ↓                          Strict/Balanced: BLOCK
  ↓                          Advisory: WARN → Continue
  ↓
Present Manual Verification
  ↓
User Confirms All Items? ────No────→ BLOCK (Manual verification incomplete)
  ↓ Yes
  ↓
PROCEED
```

**Example**:
```python
def run_validation(checks, threshold, validation_level):
    # 1. Run automated checks
    results = execute_checks(checks)

    # 2. Calculate scores
    critical_score = calculate_critical_score(results)
    overall_score = calculate_overall_score(results)

    # 3. Early exit on critical failure
    if critical_score < 100:
        return BLOCK("Critical prerequisites incomplete", results)

    # 4. Check overall threshold
    if overall_score < threshold:
        if validation_level == "strict" or validation_level == "balanced":
            return BLOCK("Overall readiness below threshold", results)
        elif validation_level == "advisory":
            WARN("Overall readiness below threshold (advisory only)")

    # 5. Manual verification
    manual_results = present_manual_checklist(checks)

    # 6. Final decision
    if all(manual_results):
        return PROCEED()
    else:
        return BLOCK("Manual verification incomplete", manual_results)
```

---

## Workflows

### Workflow 1: DOR Enforcement Before SDD Creation

**Actor**: Developer ready to create SDD
**Trigger**: PRD completed, need to start SDD

**Steps**:
1. Developer runs: `/start:specify 007` (continue existing spec)
2. System loads DOR template: `definition-of-ready.md`
3. System executes "Before Creating SDD" automated checks:
   ```bash
   # Check PRD exists
   test -f docs/specs/007-user-auth/product-requirements.md
   # Result: ✅ PASS

   # Check PRD has no placeholders
   grep -c "\[NEEDS CLARIFICATION" docs/specs/007-user-auth/product-requirements.md
   # Result: 3 matches ❌ FAIL
   ```
4. System calculates scores:
   - Critical: 3/4 (75%) ❌ Below 100%
   - Overall: 5/7 (71%) ❌ Below 85%
5. System blocks with failure message:
   ```
   ❌ Definition of Ready: BLOCKED

   Document: SDD
   Critical: 3/4 checks passed (75%)

   ⛔ Critical Failures:
     • PRD has 3 [NEEDS CLARIFICATION] markers (lines 45, 67, 89)

   Cannot proceed until critical items resolved.
   ```
6. Developer completes PRD sections
7. Developer re-runs validation: scores now 100% critical, 100% overall
8. System presents manual verification checklist
9. Developer confirms manual items
10. System allows SDD creation to proceed

**Postcondition**: SDD creation permitted only after DOR validation passes

---

### Workflow 2: DOD Validation After PRD Completion

**Actor**: Developer finished PRD content
**Trigger**: All PRD sections filled, ready to validate

**Steps**:
1. Developer marks PRD complete
2. System runs DOD validation for "PRD Completion"
3. Automated checks:
   ```bash
   # All required sections present
   grep "^## " product-requirements.md | diff - template-headers.txt
   # Result: ✅ PASS

   # No placeholders remaining
   grep -c "\[NEEDS CLARIFICATION" product-requirements.md
   # Result: 0 ✅ PASS

   # Validation checklist complete
   grep -c "^- \[x\]" product-requirements.md
   # Result: 17/17 ✅ PASS

   # Minimum word count
   wc -w product-requirements.md
   # Result: 1247 words (≥500) ✅ PASS
   ```
4. SCQA validation (if enabled):
   ```python
   sections = extract_sections(prd)
   assert "Context" in sections  # ✅ PASS
   assert "Problem Statement" in sections  # ✅ PASS
   assert "Solution Overview" in sections  # ✅ PASS
   validate_order(sections, ["Context", "Problem", "Solution"])  # ✅ PASS
   ```
5. MECE validation (if enabled):
   ```python
   # Check for duplicate user stories
   stories = extract_user_stories(prd)
   # Result: No duplicates ✅ PASS

   # Check every persona has journey
   personas = extract_personas(prd)
   journeys = extract_journeys(prd)
   # Result: All personas have journeys ✅ PASS
   ```
6. All automated checks pass
7. System presents manual verification:
   - "Is every section meaningful?"
   - "Would a new team member understand this?"
8. Developer confirms both items
9. System marks PRD as complete
10. System allows progression to SDD creation

**Postcondition**: PRD validated and ready for SDD phase

---

### Workflow 3: TASK-DOD Validation After Implementation

**Actor**: Developer completed implementation task
**Trigger**: Code written, ready to mark task complete

**Steps**:
1. Developer completes task T1.3.1: "Implement authentication service"
2. Developer runs: validation command
3. System executes TASK-DOD checks:
   ```bash
   # Build success
   npm run build
   # Exit code: 0 ✅ PASS

   # Test execution
   npm test
   # Exit code: 1 ❌ FAIL (3 test failures)

   # Coverage
   npm run test:coverage
   # Result: 72% ❌ Below 80% threshold

   # Linting
   npm run lint
   # Exit code: 0 ✅ PASS

   # Formatting
   npm run format:check
   # Exit code: 0 ✅ PASS
   ```
4. System calculates scores:
   - Critical: 2/3 (67%) ❌ Below 100%
   - Overall: 4/6 (67%) ❌ Below 85%
5. System blocks with specific failures:
   ```
   ❌ Task Definition of Done: BLOCKED

   ⛔ Critical Failures:
     • Tests failing (3 failures)

   ⚠️ Non-Critical Failures:
     • Coverage below threshold (72% < 80%)

   🔧 Next Actions:
     1. Fix failing tests
     2. Add tests to reach 80% coverage
     3. Re-run validation
   ```
6. Developer fixes failing tests
7. Developer adds tests to increase coverage
8. Developer re-runs validation: all checks pass
9. System allows task to be marked complete

**Postcondition**: Task completion verified with quality standards

---

## Edge Cases

### Edge Case 1: Validation Level Mismatch

**Given** critical score: 80%, overall score: 90%, validation level: advisory
**When** enforcement decision runs
**Then** system warns but allows progression

**Given** critical score: 80%, overall score: 90%, validation level: strict
**When** enforcement decision runs
**Then** system blocks (critical must be 100%)

**Mitigation**: Document validation level choice, ensure team understands implications

---

### Edge Case 2: Threshold Boundary

**Given** overall score: 85.0%, threshold: 85%
**When** comparing scores
**Then** PROCEED (equal to threshold counts as pass)

**Given** overall score: 84.9%, threshold: 85%
**When** comparing scores
**Then** BLOCK (below threshold)

**Implementation**: Use `>=` comparison, not `>`

---

### Edge Case 3: Manual Verification Bypassed

**Given** automated checks all pass
**When** manual verification step presented
**Then** user required to confirm each item

**Given** user attempts to skip manual verification
**When** enforcement decision runs
**Then** BLOCK until manual verification completed

**Constraint**: Both automated AND manual must pass

---

### Edge Case 4: Configuration Placeholder Not Resolved

**Given** TASK-DOD template with placeholder:
```yaml
build: [NEEDS CLARIFICATION: build command]
```

**When** attempting to run automated check
**Then** validation fails with: "Build command not configured"

**Mitigation**: Validation should check configuration completeness before running checks

---

### Edge Case 5: SCQA Validation False Positive

**Given** PRD with sections: "Background", "Current State Problem", "Proposed Solution"
**When** SCQA validation runs expecting: "Context", "Problem Statement", "Solution Overview"
**Then** validation fails (section names don't match exactly)

**Impact**: False negative blocks progression despite correct logical flow
**Mitigation**: Use fuzzy matching or semantic analysis for section name detection

---

### Edge Case 6: Zero Tests Written

**Given** implementation complete but no tests written
**When** TASK-DOD validation runs
**Then** "All tests pass" check succeeds (vacuously true: 0/0 tests pass)

**Impact**: Misleading success when no tests exist
**Mitigation**: Add check for minimum test count or coverage requirement

---

## Validation Rules

### VR-QG-001: Critical Score Enforcement

**Rule**: Critical score must always be 100% regardless of validation level

**Check**:
```python
critical_score = (critical_passed / critical_total) * 100
assert critical_score == 100, "Critical items must be 100% complete"
```

**Applies To**: All validation levels (strict, balanced, advisory)

---

### VR-QG-002: Overall Score Threshold

**Rule**: Overall score must meet or exceed configured threshold

**Check**:
```python
overall_score = (total_passed / total_checks) * 100
if validation_level in ["strict", "balanced"]:
    assert overall_score >= threshold, f"Overall score {overall_score}% below threshold {threshold}%"
elif validation_level == "advisory":
    if overall_score < threshold:
        WARN(f"Overall score {overall_score}% below threshold {threshold}% (advisory only)")
```

---

### VR-QG-003: Manual Verification Completeness

**Rule**: All manual verification items must be confirmed by user

**Check**:
```python
manual_items = get_manual_verification_items()
user_responses = collect_user_responses(manual_items)

for item, response in zip(manual_items, user_responses):
    if response != "confirmed":
        FAIL(f"Manual verification item not confirmed: {item}")
```

---

### VR-QG-004: Placeholder Detection

**Rule**: Document completeness determined by absence of `[NEEDS CLARIFICATION]` markers

**Check**:
```bash
grep -c "\[NEEDS CLARIFICATION" document.md
# Expected: 0 for complete document
```

**Regex**: `\[NEEDS CLARIFICATION[^\]]*\]`

---

### VR-QG-005: Template Section Structure Match

**Rule**: Document section headers must match template section headers exactly

**Check**:
```bash
# Extract headers from template
grep "^##" template.md | sort > template-headers.txt

# Extract headers from document
grep "^##" document.md | sort > doc-headers.txt

# Compare
diff template-headers.txt doc-headers.txt
# Expected: No differences
```

---

### VR-QG-006: Configuration Completeness

**Rule**: All configuration placeholders must be resolved before validation execution

**Check**:
```python
config = load_config("task-definition-of-done.md")
required_fields = ["build", "test", "lint", "format", "coverage_target"]

for field in required_fields:
    value = config.get(field)
    if "[NEEDS CLARIFICATION" in str(value):
        FAIL(f"Configuration field '{field}' not resolved")
```

---

## Configuration

### Quality Gate Configuration File Structure
```yaml
# Quality gate thresholds
thresholds:
  critical: 100              # Always 100% (immutable)
  dor_overall: 85            # Definition of Ready overall threshold (70-95%)
  dod_overall: 85            # Definition of Done overall threshold (70-95%)
  task_dod_overall: 85       # Task DOD overall threshold (70-95%)
  coverage_target: 80        # Code coverage target percentage (60-95%)

# Validation enforcement level
validation_level: balanced   # Options: strict | balanced | advisory

# Feature toggles
features:
  scqa: true                 # Enable SCQA validation
  scqa_scope: all            # Options: all | prd-sdd | prd-only

  mece: true                 # Enable MECE validation
  mece_scope: all            # Options: all | prd-sdd | prd-only

  consistency: automated     # Options: automated | manual | disabled

  tdd: false                 # Enable TDD cycle validation

  minimal_docs: false        # Enable minimal documentation mode

# Language-specific settings
language: javascript         # Options: javascript | python | go | rust | etc.

# Project commands (for TASK-DOD)
commands:
  build: npm run build
  test: npm test
  coverage: npm run test:coverage
  lint: npm run lint
  format: npm run format:check

  # Optional commands
  integration_test: npm run test:integration
  e2e_test: npm run test:e2e
  security_scan: npm run security:scan
  benchmark: npm run benchmark
```

### Validation Level Behavior Matrix

| Level    | Critical Failure | Overall Below Threshold | Non-Critical Failure |
|----------|------------------|-------------------------|----------------------|
| Strict   | BLOCK            | BLOCK                   | BLOCK                |
| Balanced | BLOCK            | BLOCK                   | WARN                 |
| Advisory | WARN             | WARN                    | WARN                 |

---

## FAQ

**Q: Can I skip quality gates for urgent fixes?**
A: Set `validation_level: advisory` temporarily. All checks run but nothing blocks. Document why and revert after fix.

**Q: What if automated checks have false positives?**
A: Review failure details. If incorrect, adjust check or threshold. Document exceptions. Consider disabling specific check.

**Q: Why are critical items always 100%?**
A: Critical items are must-haves. Starting without them guarantees rework. No exceptions to 100% rule.

**Q: Can I customize which checks are critical?**
A: Yes, edit quality gate templates. Move checks between critical/non-critical sections. Document rationale.

**Q: How do I handle SCQA/MECE validation failures?**
A: Review specific failure. If false positive, disable check temporarily. If real issue, fix document content.

**Q: What's the difference between DOR and DOD?**
A: DOR = prerequisites BEFORE starting. DOD = completion criteria AFTER finishing. Both enforce quality at different stages.

**Q: Can different features have different thresholds?**
A: Yes, configure per specification or per phase. Document threshold choices and reasons.

**Q: What if I can't reach coverage threshold?**
A: Either add tests or adjust threshold. Untestable code (logs, error messages) can justify lower threshold. Document decision.

**Q: How do I know if refactoring is needed?**
A: Look for code smells: duplication, long functions, unclear names, tight coupling. Refactor during VALIDATE phase.

**Q: What happens if manual verification is incomplete?**
A: Validation blocks. Developer must confirm all manual items. Cannot bypass with advisory mode.

**Q: Can I disable SCQA/MECE validation?**
A: Yes, set `features.scqa: false` and `features.mece: false`. Useful for simple features or prototypes.

**Q: What if cross-document consistency check fails?**
A: Usually indicates missing content. Either add missing content or mark as intentional scope reduction. Document why.
