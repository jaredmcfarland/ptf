---
name: create-adapter
description: Create a new PTF domain adapter through guided wizard
argument-hint: "[domain-name]"
allowed-tools:
  - Read
  - Write
  - Bash
  - Glob
  - Grep
  - AskUserQuestion
---

<objective>
Guide the user through creating a new PTF domain adapter via an interactive
question-driven wizard. Validates each section against the adapter schema
and produces a valid YAML file at `.orchestrator/adapters/{name}.yaml`.

**Creates:**
- `.orchestrator/adapters/{name}.yaml` - New domain adapter

**After this command:** Use the adapter with `/ptf:init --domain={name}`
</objective>

<context>
Existing adapters for reference:
@.orchestrator/adapters/software-development.yaml
@.orchestrator/adapters/research.yaml
</context>

<process>

## Phase 1: Introduction and Domain Basics

1. **Welcome and explain the process:**

   Present to user:
   ```
   ## PTF Adapter Creation Wizard

   This wizard will guide you through creating a new domain adapter.
   We'll collect information in phases:

   1. Basic info (name, description)
   2. Init questions (for /ptf:init clarification)
   3. Decomposition heuristics and atomicity criteria
   4. Constitution template (immutable principles)
   5. Artifact types and verification strategies
   6. Dependency patterns
   7. Review and write

   Each phase builds on the previous. Let's start with the basics.
   ```

2. **Check for existing adapters:**
   ```bash
   ls .orchestrator/adapters/*.yaml 2>/dev/null | xargs -I{} basename {} .yaml || echo ""
   ```
   Store existing adapter names to prevent duplicates.

3. **Determine adapter name:**

   If $ARGUMENTS provided:
   - Convert to kebab-case for `name` field
   - Validate pattern: `^[a-z][a-z0-9-]*$`
   - Check not in existing adapters list

   If no argument or invalid:
   Use AskUserQuestion:
   - header: "Domain Name"
   - question: "What domain will this adapter cover? (e.g., 'data analysis', 'content writing', 'api design')"
   - options:
     - "Data Analysis" - Statistical analysis, ML workflows, data pipelines
     - "Content Writing" - Articles, documentation, technical writing
     - "API Design" - REST/GraphQL API development
     - "Other" - I'll describe my domain

   Process response to kebab-case.

4. **Handle name conflicts:**

   If name already exists:
   Use AskUserQuestion:
   - header: "Name Conflict"
   - question: "Adapter '{name}' already exists. How to proceed?"
   - options:
     - "Enter different name"
     - "Overwrite existing"

5. **Collect description:**

   Use AskUserQuestion:
   - header: "Description"
   - question: "Describe what this domain covers. Include typical use cases and task types. (minimum 10 characters)"
   - options:
     - "Simple description" - Basic one-liner
     - "Detailed description" - Multiple use cases

   If "Detailed": Allow freeform multi-line input.

   Validate: length >= 10 characters

6. **Set version:**

   Default to "1.0.0" - no question needed for initial creation.

7. **Store Phase 1 data:**
   ```
   ADAPTER_NAME={processed name}
   ADAPTER_DESCRIPTION={description}
   ADAPTER_VERSION="1.0.0"
   ```

## Phase 2: Init Questions (questioning section)

1. **Show example:**

   Present to user:
   ```
   ## Phase 2: Init Questions

   These questions are asked during /ptf:init to gather context
   that shapes how goals are decomposed.

   **Example from software-development adapter:**

   - category: core_value
     question: "What's the ONE thing that must work perfectly?"
     why: "Identifies critical path, shapes verification priorities"
     options_template:
       - "The {main_feature}"
       - "The {integration}"

   You'll define 1-5 questions for your domain.
   ```

2. **Ask how many questions:**

   Use AskUserQuestion:
   - header: "Question Count"
   - question: "How many init questions do you want to define?"
   - options:
     - "1-2 (minimal)" - Quick clarification only
     - "3-4 (standard)" - Balanced coverage
     - "5+ (comprehensive)" - Thorough exploration

   Set QUESTION_TARGET based on response.

