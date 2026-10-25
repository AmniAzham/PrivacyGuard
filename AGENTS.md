# PrivacyGuard — Persistent Engineering Rules

## 1. Project Definition

PrivacyGuard is a local-first desktop application and Python library that helps users inspect, sanitize, and verify documents before sharing them externally or pasting their contents into AI tools.

Core workflow:

**Intake → Extract → Detect → Review and Policy → Sanitize → Independently Verify → Export**

The product is intended to help reduce accidental disclosure of confidential information. It must never promise that automated sanitization can guarantee complete privacy or eliminate every possible disclosure.

## 2. Non-Negotiable Security Invariants

### 2.1 Local-first processing

1. The document-processing path must not send document contents, extracted text, findings, secrets, or pseudonymization mappings to external services.
2. No telemetry, analytics, crash reporting, cloud fallback, or automatic update checks.
3. Core processing must work without an account, cloud service, or network connection.
4. Any future network-dependent feature must be a separate, documented, explicitly opt-in module.
5. Never silently fall back to remote processing.
6. Do not claim that the application is completely offline solely because unit tests pass. Distinguish application-level network controls from operating-system-level network isolation.
7. Development-time dependency installation is separate from runtime document processing. Document these boundaries clearly.

### 2.2 Fail securely

1. Unsupported, corrupt, encrypted, oversized, malformed, or suspicious input must be rejected or routed to an explicitly limited inspection-only workflow.
2. An exception, skipped critical check, timeout, incomplete transformation, or failed verification must never produce an eligible-for-export verification result.
3. The default export eligibility state is `blocked`.
4. Only an explicit, tested path in which every required critical check passes may set export eligibility to `eligible`.
5. Never silently downgrade a policy or skip a required check.
6. Preserve the source document. Produce a separate output file and refuse to overwrite the source.
7. Treat filenames, file metadata, document content, archive entries, and configuration files as untrusted input.

### 2.3 No false assurance

1. Absence of findings does not prove that a document is safe.
2. Passing tests proves behavior only for the tested cases and environments.
3. Never describe the application as compliant, certified, enterprise-ready, or production-secure without appropriate evidence and independent validation.
4. Clearly report unsupported content, skipped checks, residual risks, and uncertainty.
5. Never claim universal detection or complete sanitization.
6. A successful verification status must always be accompanied by a clear residual-risk statement.

### 2.4 Sensitive-data handling

1. Never log or persist raw sensitive values by default.
2. This includes application logs, audit events, exception messages, terminal output, temporary files, crash reports, test output, and debugging traces.
3. Mask sensitive findings in the UI and CLI by default.
4. Raw values may be retained in memory only when required for processing and verification, with scope and lifetime minimized.
5. Do not serialize findings containing raw values unless an explicitly reviewed, documented feature requires it.
6. Use synthetic data only in demonstrations, fixtures, and automated tests.
7. Synthetic secrets must be clearly fake. Configure secret scanners to exclude only explicitly identified fixtures, never entire source directories.

### 2.5 Independent verification

1. Detection, sanitization, and verification must be separate modules.
2. Verification must reopen and re-parse the actual output file from disk.
3. Do not treat the detector's in-memory state, transformation list, or successful write operation as proof that sanitization succeeded.
4. Use format-specific verification and additional techniques to reduce shared failure modes between detection and verification.
5. Where appropriate, compare the original sensitive values against the output using exact and normalized matching.
6. Check consistency of pseudonymized entities where that feature is supported.
7. A skipped, failed, or unsupported critical check must block verified export.
8. Passing verification means the declared checks passed, not that every possible sensitive disclosure has been eliminated.

### 2.6 Repository safety

1. Inspect `git status` before making changes.
2. Never discard, overwrite, or revert user work without explicit authorization.
3. Work on a feature branch when appropriate.
4. Make small, logical commits when the environment and user workflow permit.
5. Never force-push.
6. Before committing, inspect the diff for accidental secrets, sensitive fixtures, unrelated changes, and unsupported security claims.
7. Do not commit unless the repository state and intended changes have been reviewed.

## 3. Technology Stack

Use this stack unless the existing repository provides a compelling reason to deviate. Document any deviation and its trade-offs in `docs/ARCHITECTURE.md`.

- Python 3.12 or newer.
- `uv` for dependency and environment management.
- `pyproject.toml` and a committed dependency lockfile.
- Package layout under `src/privacyguard/`.
- `pytest` for automated tests.
- `hypothesis` for property-based and fuzz-oriented tests.
- `ruff` for linting and formatting.
- `mypy --strict` for static type checking under `src/`.
- `defusedxml` for parsing untrusted XML.
- `cryptography` for cryptographic operations if an optional mapping vault is implemented.
- PySide6 for the desktop interface in M5.
- `pytest-qt` for relevant desktop UI tests.
- `pip-audit` or an equivalent dependency vulnerability checker.
- CycloneDX-compatible SBOM generation during hardening.

