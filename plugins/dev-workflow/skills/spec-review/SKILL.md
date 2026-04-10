---
name: spec-review
description: Review specifications for completeness before implementation. Use when reviewing specs, requirements, or implementation plans to ensure no gaps, overlaps, or ambiguities.
version: 0.1.0
model: sonnet
---

## When to Use

Used to review specifications for completeness before implementation begins.

## Scope Constraint

The review scope is strictly limited to the spec document itself.
- Do NOT read or reference existing code to resolve issues
- Do NOT auto-resolve issues by inferring from implementation
- Surface all issues as-is; resolution is the human's responsibility

## Checklist

### 1. Source Verification
- If a primary source (docs/, design doc, etc.) is referenced, read it and verify ALL spec claims against it
- Surface discrepancies before proceeding

### 2. Coverage
- Before flagging information as missing, verify it is not already covered in another section of the same document
- All user operation paths covered (happy / error / cancel)
- All required processing per path (logging, state update, UI update)
- All spec parameters and requirements included
- Dependencies and preconditions identified (external APIs, other features, required state)
- Acceptance criteria defined (how to verify correctness)

### 3. Quality
- No gaps, overlaps, or ambiguities
- Boundary conditions and edge cases considered
- Out-of-scope items explicitly stated to prevent implicit expectations