3. **For each question (loop):**

   **3a. Category:**
   Use AskUserQuestion:
   - header: "Q{N} Category"
   - question: "What category is question {N}? (e.g., scope, methodology, constraints, quality)"
   - options:
     - "scope" - What's included/excluded
     - "methodology" - How work will be done
     - "constraints" - Limitations and requirements
     - "quality" - Success criteria and standards

   **3b. Question text:**
   Use AskUserQuestion:
   - header: "Q{N} Text"
   - question: "What is the question to ask users? (min 10 chars)"
   - options:
     - "What's the ONE thing that must work perfectly?"
     - "What existing patterns should we follow?"
     - "What constraints apply to this work?"
     - "Other" - Let me write a custom question

   **3c. Why:**
   Use AskUserQuestion:
   - header: "Q{N} Purpose"
   - question: "Why does this question matter for decomposition? (min 10 chars)"
   - options:
     - "Identifies critical path, shapes verification priorities"
     - "Ensures consistency with existing codebase"
     - "Defines boundaries and limitations"
     - "Other" - Let me explain

   **3d. Options template:**
   Use AskUserQuestion:
   - header: "Q{N} Options"
   - question: "Provide 2-4 example response options (use {placeholder} for dynamic parts)"
   - options:
     - "Standard options" - I'll provide typical choices
     - "Placeholder-based" - Options with {placeholders}
     - "Skip options" - Let users answer freeform

   **3e. Continue loop?**
   If N < QUESTION_TARGET:
     Continue to next question
   Else:
     Use AskUserQuestion:
     - header: "More Questions?"
     - question: "Add another init question? (you have {N} so far)"
     - options:
       - "Yes, add another"
       - "No, continue to decomposition"

4. **Store Phase 2 data:**
   ```
   INIT_QUESTIONS=[array of question objects]
   ```

## Phase 3: Decomposition Heuristics

1. **Show example:**

   Present to user:
   ```
   ## Phase 3: Decomposition Heuristics

   Heuristics define HOW to split goals into subgoals.

   **Example from software-development adapter:**

   - name: by-layer
     description: Split by architectural layer
     examples:
       - "Data layer (schemas, migrations)"
       - "Service layer (business logic)"
       - "API layer (endpoints)"
     when_to_use: "Feature spans multiple layers"

   You'll define at least 1 heuristic for your domain.
   ```

2. **Collect heuristics (loop):**

   **2a. Name:**
   Use AskUserQuestion:
   - header: "Heuristic {N}"
   - question: "What is this heuristic called? Use kebab-case (e.g., by-phase, by-component)"
   - options:
     - "by-phase" - Split by sequential phases
     - "by-component" - Split by logical components
     - "by-output" - Split by deliverable outputs
     - "Other" - Custom name

   **2b. Description:**
   Use AskUserQuestion:
   - header: "H{N} Description"
   - question: "How does this heuristic split goals? (min 10 chars)"
   - options:
     - "Split by sequential execution phases"
     - "Split by independent components"
     - "Split by distinct outputs/deliverables"
     - "Other" - Let me describe

   **2c. Examples:**
   Use AskUserQuestion:
   - header: "H{N} Examples"
   - question: "Give 2-4 example cuts this heuristic produces"
   - options:
     - "Provide examples" - I'll list concrete examples
     - "Use generic examples" - Research phase, Analysis phase, etc.

   **2d. When to use:**
   Use AskUserQuestion:
   - header: "H{N} Usage"
   - question: "When should this heuristic be used? (min 10 chars)"
   - options:
     - "When work has clear sequential phases"
     - "When components can be developed independently"
     - "When outputs are distinct and separable"
     - "Other" - Let me specify

   **2e. Continue:**
   Use AskUserQuestion:
   - header: "More Heuristics?"
   - question: "Add another heuristic? (you have {N} so far, minimum 1)"
   - options:
     - "Yes, add another"
     - "No, continue to atomicity criteria"

3. **Collect atomicity criteria:**

   Present to user:
   ```
   ## Atomicity Criteria

   These determine when a task is "small enough" (atomic).

   **Example criteria:**
   - single-output: Task produces one artifact
   - fresh-context-completable: Can finish in <30% context window
   - verifiable: Has concrete verification method
   ```

   **For each criterion (loop):**

   **3a. Criterion name:**
   Use AskUserQuestion:
   - header: "Criterion {N}"
   - question: "What is this criterion called? (e.g., single-output, time-bounded)"
   - options:
     - "single-output" - Task produces one artifact
     - "fresh-context" - Completable in fresh context
     - "verifiable" - Has concrete verification
     - "Other" - Custom criterion

   **3b. Check:**
   Use AskUserQuestion:
   - header: "C{N} Check"
   - question: "How do you verify this criterion is met? (min 10 chars)"
   - options:
     - "Task description mentions exactly one output file/artifact"
     - "Task can be explained and executed without external context"
     - "Task has at least one verification method defined"
     - "Other" - Let me describe

   **3c. Fail signal:**
   Use AskUserQuestion:
   - header: "C{N} Fail Signal"
   - question: "What indicates this criterion has failed? (min 10 chars)"
   - options:
     - "Multiple files mentioned in outputs"
     - "References to 'previous task' or external state"
     - "No clear way to verify completion"
     - "Other" - Let me describe

   **3d. Continue:**
   Use AskUserQuestion:
   - header: "More Criteria?"
   - question: "Add another criterion? (you have {N} so far, minimum 1)"
   - options:
     - "Yes, add another"
     - "No, continue"

