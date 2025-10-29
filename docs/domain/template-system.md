# Template System Business Rules

## Business Context

The template system provides structured document templates that enforce consistency across all specifications. Templates act as contracts that define required sections, validation checklists, and placeholder mechanisms for progressive content completion.

**Domain**: Documentation Framework
**Purpose**: Standardize document structure, guide content creation, enforce quality gates
**Actors**: Developers, Specification Tools, Validation Systems

---

## Core Concepts

### Template Taxonomy

The system provides **9 template types** organized into 3 categories:

#### Core Specification Templates (3)
1. **product-requirements.md** (PRD): Business requirements, user needs, success metrics
2. **solution-design.md** (SDD): Technical architecture, design decisions, component specifications
3. **implementation-plan.md** (PLAN): Execution tasks, test-driven development phases, validation steps

#### Quality Gate Templates (3)
4. **definition-of-ready.md** (DOR): Prerequisites before creating PRD/SDD/PLAN
5. **definition-of-done.md** (DOD): Completion criteria after creating PRD/SDD/PLAN
6. **task-definition-of-done.md** (TASK-DOD): Implementation task validation during execution

#### Supporting Documentation Templates (3)
7. **Domain Documentation**: Business rules, workflows, state machines, validation logic
8. **Pattern Documentation**: Technical patterns, architectural solutions, code structures
9. **Interface Documentation**: External API contracts, service integration specifications

**Constraint**: Core templates define mandatory sections that MUST NOT be modified during use

---

### Template Structure

Each template consists of:

1. **Validation Checklist**: Top-level verification items for document completeness
2. **Section Headers**: Required organizational structure (`## Section Name`)
3. **Subsection Headers**: Nested structure (`### Subsection Name`)
4. **Content Placeholders**: `[NEEDS CLARIFICATION: prompt text]` markers for progressive completion
5. **Optional Sections**: Conditional blocks using HTML comments (`<!-- OPTIONAL: feature -->`)
6. **Conditional Sections**: Feature-gated content (`<!-- IF: condition -->`)
7. **Metadata Annotations**: Cross-references and activity hints (`[ref: doc/section]`, `[activity: type]`)

**Structural Contract**: The hierarchy of headers defines the document's skeleton and MUST be preserved

---

### Placeholder Types

#### Content Placeholders
Markers indicating information to be filled in during specification:
```markdown
[NEEDS CLARIFICATION: What is the one-sentence vision for this feature?]
```

**Rules**:
- Appear within document sections
- Replaced progressively (only current section being worked on)
- Remaining placeholders left untouched for future cycles
- Zero placeholders = document complete (for that section)

#### Configuration Placeholders
Settings requiring project-specific values:
```yaml
build: [NEEDS CLARIFICATION: build command]
test: [NEEDS CLARIFICATION: test command]
coverage_target: [NEEDS CLARIFICATION: coverage target]  # Default: 80%
threshold: [NEEDS CLARIFICATION: dor threshold]  # Default: 85%
```

**Rules**:
- Appear in configuration sections
- Require explicit project setup
- Often have default values specified in comments
- Used by validation automation

#### Conditional Placeholders
Content that appears/disappears based on feature flags:
```markdown
<!-- OPTIONAL: MINIMAL_DOCS -->
### Minimal Mode
For teams using minimal documentation approach...
<!-- END OPTIONAL: MINIMAL_DOCS -->

<!-- IF: LANGUAGE_PYTHON -->
#### Python Environment
**Automated Checks**:
- [ ] Virtual environment activated
<!-- END IF -->
```

**Rules**:
- Entire blocks included or excluded based on configuration
- No partial inclusion within conditional blocks
- Can be nested within optional sections

---

## Business Rules

### BR-TMPL-001: Template Discovery and Loading

**Rule**: Templates are discovered from the plugin directory and loaded as immutable structural contracts

**Given** spec creation request with template: `product-requirements`
**When** loading template
**Then** read from: `plugins/start/templates/product-requirements.md`

**Given** template file does not exist
**When** attempting to load template
**Then** fail with: "Template {name}.md not found"

**Template Discovery Location**:
- **Base Path**: `plugins/start/templates/`
- **Resolution**: Relative to plugin root, NOT current working directory
- **File Format**: Markdown (`.md`)
- **Naming Convention**: Lowercase with hyphens (e.g., `product-requirements.md`)

