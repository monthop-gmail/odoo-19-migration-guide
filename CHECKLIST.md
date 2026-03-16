# Odoo 18 → 19 Migration Checklist

Copy this checklist into your PR description or use it as a reference.

## Pre-migration

- [ ] Create branch from upstream/19.0
- [ ] Apply 18.0 history with `git format-patch | git am`

## Code Changes

### Mandatory Checks
- [ ] `__manifest__.py` version: `19.0.1.0.0`
- [ ] README.rst badge URLs: `18.0` → `19.0`
- [ ] `static/description/index.html` URLs: `18.0` → `19.0`

### Field Renames
- [ ] `res.groups.users` → `user_ids` (in code AND XML)
- [ ] `res.users.groups_id` → `group_ids` (in code AND XML)

### API Changes
- [ ] `models.NewId` → `not isinstance(rec.id, int)`
- [ ] `read_group()` → `_read_group(aggregates=[...])` with tuple unpacking
- [ ] `rec._context` → `rec.env.context`

### Field Definition Changes
- [ ] Remove `auto_join=True` from `One2many` fields

### Method Changes
- [ ] `button_draft()` overrides: add `remove_move_reconcile()` if needed
- [ ] Check stored compute methods for side-effect writes to non-computed fields

### Test Changes
- [ ] Company names must be unique (not "My Company")
- [ ] Handle psycopg2 → psycopg3 imports
- [ ] Fix lazy translation formatting (W8301)
- [ ] CABA test assertions: draft entries are deleted, not reversed
- [ ] Stored compute side effects: call `invalidate_recordset()` + compute explicitly

## Post-migration

- [ ] Run pre-commit locally
- [ ] Single `[MIG] module_name: Migration to 19.0` commit on top of history
- [ ] Push and verify CI passes
