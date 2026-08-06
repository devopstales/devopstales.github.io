---
title: OpenSpec: The Spec-Driven Development Revolution
url: https://devopstales.github.io/ai/openspec-spec-driven-development-revolution/
date: 2026-04-07
---


In the age of AI-assisted coding, one problem keeps surfacing: **AI doesn't follow instructions**. OpenSpec changes that by introducing **spec-driven development**—a disciplined workflow where specifications are written *before* code is generated.

<!--more-->

## What is OpenSpec?

**OpenSpec** is a free, open-source CLI tool that brings **Spec-Driven Development (SDD)** to AI coding assistants like Cursor, Claude Code, Cline, Kilo Code, and Codex. It ensures AI agents strictly follow your requirements by generating detailed markdown specification documents *before* any code is written.

Think of OpenSpec as the **blueprint phase** for software construction. Just as architects don't start pouring concrete without plans, developers shouldn't ask AI to code without specs.

## The Problem OpenSpec Solves

Without OpenSpec, AI coding sessions typically look like this:

```
Developer: "Add user authentication"
AI:       *generates some code*
Developer: "Hmm, also add password reset"
AI:       *generates more code*
Developer: "Wait, login is broken now"
AI:       *patches things*
Developer: "We need OAuth too..."
AI:       *adds OAuth, breaks something else*

Result: Spaghetti code, inconsistent patterns, hidden bugs
```

OpenSpec replaces this chaos with a **structured, auditable workflow**.

## The OpenSpec Pipeline

Here's the complete OpenSpec pipeline visualized:

