# Software Requirements Specification (SRS) & Product Requirement Document (PRD)

**Project Name:** AI-Driven Automated JUnit Test Generator  
**Document Version:** 1.0  
**Target Developer:** Python / AI System Developer  

---

## 1. Executive Summary & Objective

The objective of this project is to build a standalone Python-based desktop/web UI application that automatically scans a local Java / Spring Boot project, analyzes the codebase using AI instructions defined in a `SKILL.md` file, and generates robust JUnit test cases.

The application must allow developers to select either a full codebase sweep or a diff-based sweep (uncommitted/committed branch changes), display real-time code coverage metrics (Current vs. Target vs. Post-Generation), and strictly ensure that the original application codebase remains unmodified and fully buildable.

---

## 2. High-Level Architecture & Tech Stack

* **Programming Language:** Python 3.10+
* **User Interface:** Streamlit, Gradio, or PyQt/CustomTkinter (Web-based local UI preferred for rapid rendering and clean visual layout)
* **Build Tool Integration:** Maven (`mvn clean test`, `jacoco:report`)
* **Version Control Integration:** Git (`GitPython` or subprocess CLI commands)
* **LLM / AI Engine Integration:** LangChain / LlamaIndex or Direct OpenAI/Anthropic/Ollama API wrappers executing instructions defined in `SKILL.md`.

---

## 3. System Workflow & Requirements

```
[ User Input: Project Path & Mode ] 
            │
            ▼
[ System Validation & Baseline Coverage Check ]
    └── Runs `mvn test` + JaCoCo to compute Current Code Coverage
            │
            ▼
[ Scope Identification ]
    ├── Mode A (Full Base): Parse all target `.java` source files in `src/main/java`
    └── Mode B (Dev Changes): Query Git for uncommitted/committed branch diffs
            │
            ▼
[ AI Test Case Generation (SKILL.md Driven) ]
    ├── Prompt LLM with source code + SKILL.md guidelines
    └── Save generated tests into `src/test/java`
            │
            ▼
[ Verification & Build Execution ]
    └── Execute `mvn clean test`
            ├── If build fails / tests fail: Run auto-repair cycle (re-prompt LLM with error logs)
            └── If build succeeds: Compute final Post-Generation Code Coverage
            │
            ▼
[ UI Update ] -> Render metrics & report
```

---

## 4. Functional Specifications

### 4.1. User Interface (UI) Requirements
1. **Local Project Path Input:**
   * Text field and directory picker button allowing the user to specify the absolute path to the local Java project root (containing `pom.xml`).
   * **Validation:** Must check for the existence of `pom.xml` and a valid `.git` directory.

2. **Execution Mode Toggle:**
   * **Full Codebase Mode:** Scans all concrete Java classes inside `src/main/java/`.
   * **Developer Changes Mode:** Identifies modified/added `.java` files by checking:
     * Uncommitted changes (`git status`, `git diff`)
     * Committed changes relative to the base branch (e.g., comparing current branch to `main`/`master`).

3. **Coverage Dashboard / Metrics Display:**
   * **Current Code Coverage ($C_{\text{current}}$):** Baseline code coverage before test generation.
   * **Target Code Coverage ($C_{\text{target}}$):** Defaults to **90%** (configurable via input box/slider).
   * **Post-Tool Code Coverage ($C_{\text{final}}$):** Coverage metric achieved after running the generated tests.
   * Progress indicator / live status logging console (e.g., "Calculating baseline coverage...", "Generating tests for UserService.java...", "Running Maven verification...").

---

### 4.2. Core Engine & Execution Requirements

#### A. Baseline Coverage Assessment
* On initialization, the tool executes Maven with JaCoCo plugin (`mvn test jacoco:report` or `mvn org.jacoco:jacoco-maven-plugin:prepare-agent test org.jacoco:jacoco-maven-plugin:report`).
* Parses `target/site/jacoco/jacoco.xml` or `target/site/jacoco/index.html` to extract overall line/instruction coverage percentage.