**Loading Mechanism** (from `spec.py`):
```python
template_file = TEMPLATES_DIR / f"{template}.md"
if not template_file.exists():
    print(f"Warning: Template {template_file} not found")
else:
    dest_file = spec_dir / f"{template}.md"
    dest_file.write_text(template_file.read_text())
```

**Constraint**: Templates are copied verbatim - all placeholders and structure preserved
**Validation**: Template file existence checked before copy operation

---

### BR-TMPL-002: Structural Contract Adherence

**Rule**: Template structure (headers, sections, subsections) MUST be preserved exactly - no additions, deletions, or reorganizations allowed

**Given** PRD template with sections:
```markdown
## Product Overview
### Vision
### Problem Statement
### Value Proposition

## User Personas
### Primary Persona
### Secondary Personas
```

**When** filling out PRD
**Then** preserve exact section structure

**FORBIDDEN**:
- Adding new sections: ❌ `## Additional Context`
- Removing sections: ❌ Deleting `## User Personas`
- Reordering sections: ❌ Moving `## Success Metrics` before `## User Personas`
- Renaming sections: ❌ Changing `## Product Overview` to `## Overview`
- Adding subsections: ❌ Adding `### Tertiary Personas` under `## User Personas`

**ALLOWED**:
- Replacing `[NEEDS CLARIFICATION]` markers with content
- Filling in section content following template structure
- Adding content within existing section boundaries
- Marking optional sections for inclusion/exclusion

**Impact of Violation**: Breaks validation automation that relies on section structure for completeness checks

**Example Violation**:
```markdown
<!-- Original Template -->
## Feature Requirements
### Must Have Features
### Should Have Features
### Could Have Features
### Won't Have (This Phase)

<!-- ❌ VIOLATION: Added new subsection -->
## Feature Requirements
### Must Have Features
### Should Have Features
### Could Have Features
### Nice to Have Features  ← NEW, breaks contract
### Won't Have (This Phase)
```

---

### BR-TMPL-003: Progressive Placeholder Replacement

**Rule**: Placeholders are replaced progressively - only in the current section being worked on, leaving all other placeholders untouched

**Given** PRD with multiple sections containing placeholders:
```markdown
## Product Overview
### Vision
[NEEDS CLARIFICATION: What is the one-sentence vision?]

### Problem Statement
[NEEDS CLARIFICATION: What specific problem are users facing?]

## User Personas
### Primary Persona: [NEEDS CLARIFICATION: persona name]
[NEEDS CLARIFICATION: Demographics, goals, pain points]
```

**When** working on "Product Overview" section during current cycle
**Then** replace only placeholders in "Product Overview":
```markdown
## Product Overview
### Vision
Enable small teams to manage multi-tenant SaaS with zero infrastructure overhead.

### Problem Statement
Current solutions require dedicated DevOps expertise for tenant isolation and data security.

## User Personas
### Primary Persona: [NEEDS CLARIFICATION: persona name]
[NEEDS CLARIFICATION: Demographics, goals, pain points]
```

**Constraint**: Placeholders in "User Personas" remain untouched for future cycles
**Rationale**: Allows focused, iterative document completion without context overload

**Command-Specific Rule** (from `/start:specify`):
> Replace [NEEDS CLARIFICATION] markers with actual content only for sections related to the current checklist item. Leave all other sections' [NEEDS CLARIFICATION] markers untouched for future cycles.

---

### BR-TMPL-004: Template as Completion Signal

**Rule**: A document is considered complete for a section when no `[NEEDS CLARIFICATION]` markers remain in that section

**Given** PRD section:
```markdown
## Product Overview
### Vision
Enable small teams to manage multi-tenant SaaS with zero infrastructure overhead.

### Problem Statement
[NEEDS CLARIFICATION: What specific problem are users facing?]
```

**When** checking section completion
**Then** "Vision" subsection is complete, "Problem Statement" is incomplete

**Detection**:
```bash
# Check section completeness
grep -c "\[NEEDS CLARIFICATION" product-requirements.md
# Count > 0 means document incomplete
```

**State Transition**:
- Document with placeholders → "In Progress" state
- Document with zero placeholders → "Complete" state

**Used By**: Specification lifecycle state machine (see BR-SPEC in specification-lifecycle.md)

---

### BR-TMPL-005: Validation Checklist Structure

