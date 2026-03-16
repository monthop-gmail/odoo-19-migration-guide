# Odoo 18 → 19 Migration — AI Agent Instructions

You are migrating an Odoo module from version 18.0 to 19.0.
Read `migration-rules.yaml` for all search/replace patterns, then follow the workflow below.

## Workflow

### Step 1: Scan the module

Run every `detect` pattern from `migration-rules.yaml` against the target module directory.
Report which rules matched and how many hits each rule has.
Do NOT start changing code until the scan is complete.

### Step 2: Apply fixes (automated rules)

For each matched rule where `auto_fix: true`:
1. Apply the replacement pattern
2. Verify the change makes sense in context (don't blindly replace — e.g. `.users` could be a custom field, not `res.groups.users`)

For each matched rule where `auto_fix: false`:
1. Show the match to the user
2. Explain what needs to change and why
3. Propose a fix and wait for confirmation

### Step 3: Version bump

1. In `__manifest__.py`: change version from `18.0.x.x.x` to `19.0.1.0.0`
2. In `README.rst`: replace all `18.0` in badge/URL to `19.0`
3. In `static/description/index.html`: replace all `18.0` in URLs to `19.0`

### Step 4: Validate

Run the checks from the `validation` section of `migration-rules.yaml`:
- No remaining `groups_id` references (unless it's a custom field)
- No remaining `models.NewId`
- No remaining `auto_join` on One2many
- No remaining `._context` (should be `.env.context`)
- No `read_group(` calls (should be `_read_group(`)
- Version in `__manifest__.py` starts with `19.0`
- No lazy translation formatting violations

### Step 5: Commit (if asked)

Use the OCA commit convention:
```
[MIG] module_name: Migration to 19.0
```

## Important Rules

- NEVER squash historical commits when preparing OCA PRs
- When unsure if a match is a false positive (e.g. `.users` on a non-groups model), ASK the user
- `button_draft` and stored compute side effects require manual review — do not auto-fix
- Test files need extra attention: company names must be unique, psycopg imports must handle both v2 and v3
- Always read the full context around a match before replacing — at least 5 lines above and below

## File References

- `README.md` — full migration guide with explanations and code examples
- `CHECKLIST.md` — copy-paste checklist for PR descriptions
- `migration-rules.yaml` — machine-readable detection and fix patterns