4. **Max recursion depth:**

   Use AskUserQuestion:
   - header: "Recursion Depth"
   - question: "Maximum decomposition depth? (deeper = smaller tasks)"
   - options:
     - "3 (shallow)" - Fewer, larger tasks
     - "5 (standard)" - Balanced task sizes
     - "7 (deep)" - Many small tasks

5. **Store Phase 3 data:**
   ```
   SUBGOAL_HEURISTICS=[array]
   ATOMICITY_CRITERIA=[array]
   MAX_RECURSION_DEPTH={number}
   ```

## Phase 4: Constitution Template

1. **Show example:**

   Present to user:
   ```
   ## Phase 4: Constitution Template

   The constitution defines immutable principles for projects using this adapter.
   It's generated during /ptf:init.

   **Standard principles (always included):**
   1. Artifact Integrity - produce only declared outputs
   2. Dependency Honesty - declare all inputs
   3. Fresh Context - tasks completable by fresh agent
   4. Verification - concrete checks, not "looks good"

   You'll add domain-specific principles.
   ```

2. **Collect domain principles:**

   Use AskUserQuestion:
   - header: "Domain Principles"
   - question: "What domain-specific principles must always hold? (2-4 principles)"
   - options:
     - "Add principles one by one"
     - "Use suggested principles"
     - "Skip domain principles" - Use only standard principles

   If "Add principles one by one":
   Loop to collect principle name and description for each.

   If "Use suggested principles":
   Generate context-appropriate suggestions based on domain name.

3. **Assemble constitution template:**

   Build template with:
   - Standard 4 core principles
   - Collected domain-specific principles
   - Placeholders: {project_name}, {project_constraints}, {timestamp}

   Validate: >= 100 characters

4. **Store Phase 4 data:**
   ```
   CONSTITUTION_TEMPLATE={assembled template string}
   ```

## Phase 5: Artifact Types

1. **Show example:**

   Present to user:
   ```
   ## Phase 5: Artifact Types

   Define what outputs this domain produces.

   **Example from research adapter:**

   types:
     - name: finding
       extensions: [.md, .yaml]
       description: Individual research finding with supporting evidence
     - name: synthesis
       extensions: [.md]
       description: Integrated analysis across multiple findings

   You'll define at least 1 artifact type.
   ```

2. **Collect artifact types (loop):**

   **2a. Name:**
   Use AskUserQuestion:
   - header: "Artifact {N}"
   - question: "What is this artifact type called? (kebab-case, e.g., report, model, analysis)"
   - options:
     - "report" - Written document output
     - "model" - Data model or ML model
     - "analysis" - Analysis results
     - "config" - Configuration file
     - "Other" - Custom type

   **2b. Extensions:**
   Use AskUserQuestion:
   - header: "A{N} Extensions"
   - question: "What file extensions? (comma-separated, include dots, e.g., .md, .yaml)"
   - options:
     - ".md" - Markdown only
     - ".md, .yaml" - Markdown and YAML
     - ".json, .yaml" - Data formats
     - "Other" - Custom extensions

   Parse and validate each starts with dot.

   **2c. Description:**
   Use AskUserQuestion:
   - header: "A{N} Description"
   - question: "Describe what this artifact type represents (min 5 chars)"
   - options:
     - "Let me describe" - Custom description
     - "Use default" - Generate from name

   **2d. Verification strategy:**
   Use AskUserQuestion:
   - header: "A{N} Verification"
   - question: "How should {name} artifacts be verified?"
   - options:
     - "exists" - File exists at path
     - "exists + syntax" - File exists and parses correctly
     - "exists + contains" - File has required content
     - "exists + runs" - Command executes successfully

   If "exists + contains":
   Use AskUserQuestion:
   - header: "Contains Check"
   - question: "What content pattern should be checked?"
   - options:
     - "Has required sections" - Check for expected headings
     - "Has metadata" - Check for frontmatter/header
     - "Other" - Custom pattern

   If "exists + runs":
   Use AskUserQuestion:
   - header: "Run Command"
   - question: "What command template? (use {path} placeholder)"
   - options:
     - "validate-yaml {path}" - YAML validation
     - "jsonlint {path}" - JSON validation
     - "markdownlint {path}" - Markdown linting
     - "Other" - Custom command

   **2e. Continue:**
   Use AskUserQuestion:
   - header: "More Artifacts?"
   - question: "Add another artifact type? (you have {N} so far)"
   - options:
     - "Yes, add another"
     - "No, continue to dependencies"

