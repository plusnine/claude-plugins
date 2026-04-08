# Default PR Template

Used when the repository does not have `.github/pull_request_template.md`.
Translate section headers per `language` in `~/.claude/settings.json` (fallback: English).

## Template

```
# Ticket Link

# Specification

# Changes

# Screenshot

# Notes
```

## Screenshot Markup Examples

### Single image (no dark mode)

```
# Screenshot
<img src="" width="320" />
```

### Before/After (no dark mode)

```
# Screenshot
| Before | After |
|--------|-------|
| <img src="" width="320" /> | <img src="" width="320" /> |
```

### Dark mode supported

```
# Screenshot
## Light
<img src="" width="320" />

## Dark
<img src="" width="320" />
```

### Dark mode + Before/After

```
# Screenshot
## Light
| Before | After |
|--------|-------|
| <img src="" width="320" /> | <img src="" width="320" /> |

## Dark
| Before | After |
|--------|-------|
| <img src="" width="320" /> | <img src="" width="320" /> |
```

### Not needed (no UI change)

```
# Screenshot
No UI changes in this PR.
```
