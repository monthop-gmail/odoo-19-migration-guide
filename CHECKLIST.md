# Odoo 18 → 19 Migration Checklist

Copy this checklist into your PR description or use it as a reference.

> For AI agents: use `migration-rules.yaml` for machine-readable detect/fix patterns.
> For full explanations and code examples: see `README.md`.

## Pre-migration

- [ ] Create branch from upstream/19.0
- [ ] Apply 18.0 history with `git format-patch | git am`
- [ ] Delete `migrations/` folder if it exists
- [ ] Remove CREDITS.rst references to prior migration funding

## Code Changes

### Mandatory Checks
- [ ] `__manifest__.py` version: `19.0.1.0.0`
- [ ] README.rst badge URLs: `18.0` → `19.0`
- [ ] `static/description/index.html` URLs: `18.0` → `19.0`

### Field Renames
- [ ] `res.groups.users` → `user_ids` (in code AND XML)
- [ ] `res.users.groups_id` → `group_ids` (in code, XML, CSV — also in `ir.ui.view`, `ir.ui.menu`, `ir.actions`, `ir.actions.report`)
- [ ] `res.groups.category_id` removed → use `privilege_id` via `res.groups.privilege`
- [ ] Core field removals: `stock.move.name` → `reference`, `sale.order.line.product_uom` → `product_uom_id`, `res.partner.mobile` removed, etc.

### API Changes
- [ ] `models.NewId` → `not isinstance(rec.id, int)`
- [ ] `read_group()` → `_read_group(aggregates=[...])` with tuple unpacking
- [ ] `rec._context` → `rec.env.context`
- [ ] `self._cr` → `self.env.cr`
- [ ] `self._uid` → `self.env.uid`
- [ ] `from odoo import SUPERUSER_ID` → `from odoo.api import SUPERUSER_ID`

### Domain & Expression Changes
- [ ] `odoo.osv.expression` → `odoo.fields.Domain` / `odoo.Domain`
- [ ] Computed field `_search` methods: return `Domain` object instead of list
- [ ] Empty list domain handling: explicitly check for `[]`

### Field Definition Changes
- [ ] `auto_join=True` on One2many → remove entirely
- [ ] `auto_join=True` on Many2one/Many2many → rename to `bypass_search_access=True`

### Model Definition Changes
- [ ] `_sql_constraints` → `models.Constraint` / `models.Index`
- [ ] `_constraints` list-of-tuples format → named class attributes with `models.Constraint`

### Method Changes
- [ ] `button_draft()` overrides: add `remove_move_reconcile()` if needed
- [ ] `toggle_active()` → `action_archive()` / `action_unarchive()`
- [ ] Check stored compute methods for side-effect writes to non-computed fields
- [ ] `create(self, vals)` → `create(self, vals_list)` — handle list of vals dicts (multi-create API)

### XML / View Changes
- [ ] `target='inline'` in `ir.actions.act_window` → `target='main'`
- [ ] Search view `<group>` elements: remove `string` and `expand` attributes
- [ ] Website template XPath changes: `o_product_terms_and_share` → `product_full_description`, `product_name` → `td_product_name`

### XML ID Changes
- [ ] `product.product_category_all` → `product.product_category_goods`
- [ ] `product.group_discount_per_so_line` → `sale.group_discount_per_so_line`

### Controller Changes
- [ ] `@route(type="json")` → `@route(type="jsonrpc")`

### Decorator / Import Changes
- [ ] `@ormcache_context` → `@ormcache` with explicit context key access
- [ ] Consider adding `@api.private` to internal methods

### Timezone
- [ ] `self.env.context.get("tz")` / pytz → `self.env.tz`

### Partner Title
- [ ] `res.partner.title` removed from base → add `partner_title` dependency, use `title_id`

### safe_eval()
- [ ] `safe_eval(expr, globals_dict={...})` → `safe_eval(expr, context={...})`

### ir.default
- [ ] `ir.default.set` on TransientModel fields broken → use `ir.config_parameter` instead

### HR Module Changes
- [ ] `hr.expense.sheet` removed → use `hr.expense` directly
- [ ] `hr.employee.base` removed → inherit `hr.employee` instead
- [ ] `res.users.department_id` removed → access via `user.employee_id.department_id`
- [ ] Expense sheet workflow methods: `action_submit_sheet` → `action_submit`, `approve_expense_sheets` → `action_approve`, `action_sheet_move_create` → `action_create_move`

### Accounting Model Changes
- [ ] `account.account.company_id` removed → do NOT pass `company_id` in create/write; use `.with_company()` on env if needed
- [ ] `product.template.type`: `"consu"` → `"goods"`, `"product"` removed → use `"goods"` + `is_storable=True`

### Test Changes
- [ ] Company names must be unique (not "My Company")
- [ ] Handle psycopg2 → psycopg3 imports
- [ ] Fix lazy translation formatting (W8301)
- [ ] CABA test assertions: draft entries are deleted, not reversed
- [ ] Stored compute side effects: call `invalidate_recordset()` + compute explicitly
- [ ] `FakeModelLoader` (odoo-test-helper) → native `add_to_registry`
- [ ] Tests must not rely on demo data — create test data explicitly
- [ ] `base.user_demo` XML ID removed — create test users explicitly
- [ ] Search functions on computed fields: handle `"in"` operator (optimizer rewrites `"="` to `"in"`)
- [ ] Consider `tracking_disable=True` in test context for performance

### CI / Dependencies
- [ ] Unreleased OCA dependencies: vendor module source + `.codecov.yml` ignore
- [ ] Vendored module `website` in `__manifest__.py` must match target repo URL

## Post-migration

- [ ] Run pre-commit locally
- [ ] Single `[MIG] module_name: Migration to 19.0` commit on top of history
- [ ] Push and verify CI passes