3. **Store Phase 5 data:**
   ```
   ARTIFACT_TYPES=[array]
   VERIFICATION_STRATEGIES={object mapping type to strategies}
   ```

## Phase 6: Dependency Patterns

1. **Show example:**

   Present to user:
   ```
   ## Phase 6: Dependency Patterns

   Define how tasks depend on each other based on artifact flow.

   **Example from research adapter:**

   common_patterns:
     - name: finding-to-summary
       from_type: finding
       to_type: summary
       description: Summaries depend on findings they aggregate
       confidence: high

   Define at least 1 pattern showing how artifact types relate.
   ```

2. **Collect dependency patterns (loop):**

   **2a. Name:**
   Use AskUserQuestion:
   - header: "Pattern {N}"
   - question: "What is this pattern called? (kebab-case, e.g., input-to-output)"
   - options:
     - "source-to-derived" - Source produces derived output
     - "component-to-assembly" - Components combine into assembly
     - "data-to-analysis" - Data feeds into analysis
     - "Other" - Custom pattern

   **2b. From/To types:**

   Present list of artifact types defined in Phase 5.

   Use AskUserQuestion:
   - header: "P{N} Source"
   - question: "Which artifact type is the SOURCE (dependency)?"
   - options: [list of defined artifact types]

   Use AskUserQuestion:
   - header: "P{N} Target"
   - question: "Which artifact type DEPENDS on the source?"
   - options: [list of defined artifact types]

   **2c. Description:**
   Use AskUserQuestion:
   - header: "P{N} Description"
   - question: "Why does this dependency exist? (min 10 chars)"
   - options:
     - "Target aggregates/synthesizes source artifacts"
     - "Target transforms source into different format"
     - "Target requires source as input data"
     - "Other" - Let me describe

   **2d. Confidence:**
   Use AskUserQuestion:
   - header: "P{N} Confidence"
   - question: "How confident is this dependency?"
   - options:
     - "high" - Almost always true
     - "medium" - Usually true
     - "low" - Sometimes true

   **2e. Continue:**
   Use AskUserQuestion:
   - header: "More Patterns?"
   - question: "Add another dependency pattern? (you have {N} so far)"
   - options:
     - "Yes, add another"
     - "No, continue"

3. **Inference hints (optional):**

   Use AskUserQuestion:
   - header: "Inference Hints"
   - question: "Any regex patterns that hint at dependencies in content? (optional)"
   - options:
     - "Yes, add hints"
     - "No, skip to review"

   If yes:
   For each hint, collect:
   - pattern: regex pattern
   - implies: what it suggests about dependencies

4. **Store Phase 6 data:**
   ```
   DEPENDENCY_PATTERNS=[array]
   INFERENCE_HINTS=[array or empty]
   ```

## Phase 7: Review and Validate

1. **Assemble complete adapter:**

   Build YAML structure from all collected data:
   ```yaml
   # yaml-language-server: $schema=../schemas/adapter.schema.yaml
   # {ADAPTER_NAME} Domain Adapter
   # Generated by /ptf:create-adapter

   name: {ADAPTER_NAME}
   description: |
     {ADAPTER_DESCRIPTION}
   version: "{ADAPTER_VERSION}"

   questioning:
     init_questions:
       {INIT_QUESTIONS formatted}

   decomposition:
     subgoal_heuristics:
       {SUBGOAL_HEURISTICS formatted}
     atomicity_criteria:
       {ATOMICITY_CRITERIA formatted}
     max_recursion_depth: {MAX_RECURSION_DEPTH}

   constitution:
     template: |
       {CONSTITUTION_TEMPLATE}

   artifacts:
     types:
       {ARTIFACT_TYPES formatted}
     verification_strategies:
       {VERIFICATION_STRATEGIES formatted}

   dependencies:
     common_patterns:
       {DEPENDENCY_PATTERNS formatted}
     inference_hints:
       {INFERENCE_HINTS formatted}
   ```

