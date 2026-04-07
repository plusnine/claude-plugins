# Task Prompt Format

## Template

```markdown
# Task {nn}: {task name}

## Context
{which spec-review requirement this addresses}
Prerequisites: {dependent tasks, or "none"}

## What to implement
{business-level description — what the feature does, not how to implement it}

## Investigation hints
{Spec-text-based pointers only — no code investigation at this stage. Describe what to look for in terms of business concepts, not file names or class names. Examples: "find where the payment flow is initiated", "find the screen that handles user onboarding". task-implement will verify and refine these.}

## Done Criteria
- [ ] {business-level completion criterion}
```