**Rule**: Every core template includes a top-level validation checklist that defines completion criteria

**Given** PRD template structure:
```markdown
# Product Requirements Document

## Validation Checklist

- [ ] All required sections are complete
- [ ] No [NEEDS CLARIFICATION] markers remain
- [ ] Problem statement is specific and measurable
- [ ] Problem is validated by evidence (not assumptions)
- [ ] Context → Problem → Solution flow makes sense
...

---

## Product Overview
...
```

**When** validating document completeness
**Then** use validation checklist as source of truth for required criteria

**Checklist Properties**:
- Located at document top, before content sections
- Followed by horizontal rule separator (`---`)
- Contains checkbox items (`- [ ]`)
- Each item is a testable criterion
- Mix of automated checks (structural) and manual checks (judgment)

**Purpose**:
- Define "definition of done" for document
- Guide authors through completion
- Enable automated validation
- Provide quality gate enforcement

---

### BR-TMPL-006: Cross-Reference Notation

**Rule**: Templates use standardized notation for referencing other documents, sections, and external resources

**Reference Syntax**:
```markdown
[ref: document/section]              # Reference to another document
[ref: document/section; lines: 1-10] # Reference with specific lines
@docs/domain/business-rules.md       # File path reference
```

**Examples**:
```markdown
- [ ] T1.1.1 Read interface contracts `[ref: interfaces/api-spec; lines: 1-10]`
- [ ] T1.3.1 Implement authentication `[ref: SDD/Section 4.2]`
- See domain rules at @docs/domain/auth-workflow.md
```

**Purpose**:
- Enable traceability between specifications
- Link implementation tasks to design decisions
- Reference supporting documentation
- Support validation automation (verify referenced sections exist)

**Used In**: Implementation plans for linking tasks to PRD/SDD sections

---

### BR-TMPL-007: Activity Hint Annotation

**Rule**: Implementation tasks include activity hints that guide specialist agent selection

**Activity Hint Syntax**:
```markdown
[activity: type]
[activity: type1, type2, type3]
```

**Examples**:
```markdown
- [ ] T1.2.1 Write authentication tests `[activity: write-tests]`
- [ ] T1.3.1 Implement user service `[activity: implement-backend]`
- [ ] T1.5.1 Review code quality `[activity: lint-code, format-code, review-code]`
- [ ] T2.1.1 Research database patterns `[activity: research-patterns]`
```

**Activity Types**:
- `write-tests`: Test-driven development
- `implement-backend`: Backend implementation
- `implement-frontend`: Frontend implementation
- `lint-code`: Code linting
- `format-code`: Code formatting
- `review-code`: Code review
- `run-tests`: Test execution
- `research-patterns`: Pattern research
- `business-acceptance`: Specification compliance validation

**Purpose**:
- Guide automated agent delegation
- Indicate type of work for task
- Enable parallel execution (same activity type can be batched)
- Support workflow orchestration

---

### BR-TMPL-008: Optional and Conditional Sections

**Rule**: Templates support optional and conditional sections using HTML comment markers

**Optional Section Syntax**:
```markdown
<!-- OPTIONAL: feature_name -->
### Optional Feature Section
Content that may or may not be included...
<!-- END OPTIONAL: feature_name -->
```

**Conditional Section Syntax**:
```markdown
<!-- IF: condition -->
### Conditional Content
Content shown only when condition is true...
<!-- END IF -->
```

**Examples**:
```markdown
<!-- OPTIONAL: MINIMAL_DOCS -->
### Minimal Mode
For teams using minimal documentation:
- Reduce checklist items by 50%
- Focus only on critical prerequisites
<!-- END OPTIONAL: MINIMAL_DOCS -->

<!-- IF: LANGUAGE_PYTHON -->
#### Python-Specific Checks
- [ ] Virtual environment activated
- [ ] Requirements installed
<!-- END IF -->

<!-- IF: LANGUAGE_GO -->
#### Go-Specific Checks
- [ ] No race conditions (check: go test -race)
- [ ] Vet passes (check: go vet)
<!-- END IF -->
```

**Rules**:
- Optional sections: Entire block included or excluded
- Conditional sections: Mutually exclusive choices (IF/ELSE IF/ELSE pattern)
- Markers must be balanced (each opening has closing marker)
- Can be nested within each other