2. **Validate against schema:**

   Check:
   - All required top-level fields present
   - name matches pattern `^[a-z][a-z0-9-]*$`
   - description >= 10 chars
   - version matches pattern `^\d+\.\d+(\.\d+)?$`
   - questioning.init_questions has >= 1 items
   - decomposition.subgoal_heuristics has >= 1 items
   - decomposition.atomicity_criteria has >= 1 items
   - constitution.template >= 100 chars with required placeholders
   - artifacts.types has >= 1 items
   - Each artifact type has verification strategy
   - dependencies.common_patterns has >= 1 items

   If validation fails, report specific issues.

3. **Present summary:**

   ```
   ## Adapter Review: {ADAPTER_NAME}

   **Name:** {ADAPTER_NAME}
   **Description:** {first 80 chars...}
   **Version:** {ADAPTER_VERSION}

   ### Sections Summary

   | Section | Items |
   |---------|-------|
   | Init Questions | {count} questions |
   | Heuristics | {count} heuristics |
   | Atomicity Criteria | {count} criteria |
   | Constitution | {char count} characters |
   | Artifact Types | {count} types |
   | Dependency Patterns | {count} patterns |

   ### Validation

   {validation results - pass/fail per section}
   ```

4. **User decision:**

   Use AskUserQuestion:
   - header: "Write Adapter?"
   - question: "Review complete. Proceed with writing the adapter file?"
   - options:
     - "Write adapter file"
     - "Edit init questions" - Go back to Phase 2
     - "Edit decomposition" - Go back to Phase 3
     - "Edit constitution" - Go back to Phase 4
     - "Edit artifacts" - Go back to Phase 5
     - "Edit dependencies" - Go back to Phase 6
     - "Cancel" - Exit without saving

   If edit: Loop back to relevant phase
   If cancel: Exit with message "Adapter creation cancelled. No files written."
   If write: Continue to Phase 8

## Phase 8: Write and Complete

1. **Write adapter file:**

   Write to `.orchestrator/adapters/{ADAPTER_NAME}.yaml` with complete YAML content.

2. **Git operations:**

   Use AskUserQuestion:
   - header: "Git Commit?"
   - question: "Stage and commit the new adapter?"
   - options:
     - "Yes, commit"
     - "No, skip"

   If yes:
   ```bash
   git add .orchestrator/adapters/{ADAPTER_NAME}.yaml
   git commit -m "$(cat <<'EOF'
   ptf: add {ADAPTER_NAME} domain adapter

   - {count} init questions
   - {count} decomposition heuristics
   - {count} artifact types

   Generated by /ptf:create-adapter

   Co-Authored-By: Claude Opus 4.5 <noreply@anthropic.com>
   EOF
   )"
   ```

3. **Completion message:**

   ```
   ---

   ## Adapter Created Successfully

   **File:** .orchestrator/adapters/{ADAPTER_NAME}.yaml

   ### Summary

   | Component | Count |
   |-----------|-------|
   | Init Questions | {count} |
   | Heuristics | {count} |
   | Atomicity Criteria | {count} |
   | Artifact Types | {count} |
   | Dependency Patterns | {count} |

   ### Next Steps

   1. **Test the adapter:**
      ```
      /ptf:init "Your goal here" --domain={ADAPTER_NAME}
      ```

   2. **Review and refine:**
      - Edit `.orchestrator/adapters/{ADAPTER_NAME}.yaml` directly for fine-tuning
      - Adjust questions if they don't elicit useful responses
      - Tune atomicity criteria if tasks are too large/small

   3. **Validate behavior:**
      - Does /ptf:init ask appropriate questions?
      - Does decomposition produce appropriately-sized tasks?
      - Do verification strategies catch real issues?

   ---
   ```

</process>

<success_criteria>
- [ ] User walked through all phases (1-8) successfully
- [ ] Adapter name is unique and valid (pattern: ^[a-z][a-z0-9-]*$)
- [ ] Description has at least 10 characters
- [ ] questioning.init_questions has at least 1 question with all required fields
- [ ] decomposition.subgoal_heuristics has at least 1 heuristic
- [ ] decomposition.atomicity_criteria has at least 1 criterion
- [ ] constitution.template >= 100 chars with {project_name}, {project_constraints}, {timestamp}
- [ ] artifacts.types has at least 1 type with extensions and description
- [ ] Each artifact type has at least 1 verification strategy
- [ ] dependencies.common_patterns has at least 1 pattern
- [ ] Full adapter validates against adapter.schema.yaml
- [ ] File written to .orchestrator/adapters/{name}.yaml
- [ ] User knows to test with `/ptf:init --domain={name}`
</success_criteria>
