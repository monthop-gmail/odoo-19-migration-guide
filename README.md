# Odoo 18 → 19 Migration Guide

Comprehensive guide for migrating Odoo modules from version 18 to 19.
Covers breaking changes, deprecated APIs, and migration patterns with real code examples.

> **Built from real OCA migration experience** — every item here caused actual CI failures.

---

## Table of Contents

1. [Field Renames](#1-field-renames)
2. [models.NewId Removed](#2-modelsnewid-removed)
3. [read_group Deprecated](#3-read_group-deprecated)
4. [One2many auto_join Removed](#4-one2many-auto_join-removed)
5. [button_draft() Changed](#5-button_draft-changed)
6. [rec._context Deprecated](#6-rec_context-deprecated)
7. [Stored Compute Side Effects](#7-stored-compute-side-effects)
8. [CABA (Cash Basis) Behavior](#8-caba-cash-basis-behavior-on-draft)
9. [Domain Empty List Handling](#9-domain-empty-list-handling)
10. [Company Name Unique Constraint](#10-company-name-unique-constraint)
11. [Test Model Registration](#11-test-model-registration)
12. [psycopg2 vs psycopg3](#12-psycopg2-vs-psycopg3)
13. [Lazy Translation Formatting](#13-lazy-translation-formatting)
14. [OCA Migration PR Guidelines](#14-oca-migration-pr-guidelines)

---

## 1. Field Renames

### res.groups: `users` → `user_ids`

```python
# 18.0
self.env["res.groups"].create({
    "name": "MyGroup",
    "users": [(4, user.id)],
})
group.users

# 19.0
self.env["res.groups"].create({
    "name": "MyGroup",
    "user_ids": [(4, user.id)],
})
group.user_ids
```

### res.users: `groups_id` → `group_ids`

```python
# 18.0
self.env.user.groups_id.ids
("group_ids", "in", self.env.user.groups_id.ids)

# 19.0
self.env.user.group_ids.ids
("group_ids", "in", self.env.user.group_ids.ids)
```

**Search tip:** `grep -rn "\.users\b" --include="*.py"` and `grep -rn "groups_id"` in your module.

---

## 2. models.NewId Removed

`odoo.models.NewId` no longer exists. It moved to `odoo.orm.identifiers.NewId`, but the recommended pattern is:

```python
# 18.0
from odoo import models
if isinstance(rec.id, models.NewId):
    ...

# 19.0
if not isinstance(rec.id, int):
    ...
```

---

## 3. read_group Deprecated

`read_group()` is deprecated. Use `_read_group()` with `aggregates` parameter.
**Key change:** Returns tuples instead of dicts.

```python
# 18.0
results = self.env["account.move.line"].read_group(
    domain=[("move_id", "in", moves.ids)],
    fields=["move_id", "account_id", "debit:sum", "credit:sum"],
    groupby=["move_id", "account_id"],
)
for group in results:
    move = group["move_id"]
    debit = group["debit"]

# 19.0
results = self.env["account.move.line"]._read_group(
    domain=[("move_id", "in", moves.ids)],
    groupby=["move_id", "account_id"],
    aggregates=["debit:sum", "credit:sum"],
)
for move, account, debit_sum, credit_sum in results:
    # move and account are recordsets (not tuples like read_group)
    # debit_sum and credit_sum are numeric values
```

**Migration steps:**
1. Replace `read_group()` with `_read_group()`
2. Move aggregated fields from `fields` to `aggregates` parameter
3. Change dict access to tuple unpacking
4. Note: groupby fields return recordsets, not `(id, name)` tuples

---

## 4. One2many auto_join Removed

`auto_join=True` is no longer supported for `One2many` fields. Simply remove it.

```python
# 18.0
review_ids = fields.One2many(
    comodel_name="tier.review",
    inverse_name="res_id",
    auto_join=True,
)

# 19.0
review_ids = fields.One2many(
    comodel_name="tier.review",
    inverse_name="res_id",
)
```

**Error message:** `TypeError: __init__() got an unexpected keyword argument 'auto_join'`

---

## 5. button_draft() Changed

In Odoo 18, `account.move.button_draft()` internally called `remove_move_reconcile()` (around line 5566).
In Odoo 19, **this call was removed**. Modules overriding `button_draft()` that depend on unreconciliation must call it explicitly.

```python
# 19.0 — add to your button_draft override
def button_draft(self):
    # In Odoo 19, button_draft no longer calls remove_move_reconcile,
    # so we must do it here before calling super.
    self.mapped("line_ids").remove_move_reconcile()
    res = super().button_draft()
    # your custom logic here
    return res
```

**Impact:** Without this fix, CABA (Cash Basis) entries and reconciliation-dependent logic will break silently.

---

## 6. rec._context Deprecated

`rec._context` triggers a `DeprecationWarning` in Odoo 19.

```python
# 18.0
value = rec._context.get("key")
timezone = self._context.get("tz")

# 19.0
value = rec.env.context.get("key")
timezone = self.env.context.get("tz")
```

**Search tip:** `grep -rn "\._context" --include="*.py"` in your module.

---

## 7. Stored Compute Side Effects

In Odoo 19, writing to **non-computed fields** as a side effect inside a stored compute method does **NOT reliably persist** during `create()`.

```python
# This pattern BREAKS in Odoo 19:
@api.depends("definition_id.approve_sequence")
def _compute_can_review(self):
    for record in self:
        # Side effect — won't persist during create() in Odoo 19!
        record.status = "pending"
        # Actual computed field — works fine
        record.can_review = record._can_review_value()
```

**Fixes:**
- Call the compute method explicitly after `create()` when needed
- Move side-effect logic to a separate method
- Set values directly after `create()`, not inside compute
- In tests, call `invalidate_recordset()` + `_compute_method()` explicitly before asserting

**Example test fix:**
```python
# 18.0 — relied on ORM flush triggering compute side effects
record.request_validation()
self.assertEqual(record.review_ids[0].status, "pending")

# 19.0 — explicitly trigger compute
record.request_validation()
record.review_ids.invalidate_recordset()
record.review_ids._compute_can_review()
self.assertEqual(record.review_ids[0].status, "pending")
```

---

## 8. CABA (Cash Basis) Behavior on Draft

In Odoo 18, setting a posted CABA entry to draft created a **reversal entry**.
In Odoo 19, draft CABA entries are **deleted** (not reversed). Posted CABA entries are still reversed.

```python
# 18.0 — after action_draft, CABA entries are reversed (count stays/increases)
bill_caba = invoice.tax_cash_basis_created_move_ids
self.assertEqual(len(bill_caba), 2)  # original + reversal

# 19.0 — after action_draft, draft CABA entries are DELETED
bill_caba = invoice.tax_cash_basis_created_move_ids
self.assertFalse(bill_caba)  # deleted, not reversed
```

**Impact on tax_invoice modules:** If your module tracks CABA through `tax_invoice_ids` with `move_line_id` having `ondelete="cascade"`, the cascade delete will also remove related tax invoice records.

---

## 9. Domain Empty List Handling

In Odoo 19, `("id", "not in", [])` may not reliably match all records. Handle empty lists explicitly:

```python
# 18.0 — worked fine
return [("id", "not in", res_ids)]

# 19.0 — handle empty case explicitly
if not res_ids:
    if model_operator == "not in":
        return [(1, "=", 1)]   # TRUE domain — matches all
    return [(0, "=", 1)]       # FALSE domain — matches none
return [("id", model_operator, res_ids)]
```

---

## 10. Company Name Unique Constraint

`res_company` has a unique constraint `res_company_name_uniq` on the `name` field. Tests creating companies must use unique names:

```python
# Bad — may conflict with existing "My Company"
self.env["res.company"].create({"name": "My Company"})

# Good — use a unique test name
self.env["res.company"].create({"name": "Test Company For Tier Validation"})
```

**Error:** `psycopg2.errors.UniqueViolation: duplicate key value violates unique constraint "res_company_name_uniq"`

---

## 11. Test Model Registration

In Odoo 19, test models should use the `add_to_registry` pattern instead of the older `_setup_models__` (double underscore) pattern.

```python
# 19.0
from odoo.tests.common import tagged

@tagged("post_install", "-at_install")
class TestMyModule(TransactionCase):
    @classmethod
    def setUpClass(cls):
        super().setUpClass()
        # Models are registered via add_to_registry in __init__.py
```

---

## 12. psycopg2 vs psycopg3

Odoo 19 may use psycopg3. Tests importing directly from psycopg2 should handle both:

```python
try:
    from psycopg2 import IntegrityError
except ImportError:
    from psycopg import errors
    IntegrityError = errors.IntegrityError
```

---

## 13. Lazy Translation Formatting

Pre-commit / ruff enforces lazy translation formatting (W8301):

```python
# Bad — string formatting before translation
_("Message %s" % variable)
_("Message {}".format(variable))
_("Message " + variable)

# Good — pass args to translation function
_("Message %s", variable)
_("Message %(name)s", name=variable)
```

---

## 14. OCA Migration PR Guidelines

When submitting migration PRs to OCA repositories:

### Preserve commit history

```bash
# 1. Fetch both branches
git fetch upstream 18.0 19.0

# 2. Create branch from 19.0
git checkout -b 19.0-mig-module_name upstream/19.0

# 3. Apply 18.0 history
git format-patch --keep-subject --no-numbered --stdout \
    upstream/19.0..upstream/18.0 -- module_name \
    | git am -3 --keep-non-patch

# 4. Make migration changes and commit
git add module_name/
git commit -m "[MIG] module_name: Migration to 19.0"

# 5. Push
git push origin 19.0-mig-module_name
```

### Commit structure
- All historical commits from 18.0 preserved
- Single `[MIG] module_name: Migration to 19.0` commit on top
- **Never squash** the historical commits

### Version bump checklist
- `__manifest__.py`: version `18.0.x.x.x` → `19.0.1.0.0`
- `README.rst`: update badge URLs from `18.0` to `19.0`
- `static/description/index.html`: update URLs from `18.0` to `19.0`

---

## Migration Checklist

Use this checklist when migrating a module:

- [ ] Version bump in `__manifest__.py`
- [ ] Update README.rst and index.html URLs
- [ ] Search for `groups_id` → `group_ids`
- [ ] Search for `.users` on groups → `.user_ids`
- [ ] Search for `models.NewId` → `not isinstance(id, int)`
- [ ] Search for `read_group` → `_read_group` with aggregates
- [ ] Search for `auto_join` on One2many → remove
- [ ] Search for `button_draft` override → add `remove_move_reconcile()` if needed
- [ ] Search for `._context` → `.env.context`
- [ ] Check stored compute methods for side effects
- [ ] Check test company names for uniqueness
- [ ] Check psycopg2 imports
- [ ] Run pre-commit for translation formatting

---

## AI Agent Usage

This repo is optimized for AI-assisted migration. Point your AI agent at this repo and it will know what to do.

| File | Purpose |
|------|---------|
| `CLAUDE.md` | Agent instructions — workflow, rules, and guardrails |
| `migration-rules.yaml` | Machine-readable detect/fix patterns with regex, severity, and auto-fix flags |
| `README.md` | Full guide with explanations and code examples (this file) |
| `CHECKLIST.md` | Copy-paste checklist for PR descriptions |

### Quick start with Claude Code

```bash
# From your Odoo module directory:
claude "Migrate this module from Odoo 18 to 19. Use the guide at /path/to/odoo-19-migration-guide"
```

The agent will:
1. Scan your module against all known breaking changes
2. Auto-fix safe patterns (field renames, `_context`, `auto_join`)
3. Flag patterns that need human review (`button_draft`, compute side effects)
4. Bump version and update URLs
5. Validate that no known issues remain

---

## Contributing

Found a new breaking change? Open an issue or PR!

## License

This guide is provided as-is for the Odoo community. MIT License.