**Configuration-Driven**:
```yaml
features:
  minimal_docs: true
  language: python
```

**Processing**: Template processor includes/excludes blocks based on configuration

---

### BR-TMPL-009: Metadata Annotations

**Rule**: Templates include metadata annotations for parallel execution, component tagging, and dependency tracking

**Metadata Syntax**:
```markdown
[parallel: true]           # Tasks can run concurrently
[component: name]          # Multi-component feature tagging
[ref: doc/section]         # Specification reference
[activity: type]           # Activity hint for agent selection
```

**Examples**:
```markdown
- [ ] T2.1 Component A `[parallel: true]` `[component: auth-service]`
    - [ ] T2.1.1 Write tests `[activity: write-tests]`
    - [ ] T2.1.2 Implement `[activity: implement-backend]`

- [ ] T2.2 Component B `[parallel: true]` `[component: user-service]`
    - [ ] T2.2.1 Write tests `[activity: write-tests]`
    - [ ] T2.2.2 Implement `[activity: implement-backend]`

- [ ] T3.1 Integration Tests `[ref: PRD/Section 5.2]`
```

**Usage**:
- `[parallel: true]`: Marks tasks that can execute simultaneously
- `[component: name]`: Groups related tasks for multi-component features
- `[ref: doc/section]`: Links tasks to specification sections for traceability
- `[activity: type]`: Hints for agent selection during implementation

**Purpose**:
- Enable parallelization during implementation
- Track component boundaries
- Enforce traceability to specifications
- Guide automated workflow orchestration

---

## Template Versioning

### Versioning Model

**Rule**: Templates are versioned at the plugin level, not individual file level

**Given** plugin version: `1.2.0`
**When** templates are updated
**Then** all templates share plugin version: `1.2.0`

**Version Location**: Plugin manifest or release tag, NOT within template files

**Migration Strategy**:
- Old specifications use templates from their creation version
- New specifications use latest plugin templates
- No automatic migration of existing specifications
- Manual migration if template structure changes significantly

**Backward Compatibility**:
- Minor template changes (new optional sections): Compatible
- Structural changes (reorder sections, rename headers): Breaking
- Breaking changes require major version bump

---

## Workflows

### Workflow 1: Template Application During Specification Creation

**Actor**: Developer using `/start:specify` command
**Trigger**: New specification initiated

**Steps**:
1. Developer runs: `/start:specify "User Authentication"`
2. System creates spec directory: `docs/specs/007-user-authentication/`
3. System copies PRD template: `plugins/start/templates/product-requirements.md` → `docs/specs/007-user-authentication/product-requirements.md`
4. PRD contains all placeholders intact:
   ```markdown
   ### Vision
   [NEEDS CLARIFICATION: What is the one-sentence vision for this feature?]
   ```
5. Developer works through validation checklist, replacing placeholders section by section
6. System detects completion when no `[NEEDS CLARIFICATION]` markers remain

**Postcondition**: Specification has structured PRD following template contract

---

### Workflow 2: Progressive Section Completion

**Actor**: Developer iterating on PRD
**Trigger**: Working through validation checklist

**Steps**:
1. Developer reads validation checklist item 1: "Problem statement is specific and measurable"
2. Developer focuses on "Problem Statement" section
3. Developer replaces placeholder with actual content:
   ```markdown
   ### Problem Statement
   Small development teams lack cost-effective multi-tenancy solutions. Current options require dedicated DevOps expertise, making SaaS deployment prohibitively expensive for teams under 10 people.
   ```
4. Developer leaves other sections untouched:
   ```markdown
   ### Value Proposition
   [NEEDS CLARIFICATION: Why will users choose this solution over alternatives?]
   ```
5. Developer moves to checklist item 2, repeats process

**Postcondition**: Document fills in progressively, maintaining structure

---

### Workflow 3: Quality Gate Template Usage

**Actor**: Automated validation system
**Trigger**: Document completion checkpoint (PRD finished, ready for SDD)

**Steps**:
1. System loads DOR template for "Before Creating SDD" section
2. System runs automated checks:
   ```bash
   # Check: PRD file exists
   test -f docs/specs/007-user-authentication/product-requirements.md

   # Check: PRD has no placeholders
   grep -c "\[NEEDS CLARIFICATION" product-requirements.md
   # Expected: 0
   ```
