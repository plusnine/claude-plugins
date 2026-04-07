# review.md Output Format

## Structure

### Header

```
# Spec Review — {JIRA ID or kebab-case name}

Spec: {URL}
Review date: {YYYY-MM-DD}
Note: {optional — out-of-scope items, caveats}
```

---

### AI Summary

Place immediately after the header. English only. Contains the minimum information needed for dev-planning and subsequent spec-review rounds to orient before reading details.

```
## AI Summary

Verdict: {PASS | CONDITIONAL PASS | FAIL}
Open items:
| ID | Priority | Summary |
|----|----------|---------|
| D-2 | 🔴 Required | ... (one phrase) |
| Q-3 | 🟡 Recommended | ... (one phrase) |
```

If there are no open items, write "none" on the Open items line.

---

### Verdict

```
## Verdict: {PASS | CONDITIONAL PASS | FAIL}

### Open Items

| ID | Priority | Description |
|----|----------|-------------|
| D-2 | 🔴 Required | ... |
| Q-3 | 🟡 Recommended | ... |
```

If there are no open items, write "none" in the Open Items section.

---

### Check Results

#### 1. Source Verification

Include all items, including ✅ entries.

| # | Item | Result |
|---|------|--------|
| V-1 | ... | ✅ |

#### 2. Coverage

Include all items, including ✅ entries.

Use subsections for: User Operation Paths / Logs / API / Dependencies & Preconditions / Acceptance Criteria.

#### 3. Quality

Include unresolved items only. Resolved items must be moved to the Resolution Log.

| # | Priority | Type | Description |
|---|----------|------|-------------|
| Q-1 | 🔴 Required | Ambiguous | ... |
| E-5 | ⚪ Optional | Edge case | ... |

Priority definitions:

| Priority | Meaning | Effect on verdict |
|----------|---------|-------------------|
| 🔴 Required | Must be resolved before implementation | Causes FAIL |
| 🟡 Recommended | Recommended to resolve, but implementation may proceed | Causes CONDITIONAL PASS |
| ⚪ Optional | Optional | No effect on verdict |

---

### Resolution Log

Append entries as issues are resolved. Never delete or overwrite existing entries.

```
## Resolution Log

| ID | Round / Date | Response & Reason | Status |
|----|-------------|-------------------|--------|
| V-2 | 1 / 2026-04-03 | Confirmed with PMG — covered by sub-spec, no spec change needed | ✅ Resolved |
| Q-1 | 2 / 2026-04-10 | Under confirmation with server team | 🔄 Pending |
```

Rules:
- `✅ Resolved` — reason required; record who confirmed and what the outcome was. "looks fine" is not acceptable.
- `🔄 Pending` — reason required; record who is confirming.