Suggested package layout:

    src/privacyguard/
        __init__.py
        models.py
        intake/
        formats/
        detect/
        policy/
        sanitize/
        verify/
        audit/
        config/
        cli/
        ui/

Keep dependencies minimal. Before introducing a dependency:

1. Explain why it is needed.
2. Review its maintenance status, licensing, security history, and transitive dependencies where feasible.
3. Pin and lock dependencies.
4. Avoid unnecessary network-capable services in the processing path.
5. Never install or execute arbitrary generated or downloaded code without review.

For untrusted OOXML input, use hardened XML parsing. Do not use unsafe XML parsing configurations. Avoid bare `xml.etree` or `lxml` for untrusted content unless the chosen configuration and parser behavior have been explicitly reviewed and tested.

Do not invent or hand-roll cryptographic primitives.

## 4. Core Data Models

Define stable, typed models in `src/privacyguard/models.py`.

### Finding

Include:

- `id`
- `category`
- `severity`
- `confidence`
- `detector_id`
- `location`
- `span`
- `masked_preview`
- `explanation`
- `recommended_action`

Keep raw sensitive values in memory only when necessary. Never serialize them by default.

### Policy

Include:

- `id`
- `version`
- Per-category action: `redact`, `placeholder`, `pseudonymize`, or `keep`
- Metadata handling rules
- Processing limits
- Required verification checks
- Explicit handling of unsupported content

Validate policies before use. Unsafe or ambiguous configurations must not silently weaken security requirements.

### SanitizationPlan

An ordered, reviewable set of proposed transformations and user-approved decisions.

The plan must make it possible to distinguish proposed actions from actions actually applied.

### VerificationReport

Include:

- Checks performed
- Checks passed
- Checks failed
- Checks skipped
- Unsupported content
- Residual-risk notes
- Verification method
- `export_eligibility`: `eligible`, `blocked`, or `inspect_only`

Default eligibility must be `blocked`. Only a tested, explicit path satisfying every required critical check may change it to `eligible`.

### AuditEvent

Include only necessary operational metadata, such as:

- Operation ID
- Timestamp
- Format
- File-size category
- Policy ID and version
- Finding counts by category
- Actions applied
- Verification outcome
- Error category
- Application version

Never store document contents, extracted text, raw sensitive values, pseudonymization mappings, or full file paths by default.

## 5. Required Commands

Create and maintain these commands as early as practical:

    uv sync
    uv run ruff check . 
    uv run ruff format --check .
    uv run mypy src
    uv run pytest -q
    uv run pytest -q -m security
    uv run pytest -q -m network
    uv run privacyguard --help

Tests must be marked consistently. If a command cannot run because of an environment limitation, state the exact limitation and do not claim success.

Run dependency auditing and secret scanning where feasible.

## 6. Documentation Rules

Create and maintain:

- `README.md`
- `SECURITY.md`
- `docs/PRODUCT_REQUIREMENTS.md`
- `docs/ARCHITECTURE.md`
- `docs/THREAT_MODEL.md`
- `docs/SECURITY_CONTROLS.md`
- `docs/FORMAT_SUPPORT.md`
- `docs/TEST_STRATEGY.md`
- `docs/ROADMAP.md`

Keep documentation synchronized with the actual code.

Label features accurately:

- **Implemented**
- **Implemented — partially validated**
- **Planned**
- **Unsupported**

Do not document a planned feature as implemented.

For each supported format, document extraction scope, sanitizable content, metadata handling, unsupported features, fidelity limitations, verification method, format-specific risks, and test coverage.

## 7. Engineering Workflow

Use this loop for each vertical slice:

1. Inspect the current implementation and relevant tests.
2. Make the smallest complete design decision.
3. Record consequential architectural choices.
4. Implement the vertical slice.
5. Add positive and negative tests.
6. Run relevant checks and inspect their actual output.
7. Fix discovered defects.
8. Update documentation.
9. Review the diff.
10. Commit a small logical change if appropriate.
11. Report actual results and remaining work.

Do not attempt to implement an entire large milestone in one enormous edit.

Keep the end-to-end path working as the product grows. Do not weaken acceptance criteria to make a milestone appear complete.

If a milestone is too large for the current session, finish the current vertical slice, test it, document remaining work, and stop at a reproducible checkpoint.

Do not ask for approval on routine implementation choices. Ask only when a consequential decision cannot safely be resolved from the repository or documented requirements.

## 8. Required Milestone Report

At the end of every milestone, report:

1. What changed and which files were modified.
2. Exact commands executed and actual outcomes.
3. Test totals and failures.
4. Security controls added.
5. Known defects and unsupported features.
6. Network behavior and the scope of its validation.
7. Features that are implemented but not fully validated.
8. Next recommended task.

Never claim a feature, fix, test, or security guarantee works without having run the relevant checks and inspected the results.