3. System calculates score:
   - Critical items: 4/4 (100%)
   - Overall items: 7/7 (100%)
4. System presents manual verification checklist to developer
5. Developer confirms manual checks
6. System allows progression to SDD creation

**Postcondition**: Quality gate enforced, SDD creation permitted

---

### Workflow 4: Template Customization for Project

**Actor**: Project lead
**Trigger**: Need project-specific template adjustments

**Steps**:
1. Lead copies template to project: `cp plugins/start/templates/task-definition-of-done.md .skogai-team/task-dod-custom.md`
2. Lead fills in configuration placeholders:
   ```yaml
   commands:
     build: npm run build
     test: npm test
     coverage: npm run test:coverage
     lint: npm run lint

   coverage:
     target: 80%

   thresholds:
     overall: 85%
   ```
3. Lead adds language-specific sections:
   ```markdown
   <!-- IF: LANGUAGE_JAVASCRIPT -->
   #### JavaScript-Specific Checks
   - [ ] TypeScript compilation (check: tsc --noEmit)
   - [ ] Bundle size acceptable (check: npm run bundlesize)
   <!-- END IF -->
   ```
4. Project uses customized template for all implementations

**Postcondition**: Project has tailored quality gates

---

## Edge Cases

### Edge Case 1: Conflicting Template Modifications

**Given** developer modifies template structure by adding custom section
**When** validation automation runs
**Then** automation fails to find expected sections

**Scenario**:
```markdown
<!-- Original Template -->
## User Personas
### Primary Persona
### Secondary Personas

<!-- ❌ Modified (violates contract) -->
## User Personas
### Primary Persona
### Secondary Personas
### Tertiary Personas  ← ADDED
```

**Impact**: Validation checklist expects 2 subsections, finds 3
**Mitigation**: Validation should warn about unexpected structure
**Best Practice**: Use optional sections instead of modifying structure

---

### Edge Case 2: Partial Placeholder Replacement

**Given** placeholder spanning multiple lines:
```markdown
[NEEDS CLARIFICATION: What are the demographics, goals, and pain points of the primary persona? Include age range, role, technical expertise, success criteria, and frustrations with current solutions.]
```

**When** developer partially addresses prompt:
```markdown
Age: 25-40, Role: Startup Founder
[NEEDS CLARIFICATION: technical expertise, success criteria, and frustrations]
```

**Then** placeholder marker still present, section considered incomplete

**Detection**: `grep "\[NEEDS CLARIFICATION"` still returns match
**Best Practice**: Fully replace placeholder or leave entirely intact

---

### Edge Case 3: Template Version Mismatch

**Given** specification created with plugin version 1.0 (old template)
**When** plugin upgraded to version 2.0 (new template structure)
**Then** existing specification uses old structure, new specs use new structure

**Impact**: Cross-specification validation may fail if structure incompatible
**Mitigation**:
- Document template version in specification metadata
- Provide migration scripts for breaking changes
- Maintain backward compatibility where possible

---

### Edge Case 4: Circular References

**Given** PLAN task references SDD section:
```markdown
- [ ] T1.3.1 Implement auth service `[ref: SDD/Section 4.2]`
```

**When** SDD Section 4.2 references PLAN task:
```markdown
## 4.2 Authentication Service
Implementation details in [ref: PLAN/Task T1.3.1]
```

**Then** circular reference created

**Impact**: Confusing dependency graph, unclear source of truth
**Best Practice**: References flow one direction: PLAN → SDD → PRD (never reverse)

---

### Edge Case 5: Missing Template Files

**Given** plugin templates directory incomplete
**When** requesting template: `--add custom-quality-gate`
**Then** template file not found: `plugins/start/templates/custom-quality-gate.md`

**Current Behavior**: Warning issued, continues execution
**Improved Behavior**: Could fail fast with error and list available templates

---

## Validation Rules

### VR-TMPL-001: Section Header Structure Preservation

**Rule**: All section headers (`##`) and subsection headers (`###`) from template must be preserved

**Check**:
```bash
# Extract headers from template
grep "^##" plugins/start/templates/product-requirements.md > template-headers.txt

# Extract headers from filled document
grep "^##" docs/specs/007-user-authentication/product-requirements.md > doc-headers.txt

# Compare
diff template-headers.txt doc-headers.txt
# Expected: No differences
```

**Failure**: Any added, removed, or reordered headers

