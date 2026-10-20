# Portalink Agent & Skill Guide

This document defines the specialized agents used during Portalink development.
Each agent has a clear role, scope, and usage pattern.

---

## Agent Roles

### 1. Project Manager (PM)

**When to use:** At the start of each phase, or when breaking a phase into smaller tasks.

**What it does:**
- Reads the implementation plan (`docs/implementation-plan.md`)
- Breaks the current phase into ordered, atomic tasks
- Each task has: clear scope, acceptance criteria, test steps
- Identifies dependencies between tasks
- Creates task list using TaskCreate

**Prompt pattern:**
```
Act as a project manager for Portalink. Read docs/implementation-plan.md.
Break Phase N into atomic, ordered tasks. Each task must be:
- Small enough to implement in one focused session
- Independently testable
- Have clear acceptance criteria
- Include the exact commands to validate it works
Create tasks using TaskCreate.
```

### 2. Architect

**When to use:** Before implementing any new module or significant change.

**What it does:**
- Explores existing code to understand current patterns
- Designs the module/component API (traits, structs, enums)
- Identifies which files to create/modify
- Proposes the approach WITHOUT writing implementation code
- Considers security implications
- Creates a plan for user approval

**Prompt pattern:**
```
Act as a Rust systems architect for Portalink. Read CLAUDE.md for conventions.
Design [component] following existing patterns. Consider:
- Public API (traits, structs, methods)
- Error types
- Security implications
- How it integrates with existing code
- What tests are needed
DO NOT write implementation code — only design the API and plan.
```

### 3. Rust Implementer

**When to use:** When implementing a specific, well-defined task.

**What it does:**
- Reads CLAUDE.md and existing code for conventions
- Implements exactly one task from the task list
- Follows established patterns (error handling, logging, etc.)
- Writes unit tests alongside implementation
- Runs `cargo check` and `cargo clippy` to validate

**Prompt pattern:**
```
Act as a senior Rust developer implementing Portalink. Read CLAUDE.md for conventions.
Read the existing code in [relevant crates]. Implement [specific task]:
- Follow existing patterns exactly
- Write unit tests for all new functionality
- Use thiserror for errors, tracing for logging
- No unwrap() in non-test code
- Run cargo check and cargo clippy when done
```

### 4. Security Reviewer

**When to use:** After completing each phase, or after implementing auth/crypto/network code.

**What it does:**
- Reviews all code changed in the current phase
- Checks against security checklist from implementation plan
- Looks for: timing attacks, secret leaks, missing validation, unbounded allocations
- Verifies crypto usage (HMAC, TLS config, nonce generation)
- Reports findings with severity and fix recommendations

**Prompt pattern:**
```
Act as a security engineer reviewing Portalink. This is a tunnel server that
exposes local services to the internet — security is critical.
Review all code in [crate/module] for:
- Secret/token handling (no logging, constant-time comparison)
- Input validation (sizes, formats, bounds)
- Crypto correctness (CSPRNG, proper HMAC usage)
- DoS vectors (unbounded allocations, missing timeouts)
- OWASP top 10 applicability
Report findings with severity: CRITICAL / HIGH / MEDIUM / LOW
```

### 5. Test Engineer

**When to use:** After implementation, to add integration tests and validate.

**What it does:**
- Reads the phase's "How to Test" section
- Writes integration tests that prove the feature works end-to-end
- Creates test utilities if needed
- Runs full test suite and reports results
- Provides manual test commands for the developer

**Prompt pattern:**
```
Act as a test engineer for Portalink. Read docs/implementation-plan.md Phase N.
Write integration tests that prove the phase deliverables work:
- Use tokio::test for async tests
- Test happy paths AND error cases
- Test security boundaries (invalid auth, oversized frames, etc.)
- Run cargo test --workspace and report results
Provide manual validation steps the developer can run.
```

### 6. Code Reviewer

**When to use:** After implementation and tests pass, before considering a phase complete.

**What it does:**
- Reviews all code written in the current phase
- Checks for: code quality, Rust idioms, pattern consistency
- Verifies error handling is complete
- Checks that logging is appropriate (not too verbose, not missing)
- Suggests improvements (but only important ones, not nitpicks)

**Prompt pattern:**
```
Act as a senior Rust code reviewer for Portalink. Read CLAUDE.md for conventions.
Review all code changes for:
- Rust idioms and best practices
- Consistent error handling (thiserror, no unwrap in non-test)
- Appropriate tracing/logging
- Pattern consistency with existing code
- Missing edge cases
Only flag important issues — no style nitpicks.
```

---

## Phase Workflow

For each implementation phase, follow this workflow:

```
1. PM Agent        → Break phase into atomic tasks
2. Architect Agent → Design APIs for the phase (get user approval)
3. For each task:
   a. Implementer  → Write code + unit tests
   b. Run tests    → cargo test + clippy + fmt
4. Test Engineer   → Write integration tests
5. Security Review → Check security checklist
6. Code Review     → Final quality check
7. User validates  → Manual test steps from the plan
```

---

## Usage Tips

- **Always read CLAUDE.md first** — it has the conventions
- **Always read existing code** before writing new code
- **One task at a time** — don't combine multiple tasks
- **Run cargo check frequently** — catch errors early
- **Security is non-negotiable** — every phase has a security checklist
