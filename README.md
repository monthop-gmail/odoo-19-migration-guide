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
6. [Internal Attributes Deprecated](#6-internal-attributes-deprecated)
7. [Stored Compute Side Effects](#7-stored-compute-side-effects)
8. [CABA (Cash Basis) Behavior](#8-caba-cash-basis-behavior-on-draft)
9. [Domain Empty List Handling](#9-domain-empty-list-handling)
10. [Company Name Unique Constraint](#10-company-name-unique-constraint)
11. [Test Model Registration](#11-test-model-registration)
12. [psycopg2 vs psycopg3](#12-psycopg2-vs-psycopg3)
13. [Lazy Translation Formatting](#13-lazy-translation-formatting)
14. [Domain Expression API](#14-domain-expression-api)
15. [SQL Constraints Refactored](#15-sql-constraints-refactored)
16. [Timezone Access](#16-timezone-access)
17. [Route type="json" → type="jsonrpc"](#17-route-typejson--typejsonrpc)
18. [toggle_active → action_archive/action_unarchive](#18-toggle_active--action_archiveaction_unarchive)
19. [@api.private Decorator](#19-apiprivate-decorator)
20. [SUPERUSER_ID Import Changed](#20-superuser_id-import-changed)
21. [@ormcache_context Removed](#21-ormcache_context-removed)
22. [Demo Data Not Installed in Tests](#22-demo-data-not-installed-in-tests)
23. [Delete Old migrations Folder](#23-delete-old-migrations-folder)
24. [OCA Migration PR Guidelines](#24-oca-migration-pr-guidelines)
25. [Search Function Operator Rewriting](#25-search-function-operator-rewriting)
26. [res.partner.title Removed from Base](#26-respartnertitle-removed-from-base)
27. [safe_eval() Signature Changed](#27-safe_eval-signature-changed)
28. [base.user_demo Removed](#28-baseuser_demo-removed)
29. [hr.expense.sheet Removed](#29-hrexpensesheet-removed)
30. [department_id Moved to hr.employee](#30-department_id-moved-to-hremployee)
31. [OCA Unreleased Dependencies in CI](#31-oca-unreleased-dependencies-in-ci)
32. [create(self, vals) → create(self, vals_list)](#32-createself-vals--createself-vals_list)
33. [res.groups: category_id → privilege_id via res.groups.privilege](#33-resgroups-category_id--privilege_id-via-resgroupsprivilege)
34. [ir.actions.act_window: target='inline' → target='main'](#34-iractionsact_window-targetinline--targetmain)
35. [Search View Group Attributes Removed](#35-search-view-group-attributes-removed)
36. [ir.default.set Broken for TransientModel Fields](#36-irdefaultset-broken-for-transientmodel-fields)
37. [Field Removals in Core Models](#37-field-removals-in-core-models)
38. [XML ID Changes](#38-xml-id-changes)
39. [Website Template XPath Changes](#39-website-template-xpath-changes)
40. [_constraints List Format → Named Class Attributes](#40-_constraints-list-format--named-class-attributes)

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

`auto_join=True` is no longer supported for `One2many` fields. For `Many2one`/`Many2many`, it has been renamed to `bypass_search_access`.

Ref: [odoo/odoo#219627](https://github.com/odoo/odoo/pull/219627)

```python
# 18.0 — One2many
review_ids = fields.One2many(
    comodel_name="tier.review",
    inverse_name="res_id",
    auto_join=True,
)

# 19.0 — One2many: just remove it
review_ids = fields.One2many(
    comodel_name="tier.review",
    inverse_name="res_id",
)

# 18.0 — Many2one
partner_id = fields.Many2one("res.partner", auto_join=True)

# 19.0 — Many2one: rename to bypass_search_access
partner_id = fields.Many2one("res.partner", bypass_search_access=True)
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

## 6. Internal Attributes Deprecated

Direct access to `_context`, `_cr`, and `_uid` triggers `DeprecationWarning` in Odoo 19. Use `self.env.*` instead.

```python
# 18.0
value = rec._context.get("key")
self._cr.execute("SELECT 1")
uid = self._uid

# 19.0
value = rec.env.context.get("key")
self.env.cr.execute("SELECT 1")
uid = self.env.uid
```

**Search tip:** `grep -rn "\._context\|self\._cr\|self\._uid" --include="*.py"` in your module.

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

In Odoo 19, test models use the native `add_to_registry` pattern. The `odoo-test-helper` library with `FakeModelLoader` is no longer needed.

```python
# 18.0 — using odoo-test-helper
from odoo_test_helper import FakeModelLoader

@classmethod
def setUpClass(cls):
    cls.loader = FakeModelLoader(cls.env, cls.__module__)
    cls.loader.backup_registry()
    from .fake_models import FakeModel
    cls.loader.update_registry((FakeModel,))

@classmethod
def tearDownClass(cls):
    cls.loader.restore_registry()

# 19.0 — native add_to_registry
from odoo.orm.model_classes import add_to_registry

@classmethod
def setUpClass(cls):
    super().setUpClass()
    from .fake_models import FakeModel
    add_to_registry(cls.registry, FakeModel)
    cls.registry._setup_models_(cls.env.cr, ["fake.model"])
    cls.registry.init_models(
        cls.env.cr, ["fake.model"], {"models_to_check": True}
    )
    cls.addClassCleanup(cls.registry.__delitem__, "fake.model")
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

## 14. Domain Expression API

`odoo.osv.expression` is replaced by the new `odoo.fields.Domain` / `odoo.Domain` API.

Ref: [odoo/odoo#217708](https://github.com/odoo/odoo/pull/217708), [odoo/odoo#170009](https://github.com/odoo/odoo/pull/170009)

```python
# 18.0
from odoo.osv import expression
domain = expression.AND([domain1, domain2])
domain = expression.OR([domain1, domain2])

# 19.0
from odoo.fields import Domain
# or: from odoo import Domain
domain = Domain(domain1) & Domain(domain2)  # AND
domain = Domain(domain1) | Domain(domain2)  # OR
```

Also, computed non-stored field `search` methods must now return a `Domain` object:

```python
# 18.0
def _search_my_field(self, operator, value):
    return [("other_field", operator, value)]

# 19.0
def _search_my_field(self, operator, value):
    return Domain([("other_field", operator, value)])
```

Ref: [odoo/odoo@4f0d467](https://github.com/odoo/odoo/commit/4f0d4670edea0e989fc939e3b2e75cefd351cfc3)

---

## 15. SQL Constraints Refactored

`_sql_constraints` class variable is replaced by `models.Constraint` and `models.Index`.

Ref: [odoo/odoo#175783](https://github.com/odoo/odoo/pull/175783)

```python
# 18.0
class MyModel(models.Model):
    _name = "my.model"
    _sql_constraints = [
        ("name_uniq", "unique(name)", "Name must be unique!"),
    ]

# 19.0
class MyModel(models.Model):
    _name = "my.model"
    _constraints = [
        models.Constraint(
            "unique(name)",
            "Name must be unique!",
        ),
    ]
```

---

## 16. Timezone Access

Direct pytz manipulation and `context.get("tz")` is replaced by `self.env.tz`.

Ref: [odoo/odoo#221541](https://github.com/odoo/odoo/pull/221541)

```python
# 18.0
import pytz
tz = pytz.timezone(self.env.context.get("tz") or "UTC")

# 19.0
tz = self.env.tz  # returns a timezone object directly
```

---

## 17. Route type="json" → type="jsonrpc"

In controller `@route` decorators, `type="json"` is renamed to `type="jsonrpc"`.

Ref: [odoo/odoo#183636](https://github.com/odoo/odoo/pull/183636)

```python
# 18.0
@http.route("/my/endpoint", type="json", auth="user")
def my_endpoint(self, **kwargs):
    ...

# 19.0
@http.route("/my/endpoint", type="jsonrpc", auth="user")
def my_endpoint(self, **kwargs):
    ...
```

---

## 18. toggle_active → action_archive/action_unarchive

`toggle_active()` is replaced by explicit `action_archive()` and `action_unarchive()`.

Ref: [odoo/odoo#183691](https://github.com/odoo/odoo/pull/183691)

```python
# 18.0
record.toggle_active()

# 19.0
record.action_archive()    # set active=False
record.action_unarchive()  # set active=True
```

If you override `toggle_active`, move the logic to `action_archive`/`action_unarchive` as appropriate.

---

## 19. @api.private Decorator

New `@api.private` decorator marks methods as inaccessible from external APIs (XML-RPC, JSON-RPC).

Ref: [odoo/odoo#195402](https://github.com/odoo/odoo/pull/195402)

```python
# 19.0 — mark internal methods
from odoo import api

@api.private
def _internal_method(self):
    ...
```

This is informational — existing code won't break, but consider adding it to internal methods for security.

---

## 20. SUPERUSER_ID Import Changed

`SUPERUSER_ID` import location changed.

Ref: [odoo/odoo@d6a955f](https://github.com/odoo/odoo/commit/d6a955f10483)

```python
# 18.0
from odoo import SUPERUSER_ID

# 19.0
from odoo.api import SUPERUSER_ID
```

---

## 21. @ormcache_context Removed

`@ormcache_context` decorator is removed. Use `@ormcache` with explicit context key access.

Ref: [odoo/odoo#220725](https://github.com/odoo/odoo/pull/220725)

```python
# 18.0
from odoo.tools import ormcache_context

@ormcache_context(keys=("lang",))
def _get_name(self):
    ...

# 19.0
from odoo.tools import ormcache

@ormcache("self.env.context.get('lang')")
def _get_name(self):
    ...
```

---

## 22. Demo Data Not Installed in Tests

In Odoo 19, demo data is **no longer installed by default** in tests. Tests must create their own data explicitly.

```python
# 18.0 — could rely on demo data
partner = self.env.ref("base.res_partner_1")

# 19.0 — create test data explicitly
partner = self.env["res.partner"].create({"name": "Test Partner"})
```

Also consider setting `tracking_disable` in test setUp for performance:

```python
@classmethod
def setUpClass(cls):
    super().setUpClass()
    cls.env = cls.env(context=dict(cls.env.context, tracking_disable=True))
```

---

## 23. Delete Old migrations Folder

When migrating, delete the entire `migrations/` folder from the module if it exists. These are historical scripts from previous version upgrades and should not be carried forward.

---

## 24. OCA Migration PR Guidelines

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

## 25. Search Function Operator Rewriting

**Severity: HIGH** — Silently breaks search functions on computed non-stored fields.

In Odoo 19, the domain optimizer rewrites `("field", "=", value)` into `("field", "in", [value])` at `OptimizationLevel.BASIC` **before** the field's search function is called at `OptimizationLevel.FULL`. This means search methods that only check for `operator == "="` will never match.

### Root cause

In `odoo/orm/domains.py`, `_OPTIMIZATIONS_FOR[BASIC]["="]` includes `_operator_equal_as_in`, which runs at optimization level 1 (BASIC). Search functions are invoked at level 3 (FULL). So by the time your search function sees the domain, `=` has already become `in`.

### How to fix

Search functions must handle **both** the original and rewritten operator forms:

```python
# 18.0 — only handles "=" operator
def _search_my_field(self, operator, value):
    if operator == "=" and value is False:
        return [("id", "in", self._get_no_value_ids())]
    ...

# 19.0 — must also handle "in" with OrderedSet({False})
def _search_my_field(self, operator, value):
    if (operator == "=" and value is False) or (
        operator == "in" and list(value) == [False]
    ):
        # IMPORTANT: normalize value back when rewriting operator
        value = False
        return [("id", "in", self._get_no_value_ids())]
    ...
```

### Key detail: OrderedSet, not list

The optimizer wraps values in `OrderedSet`, not a plain list. So `value == [False]` will **never match** — you must use `list(value) == [False]` to compare correctly.

Also, when flipping the operator (e.g., `"="` → `"!="`) remember to normalize `value` back to `False`, otherwise downstream searches get nonsensical domains like `("other_field", "!=", OrderedSet({False}))`.

### Real-world example

`base_tier_validation._search_reviewer_ids` broke because:
1. Test searched `[("reviewer_ids", "=", False)]`
2. Optimizer rewrote to `[("reviewer_ids", "in", [False])]`
3. Search function received `operator="in"`, `value=[False]`
4. Only checked `operator == "="`, so didn't flip the logic
5. Returned wrong results (0 records instead of expected)

---

## Migration Checklist

Use this checklist when migrating a module:

- [ ] Version bump in `__manifest__.py`
- [ ] Update README.rst and index.html URLs
- [ ] Delete `migrations/` folder if exists
- [ ] Search for `groups_id` → `group_ids`
- [ ] Search for `.users` on groups → `.user_ids`
- [ ] Search for `models.NewId` → `not isinstance(id, int)`
- [ ] Search for `read_group` → `_read_group` with aggregates
- [ ] Search for `auto_join` → remove (One2many) or rename to `bypass_search_access` (Many2one/Many2many)
- [ ] Search for `button_draft` override → add `remove_move_reconcile()` if needed
- [ ] Search for `._context` / `._cr` / `._uid` → `.env.context` / `.env.cr` / `.env.uid`
- [ ] Search for `odoo.osv.expression` → `odoo.fields.Domain`
- [ ] Search for `_sql_constraints` → `models.Constraint`
- [ ] Search for timezone via `context.get("tz")` → `self.env.tz`
- [ ] Search for `type="json"` in routes → `type="jsonrpc"`
- [ ] Search for `toggle_active` → `action_archive` / `action_unarchive`
- [ ] Search for `from odoo import SUPERUSER_ID` → `from odoo.api import SUPERUSER_ID`
- [ ] Search for `@ormcache_context` → `@ormcache`
- [ ] Check stored compute methods for side effects
- [ ] Check test company names for uniqueness
- [ ] Check psycopg2 imports
- [ ] Check `FakeModelLoader` → native `add_to_registry`
- [ ] Check tests don't rely on demo data
- [ ] Check search functions on computed fields — handle `"in"` operator (optimizer rewrites `"="` to `"in"`)
- [ ] Run pre-commit for translation formatting
- [ ] `res.partner.title` → replaced by OCA `partner_title` module (`title_id` field)
- [ ] `safe_eval()` signature: `globals_dict=` → `context=`
- [ ] `base.user_demo` removed — create test users explicitly
- [ ] `hr.expense.sheet` removed — use `hr.expense` directly
- [ ] `department_id` moved from `res.users` to `hr.employee`
- [ ] OCA unreleased dependencies — vendor modules + `.codecov.yml` ignore
- [ ] `create(self, vals)` → `create(self, vals_list)` — handle list of vals dicts
- [ ] `res.groups.category_id` removed → use `privilege_id` via `res.groups.privilege`
- [ ] `target='inline'` in `ir.actions.act_window` → `target='main'`
- [ ] Search view `<group>` elements: remove `string` and `expand` attributes
- [ ] `ir.default.set` on TransientModel fields → use `ir.config_parameter` instead
- [ ] Core field removals: `stock.move.name` → `reference`, `sale.order.line.product_uom` → `product_uom_id`, etc.
- [ ] XML ID changes: `product.product_category_all` → `product.product_category_goods`, `product.group_discount_per_so_line` → `sale.group_discount_per_so_line`
- [ ] Website template XPath changes: `o_product_terms_and_share` → `product_full_description`, `product_name` → `td_product_name`
- [ ] `_constraints` list-of-tuples format → named class attributes with `models.Constraint`

---

## 26. res.partner.title Removed from Base

In Odoo 19, the `title` field on `res.partner` has been **removed from the base module**. It is now provided by the OCA `partner_title` module as `title_id` (Many2one to `res.partner.title`).

### Impact

Any module that references `partner.title` will fail with:

```
ValueError: Wrong @depends on '_compute_name' (compute method of field res.partner.name).
Dependency field 'title' not found in model res.partner.
```

### How to fix

1. Add `partner_title` to your module's dependencies in `__manifest__.py`
2. Replace all `title` references with `title_id`

```python
# 18.0
@api.depends("title", "firstname", "lastname")
def _compute_name(self):
    title = self.title.name
    self.title = False  # onchange

# 19.0
@api.depends("title_id", "firstname", "lastname")
def _compute_name(self):
    title = self.title_id.name
    self.title_id = False  # onchange
```

```python
# __manifest__.py
"depends": ["partner_firstname", "partner_title"],  # add partner_title
```

**Also update XML views and search domains:**
```python
# 18.0
.search([("title", "!=", False)])

# 19.0
.search([("title_id", "!=", False)])
```

**Test gotcha:** If `partner_title` installs demo data (like "Miss", "Mr"), creating titles with the same name will violate the unique constraint. Use unique test names.

---

## 27. safe_eval() Signature Changed

In Odoo 19, `safe_eval()` no longer accepts the `globals_dict` keyword argument. The parameter has been renamed to `context`.

```python
# 18.0
from odoo.tools.safe_eval import safe_eval
result = safe_eval(expression, globals_dict={"rec": record})

# 19.0
from odoo.tools.safe_eval import safe_eval
result = safe_eval(expression, context={"rec": record})
```

**New signature:** `safe_eval(expr, /, context=None, *, mode='eval', filename=None)`

**Error message:** `TypeError: safe_eval() got an unexpected keyword argument 'globals_dict'`

---

## 28. base.user_demo Removed

The XML ID `base.user_demo` no longer exists in Odoo 19 CI. Tests that reference it will fail:

```
ValueError: External ID not found in the system: base.user_demo
```

### How to fix

Create test users explicitly instead of using `env.ref`:

```python
# 18.0
self.test_user = self.env.ref("base.user_demo")

# 19.0
self.test_user = self.env["res.users"].create({
    "name": "Test User",
    "login": "test_user",
    "email": "test@example.com",
})
```

**Related:** Other demo XML IDs like `base.res_partner_1`, `base.res_partner_12`, `product.product_product_7` are also not available. See [Section 22](#22-demo-data-not-installed-in-tests).

---

## 29. hr.expense.sheet Removed

In Odoo 19, the `hr.expense.sheet` model has been **completely removed**. Expense workflows now use `hr.expense` directly.

### Impact

- Any module inheriting `hr.expense.sheet` will fail at model registration
- Views, actions, and security rules referencing `hr.expense.sheet` will break
- Tests creating expense sheets need a complete rewrite

### How to fix

Rewrite all `hr.expense.sheet` logic to work with `hr.expense` directly:

```python
# 18.0
class HrExpenseSheet(models.Model):
    _inherit = "hr.expense.sheet"
    custom_field = fields.Char()

# 19.0
class HrExpense(models.Model):
    _inherit = "hr.expense"
    custom_field = fields.Char()
```

Security rules, views, and menu items must also be updated to reference `hr.expense` instead of `hr.expense.sheet`.

---

## 30. department_id Moved to hr.employee

In Odoo 19, `department_id` has been **removed from `res.users`**. It now lives exclusively on `hr.employee`.

### Also: `hr.employee.base` removed

The abstract model `hr.employee.base` no longer exists. Modules inheriting it must use `hr.employee` instead.

```python
# 18.0
class HrEmployeeBase(models.AbstractModel):
    _inherit = "hr.employee.base"

user.department_id  # direct access on res.users

# 19.0
class HrEmployee(models.Model):
    _inherit = "hr.employee"

user.employee_id.department_id  # access via employee
```

### Test implications

Tests that assign `department_id` on users must first create an employee record:

```python
# 18.0
user.department_id = department

# 19.0
employee = self.env["hr.employee"].create({
    "name": user.name,
    "user_id": user.id,
    "department_id": department.id,
})
```

---

## 31. OCA Unreleased Dependencies in CI

When migrating module A that depends on module B, but module B's 19.0 version isn't published on PyPI yet, OCA CI will fail with:

```
ERROR: Could not find a version that satisfies the requirement odoo-addon-module_b==19.0.*
```

### How to fix

1. **Include the dependency module's source code** in your PR branch (copy from the other repo's migration PR branch)
2. **Add `.codecov.yml`** to exclude vendored modules from coverage analysis:

```yaml
# .codecov.yml
coverage:
  status:
    project:
      default:
        target: auto
    patch:
      default:
        target: auto
ignore:
  - "vendored_module_a/**"
  - "vendored_module_b/**"
```

3. **Fix `__manifest__.py` website URL** in vendored modules — OCA pre-commit checks that `website` matches the current repo URL:

```python
# If vendoring from partner-contact into l10n-thailand:
"website": "https://github.com/OCA/l10n-thailand",  # must match target repo
```

**Tip:** Once the dependency module's PR is merged and published, remove the vendored copy and the `.codecov.yml` ignore entry.

---

## 32. create(self, vals) → create(self, vals_list)

In Odoo 19, `create()` always receives a **list** of vals dicts (multi-create API). Modules overriding `create(self, vals)` that call `vals.get()` directly will crash with `AttributeError: 'list' object has no attribute 'get'`.

```python
# 18.0
@api.model
def create(self, vals):
    if vals.get('url_handler'):
        # validate url_handler
    return super().create(vals)

# 19.0
@api.model
def create(self, vals_list):
    if not isinstance(vals_list, list):
        vals_list = [vals_list]
    for vals in vals_list:
        if vals.get('url_handler'):
            # validate url_handler
    return super().create(vals_list)
```

**Error:** `AttributeError: 'list' object has no attribute 'get'`

---

## 33. res.groups: category_id → privilege_id via res.groups.privilege

In Odoo 19, `res.groups` no longer has a direct `category_id` field. Instead, it uses `privilege_id` pointing to `res.groups.privilege`, which then has a `category_id` pointing to `ir.module.category`.

```xml
<!-- 18.0 -->
<record id="marketplace_category" model="ir.module.category">
    <field name="name">Marketplace</field>
</record>
<record id="my_group" model="res.groups">
    <field name="category_id" ref="marketplace_category"/>
</record>

<!-- 19.0 -->
<record id="marketplace_privilege" model="res.groups.privilege">
    <field name="name">Marketplace</field>
    <field name="category_id" ref="marketplace_category"/>
</record>
<record id="my_group" model="res.groups">
    <field name="privilege_id" ref="marketplace_privilege"/>
</record>
```

**Error:** `ValueError: Invalid field 'category_id' in 'res.groups'`

---

## 34. ir.actions.act_window: target='inline' → target='main'

`target='inline'` is no longer a valid value for `ir.actions.act_window`. Use `target='main'` instead.

```xml
<!-- 18.0 -->
<field name="target">inline</field>

<!-- 19.0 -->
<field name="target">main</field>
```

---

## 35. Search View Group Attributes Removed

In search views, `<group>` elements no longer support `string` or `expand` attributes.

```xml
<!-- 18.0 -->
<group expand="0" string="Group By">
    <filter .../>
</group>

<!-- 19.0 -->
<group>
    <filter .../>
</group>
```

---

## 36. ir.default.set Broken for TransientModel Fields

`ir.default.set` via XML function calls no longer works for fields on TransientModel (e.g. `res.config.settings`) because the `field_id` lookup returns null. Use Python defaults or `ir.config_parameter` instead.

---

## 37. Field Removals in Core Models

Many fields were removed or renamed in Odoo 19 core models (Webkul/Marketplace-relevant):

| Model | Removed/Changed | Replacement |
|-------|----------------|-------------|
| `res.partner` | `mobile` | removed |
| `res.partner` | `title` | removed from base (see [Section 26](#26-respartnertitle-removed-from-base)) |
| `stock.picking` | `date` | use `scheduled_date` |
| `stock.picking` | `group_id` | removed |
| `stock.move` | `name` | `reference` |
| `stock.move` | `product_uom_category_id` | removed |
| `sale.order.line` | `product_uom` | `product_uom_id` |
| `sale.order.line` | `product_uom_category_id` | removed |
| `sale.order` | `procurement_group_id` | removed |
| `product.template` | `uom_po_id` | removed |
| `product.template` | `_get_default_category_id()` | removed |

---

## 38. XML ID Changes

| Old XML ID | New XML ID |
|-----------|-----------|
| `product.product_category_all` | `product.product_category_goods` |
| `product.group_discount_per_so_line` | `sale.group_discount_per_so_line` |

---

## 39. Website Template XPath Changes

Several website template elements were renamed/removed in Odoo 19. Inherited views using XPath must be updated:

| Old XPath target | Replacement |
|-----------------|-------------|
| `//div[@id='o_product_terms_and_share']` | `//div[@id='product_full_description']` |
| `//td[@id='product_name']` | `//td[@name='td_product_name']` |

---

## 40. _constraints List Format → Named Class Attributes

The `_constraints` list syntax is no longer supported. Use named class attributes with `models.Constraint`:

```python
# 18.0
_constraints = [
    ('name_uniq', 'unique(name)', 'Name must be unique!'),
]

# 19.0 (named attribute form)
_name_uniq = models.Constraint(
    'unique(name)',
    'Name must be unique!',
)
```

Note: This differs from [Section 15](#15-sql-constraints-refactored) which shows the `_sql_constraints` migration. The old `_constraints` list-of-tuples format is completely removed.

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