#### B. Test Generation via `SKILL.md`
* The engine reads a root or configurable `SKILL.md` file.
* For each target class identified in the selected scope:
  * Reads the source code file.
  * Formats a prompt combining the contents of `SKILL.md`, target class context, dependent mock requirements, and Spring Boot annotations context.
  * Calls the LLM to write corresponding unit tests using standard JUnit 5 / Mockito conventions.
* Test files must be written exclusively inside `src/test/java/` (following standard package structures).

#### C. STRICT RULE: Zero Source Code Modification
* The tool **MUST NOT** modify, reformat, edit, or delete any file under `src/main/` or any non-test configuration/resource files.
* File system locks/checks must ensure only files in `src/test/java/` are created or modified.

#### D. Maven Verification & Auto-Repair Loop
* After generating test classes, run `mvn clean test`.
* If compilation fails or tests fail:
  1. Capture Maven output / failure stack traces.
  2. Feed the stack trace back to the LLM along with the generated test class and target source file.
  3. Re-generate/fix the test class (up to a maximum retry limit of 3 attempts per class).
* Ensure `mvn clean test` exits with code `0`.

---

## 5. `SKILL.md` Specification & Instructions

The developer must implement `SKILL.md` parser/prompt-injector according to the following template structure:

```markdown
# JUnit Test Case Generation Skill Guidelines

## Objective
Generate complete, robust, maintainable JUnit 5 test cases for Java/Spring Boot applications to achieve or exceed the target code coverage (default 90%).

## Strict Guardrails
1. DO NOT modify any application code under `src/main/`.
2. ONLY generate or edit test classes inside `src/test/java/`.
3. Generated tests MUST compile and pass cleanly via `mvn clean test`.

## Technical Conventions
- **Frameworks:** JUnit 5 (Jupiter), Mockito (`@Mock`, `@InjectMocks`, `@ExtendWith(MockitoExtension.class)` or `@WebMvcTest` / `@SpringBootTest` where appropriate).
- **Structure:** Follow standard Arrange-Act-Assert (AAA) pattern.
- **Coverage Target:** Focus heavily on boundary conditions, edge cases, null checks, exception paths, and conditional branches (if/else/switch) to maximize JaCoCo branch & line coverage.
- **Spring Context:** Use lightweight slice testing (`@WebMvcTest`, `@DataJpaTest`) over full `@SpringBootTest` whenever possible to maintain fast test execution times.

## Execution Scope Instructions
- **Full Mode:** Process all eligible concrete classes in `src/main/java/`.
- **Developer Changes Mode:** Focus only on target classes that match git diffs (uncommitted or branch commits). Ensure new or updated methods in those target classes have comprehensive test coverage.
```

---

## 6. Project Directory Layout (Python Tool)

```text
ai-junit-generator/
├── app.py                     # UI Application entrypoint (Streamlit/PyQt/Gradio)
├── config.py                  # Configuration & default settings
├── SKILL.md                   # Core AI prompt/rule specifications
├── requirements.txt           # Python dependencies
├── core/
│   ├── __init__.py
│   ├── git_handler.py         # Git diff and uncommitted file detection
│   ├── maven_runner.py        # Executing Maven builds and parsing JaCoCo XML reports
│   ├── llm_client.py          # AI interaction module (injecting SKILL.md rules)
│   └── test_generator.py      # Orchestrator & auto-repair loop
└── templates/                 # UI assets / CSS styles
```

---

## 7. Acceptance Criteria & Definition of Done (DoD)

1. **Path Selection:** User provides a valid local path to a Java Spring Boot repo; invalid paths show proper UI error messages.
2. **Coverage Metrics UI:** UI clearly displays:
   * Current Coverage %
   * Target Coverage % (Editable/Default 90%)
   * Post-Generation Coverage %
3. **Toggle Functionality:**
   * Selecting **"Full Codebase"** generates tests for all classes under `src/main/java`.
   * Selecting **"Developer Changes"** strictly isolates target classes with git diffs (uncommitted or committed vs base branch).
4. **Source Integrity:** Running the tool leaves git diff status for `src/main/` completely clean.
5. **Successful Build:** Final execution of `mvn clean test` on the target Java project completes with zero failures and build success status.