---

### VR-TMPL-002: Placeholder Syntax Validation

**Rule**: Placeholders must follow exact syntax: `[NEEDS CLARIFICATION: prompt text]`

**Valid**:
- `[NEEDS CLARIFICATION: What is the vision?]`
- `[NEEDS CLARIFICATION: persona name]`

**Invalid**:
- `[NEEDS CLARIFICATION]` (missing prompt)
- `[TBD: something]` (wrong marker)
- `[NEEDS_CLARIFICATION: text]` (underscore instead of space)

**Regex**: `\[NEEDS CLARIFICATION:[^\]]+\]`

---

### VR-TMPL-003: Validation Checklist Completeness

**Rule**: All checklist items must be checked (`[x]`) for document to be considered complete

**Check**:
```bash
# Count total checklist items
grep -c "^- \[ \]" product-requirements.md

# Count checked items
grep -c "^- \[x\]" product-requirements.md

# Expected: checked_items >= total_items (allowing extras)
```

**Note**: Unchecked items indicate incomplete validation

---

### VR-TMPL-004: Reference Target Existence

**Rule**: All `[ref: target]` annotations must point to existing documents/sections

**Check**:
```bash
# Extract references
grep -o "\[ref: [^\]]*\]" implementation-plan.md

# For each reference, verify target exists
# Example: [ref: SDD/Section 4.2] → verify Section 4.2 in SDD
```

**Failure**: Reference points to non-existent section
**Mitigation**: Validation tool should list broken references

---

### VR-TMPL-005: Configuration Placeholder Resolution

**Rule**: All configuration placeholders must be resolved with actual values before validation execution

**Check**:
```bash
# Find unresolved configuration placeholders
grep "\[NEEDS CLARIFICATION:" task-definition-of-done.md | grep -E "(build command|test command|coverage target)"

# Expected: No matches (all resolved)
```

**Examples of Resolution**:
```yaml
# Before
build: [NEEDS CLARIFICATION: build command]

# After
build: npm run build
```

---

## Configuration

### Template Directory Structure
```
plugins/start/templates/
├── product-requirements.md
├── solution-design.md
├── implementation-plan.md
├── definition-of-ready.md
├── definition-of-done.md
└── task-definition-of-done.md
```

### Template Feature Flags
```yaml
features:
  minimal_docs: false       # Enable minimal documentation mode
  scqa_validation: true     # Enable SCQA logical flow checks
  mece_validation: true     # Enable MECE coverage checks
  tdd_enforcement: true     # Enforce test-driven development

validation_level: balanced  # strict | balanced | advisory

language: javascript        # python | go | rust | etc.

thresholds:
  dor_overall: 85          # Definition of Ready threshold
  dod_overall: 85          # Definition of Done threshold
  task_dod_overall: 85     # Task DOD threshold
  coverage_target: 80      # Code coverage target percentage
```

---

## FAQ

**Q: Can I add my own sections to templates?**
A: No, this violates the structural contract. Use optional sections or supplementary documentation instead.

**Q: What if my project doesn't need all template sections?**
A: Mark sections as "N/A" or use conditional/optional sections. Don't delete sections.

**Q: How do I customize templates for my team?**
A: Copy templates to project directory, fill in configuration placeholders, add language-specific conditional sections.

**Q: Can I replace placeholder syntax with something else?**
A: Not recommended. Tooling expects `[NEEDS CLARIFICATION: ...]` syntax. Changing it breaks automation.

**Q: What happens if I accidentally delete a template section?**
A: Validation will fail when checking for required sections. Restore section from template.

**Q: How do I know which placeholders to fill in each cycle?**
A: Follow the validation checklist order. Each checklist item corresponds to specific sections/placeholders.

**Q: Can I reorder sections to match my mental model?**
A: No, section order is part of the structural contract. Templates enforce logical flow (Context → Problem → Solution).

**Q: What if a placeholder prompt doesn't make sense for my feature?**
A: Interpret it for your context or mark as "Not Applicable" with justification. Don't delete the section.

**Q: How do I handle templates that are too detailed for my simple feature?**
A: Enable `minimal_docs` mode or use abbreviated responses. Maintain structure even if sections are brief.

**Q: Can I create my own template types?**
A: Yes, for supplementary documentation (domain rules, patterns, interfaces). Core specification templates should follow provided structure.