```
┌─────────────────────────────────────────────────────────────────────┐
│                        OPENSPEC PIPELINE                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌──────────┐     ┌──────────┐     ┌──────────┐     ┌──────────┐  │
│  │   INIT   │────▶│ PROPOSAL │────▶│  APPLY   │────▶│ ARCHIVE  │  │
│  │          │     │          │     │          │     │          │  │
│  │ Setup    │     │ Draft    │     │ Execute  │     │ Finalize │  │
│  │ Project  │     │ Spec     │     │ Tasks    │     │ & Update │  │
│  └──────────┘     └──────────┘     └──────────┘     └──────────┘  │
│       │               │               │               │            │
│       ▼               ▼               ▼               ▼            │
│  project.md       proposal.md    tasks.md       archive/          │
│  specs/           tasks.md       code changes   updated specs     │
│                   design.md      validated      cleaned up        │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

## Step-by-Step Breakdown

### Step 1: INIT — Initialize the Specification

```
┌──────────────────────────────────────────────┐
│              INIT PHASE                       │
├──────────────────────────────────────────────┤
│                                              │
│   $ openspec init                            │
│                                              │
│   Creates:                                   │
│   ┌─ openspec/                               │
│   │  ├── project.md    ← Project context     │
│   │  ├── specs/        ← Current capabilities│
│   │  └── changes/      ← Active proposals    │
│   │                                          │
│   project.md defines:                        │
│   • Tech stack & conventions                 │
│   • Code style & patterns                    │
│   • Testing strategy                         │
│   • Architectural constraints                │
│                                              │
└──────────────────────────────────────────────┘
```

The `init` command sets up your project's **source of truth**. This file tells AI assistants *how* your project is structured and *what rules* they must follow.

### Step 2: PROPOSAL — Draft the Specification

```
┌──────────────────────────────────────────────────────┐
│                  PROPOSAL PHASE                       │
├──────────────────────────────────────────────────────┤
│                                                      │
│   $ openspec propose add-user-auth                   │
│                                                      │
│   Generates:                                         │
│   ┌─ changes/add-user-auth/                          │
│   │  ├── proposal.md   ← Why & what                 │
│   │  ├── tasks.md      ← Step-by-step checklist     │
│   │  ├── design.md     ← Technical decisions        │
│   │  └── specs/                                       │
│   │      └── user-auth/                               │
│   │          └── spec.md ← Requirement deltas        │
│   │                                                  │
│   Validate:                                          │
│   $ openspec validate add-user-auth --strict         │
│                                                      │
│   Spec delta format:                                 │
│   ## ADDED Requirements                              │
│   ### Requirement: User Login                        │
│   The system SHALL authenticate users via email      │
│   #### Scenario: Valid credentials                    │
│   WHEN user provides valid email and password        │
│   THEN system grants access and creates session      │
│                                                      │
└──────────────────────────────────────────────────────┘
```

This is where you **define the change** before any code is written. Every requirement uses **normative language** (`SHALL`/`MUST`) and includes testable `Scenario` blocks.

### Step 3: APPLY — Execute the Specification

```
┌──────────────────────────────────────────────────────┐
│                   APPLY PHASE                         │
├──────────────────────────────────────────────────────┤
│                                                      │
│   AI Assistant reads the proposal and:               │
│                                                      │
│   ┌────────────────────────────────────────────┐    │
│   │  tasks.md Checklist:                       │    │
│   │                                            │    │
│   │  ☐ Create User model with email/password   │    │
│   │  ☐ Implement JWT token generation          │    │
│   │  ☐ Add login endpoint POST /api/auth/login │    │
│   │  ☐ Write unit tests for auth service       │    │
│   │  ☐ Add integration tests for login flow    │    │
│   │  ☐ Update API documentation                │    │
│   │                                            │    │
│   └────────────────────────────────────────────┘    │
│                                                      │
│   AI follows spec EXACTLY:                           │
│   • Implements each task in order                     │
│   • Matches defined scenarios                         │
│   • Respects project conventions                      │
│   • No hallucination, no deviation                    │
│                                                      │
│   Progress tracked:                                  │
│   $ openspec show add-user-auth                      │
│                                                      │
└──────────────────────────────────────────────────────┘
```

During Apply, the AI coding assistant **executes tasks sequentially**, checking them off as completed. The spec acts as guardrails—preventing scope creep and ensuring consistency.

### Step 4: ARCHIVE — Finalize and Update

```
┌──────────────────────────────────────────────────────┐
│                  ARCHIVE PHASE                        │
├──────────────────────────────────────────────────────┤
│                                                      │
│   After deployment & verification:                   │
│                                                      │
│   $ openspec archive add-user-auth --yes             │
│                                                      │
│   What happens:                                      │
│   ┌────────────────────────────────────────────┐    │
│   │  1. changes/add-user-auth/ → archive/      │    │
│   │  2. specs/ updated with new capabilities   │    │
│   │  3. project state synchronized             │    │
│   │  4. Spec deltas merged into main specs     │    │
│   └────────────────────────────────────────────┘    │
│                                                      │
│   Result:                                            │
│   ┌─ openspec/                                       │
│   │  ├── project.md                                  │
│   │  ├── specs/                                      │
│   │  │   └── user-auth/ ← NEW capability added      │
│   │  │       ├── spec.md                             │
│   │  │       └── design.md                           │
│   │  ├── changes/  ← Empty, ready for next          │
│   │  └── archive/                                    │
│   │      └── add-user-auth/ ← Historical record     │
│   │                                                  │
│   System truth is now UP TO DATE                     │
│                                                      │
└──────────────────────────────────────────────────────┘
```

Archive is the **cleanup and synchronization** step. Completed changes are moved to history, and the main specification files are updated to reflect the new system state.

## Complete Workflow Visualization

```
                         ┌─────────────────┐
                         │   DEVELOPER     │
                         │   defines need  │
                         └────────┬────────┘
                                  │
                                  ▼
                    ┌─────────────────────────┐
                    │   1. INIT               │
                    │   openspec init         │
                    │   Setup project.md      │
                    └────────────┬────────────┘
                                 │
                                 ▼
                    ┌─────────────────────────┐
                    │   2. PROPOSE            │
                    │   openspec propose      │
                    │   Create spec delta     │
                    │   define requirements   │
                    │   write scenarios       │
                    └────────────┬────────────┘
                                 │
                                 ▼
                    ┌─────────────────────────┐
                    │   VALIDATE              │
                    │   openspec validate     │
                    │   --strict              │◀──┐
                    └────────────┬────────────┘   │
                                 │                │
                            ✓ pass?               │
                                 │                │
                       ┌─────┴──────┐             │
                       │            │             │
                      YES          NO             │
                       │            │             │
                       ▼            └─────────────┘
                    ┌─────────────────────────┐
                    │   3. APPLY              │
                    │   AI reads spec         │
                    │   Executes tasks.md     │
                    │   line by line          │
                    └────────────┬────────────┘
                                 │
                                 ▼
                    ┌─────────────────────────┐
                    │   TEST & VERIFY         │
                    │   Run tests             │
                    │   Deploy & verify       │
                    └────────────┬────────────┘
                                 │
                                 ▼
                    ┌─────────────────────────┐
                    │   4. ARCHIVE            │
                    │   openspec archive      │
                    │   Update specs          │
                    │   Move to archive/      │
                    └────────────┬────────────┘
                                 │
                                 ▼
                    ┌─────────────────────────┐
                    │   SYSTEM TRUTH          │
                    │   Updated & current     │
                    │   Ready for next change │
                    └─────────────────────────┘
```

## Directory Structure

OpenSpec maintains a clean, version-controllable structure:

```
openspec/
├── project.md              # Project context, conventions, constraints
├── specs/                  # Current system capabilities ("source of truth")
│   └── user-auth/
│       ├── spec.md         # Requirements with scenarios
│       └── design.md       # Technical decisions
├── changes/                # Active proposals
│   └── add-user-auth/
│       ├── proposal.md     # Why this change is needed
│       ├── tasks.md        # Implementation checklist
│       ├── design.md       # Technical approach
│       └── specs/
│           └── user-auth/
│               └── spec.md # Delta: ADDED/MODIFIED/REMOVED requirements
└── archive/                # Completed changes (historical record)
    └── add-user-auth/      # Preserved for audit & reference
```

## Key Commands at a Glance

```
┌─────────────────────────────────────────────────────────────┐
│                    OPEN-SPEC CHEAT SHEET                    │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Initialize           $ openspec init                       │
│  Update spec          $ openspec update                     │
│  List changes         $ openspec list                       │
│  List specs           $ openspec list --specs               │
│  Show item            $ openspec show [change-id]           │
│  Diff changes         $ openspec diff [change-id]           │
│  Validate strict      $ openspec validate [id] --strict     │
│  Archive completed    $ openspec archive [id] --yes         │
│                                                             │
│  Change ID format: kebab-case, verb-led                     │
│  Examples: add-japanese-localization                        │
│           update-auth-middleware                            │
│           remove-legacy-api                                 │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

## Spec Delta Syntax

OpenSpec uses a precise format for defining changes:

```markdown
## ADDED Requirements

### Requirement: User Registration
The system SHALL allow users to register with email and password.

#### Scenario: Successful registration
WHEN user submits valid registration form
AND email is not already registered
AND password meets complexity requirements
THEN system creates new user account
AND system sends verification email
AND system returns success response

#### Scenario: Duplicate email
WHEN user submits registration form
AND email is already registered
THEN system returns error "Email already in use"
AND no new account is created

## MODIFIED Requirements

### Requirement: Password Policy
[REQUIREMENT] The system SHALL enforce minimum password complexity.
- Password MUST be at least 12 characters
- Password MUST contain uppercase and lowercase letters
- Password MUST contain at least one number
- Password MUST contain at least one special character

## REMOVED Requirements

### Requirement: Legacy Token Format
The system SHALL NO LONGER support JWT tokens with HS256 algorithm.
[REMOVED] All tokens MUST use RS256 algorithm instead
```

## When to Use (and Skip) OpenSpec

```
┌─────────────────────────────────────────────┐
│           WHEN TO USE OPENSPEC              │
├─────────────────────────────────────────────┤
│                                             │
│  ✓ New features (0 → 1)                     │
│  ✓ Major refactoring (1 → n)                │
│  ✓ Architectural changes                    │
│  ✓ Multi-step implementations               │
│  ✓ Team collaboration & code review         │
│  ✓ Compliance & audit requirements          │
│                                             │
│  ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─    │
│                                             │
│  ✗ Simple bug fixes                         │
│  ✗ Typos and formatting                     │
│  ✗ Non-breaking config changes              │
│  ✗ One-line patches                         │
│                                             │
└─────────────────────────────────────────────┘
```

## Why OpenSpec Works

1. **Single Source of Truth**: `project.md` and `specs/` define exactly what exists and how it works
2. **AI-Readable Format**: Markdown specs are designed for AI assistants to parse and follow
3. **Strict Validation**: `openspec validate --strict` catches missing scenarios and formatting errors
4. **Audit Trail**: Every change is documented, validated, and archived
5. **No Context Rot**: AI always reads the current state—no assumptions, no hallucination

## Conclusion

OpenSpec represents a paradigm shift from **vibe coding** to **spec-driven development**. By requiring specifications before implementation, it ensures AI-generated code is consistent, auditable, and aligned with your project's architecture.

The pipeline is simple: **Init → Propose → Validate → Apply → Archive**. But the impact is profound: fewer bugs, cleaner code, and AI that actually follows your instructions.

In a world where AI can generate code in seconds, OpenSpec ensures that code is **correct by design**.

