# Odoo 18 → 19 Migration — AI Agent Instructions

You are migrating an Odoo module from version 18.0 to 19.0.
Read `migration-rules.yaml` for all search/replace patterns, then follow the workflow below.

## Workflow

### Step 1: Scan the module

Run every `detect` pattern from `migration-rules.yaml` against the target module directory.
Report which rules matched and how many hits each rule has.
Do NOT start changing code until the scan is complete.

### Step 2: Cleanup

- Delete the `migrations/` folder if it exists
- Remove CREDITS.rst references to prior migration funding

### Step 3: Apply fixes (automated rules)

For each matched rule where `auto_fix: true`:
1. Apply the replacement pattern
2. Verify the change makes sense in context (don't blindly replace — e.g. `.users` could be a custom field, not `res.groups.users`)

For each matched rule where `auto_fix: false`:
1. Show the match to the user
2. Explain what needs to change and why
3. Propose a fix and wait for confirmation

### Step 4: Version bump

1. In `__manifest__.py`: change version from `18.0.x.x.x` to `19.0.1.0.0`
2. In `README.rst`: replace all `18.0` in badge/URL to `19.0`
3. In `static/description/index.html`: replace all `18.0` in URLs to `19.0`

### Step 5: Validate

Run ALL checks from the `validation` section of `migration-rules.yaml`:

**Must not exist:**
- `groups_id` references (unless custom field)
- `models.NewId`
- `auto_join=True`
- `._context` / `self._cr` / `self._uid`
- `.read_group(` (should be `._read_group(`)
- `from odoo.osv import expression` (should use `odoo.fields.Domain`)
- `_sql_constraints` (should use `models.Constraint`)
- `type="json"` in routes (should be `type="jsonrpc"`)
- `toggle_active` (should be `action_archive`/`action_unarchive`)
- `ormcache_context` (should be `@ormcache`)
- `from odoo import SUPERUSER_ID` (should be `from odoo.api import`)
- `FakeModelLoader` (should use native `add_to_registry`)
- `globals_dict=` in safe_eval calls (should be `context=`)
- `env.ref("base.user_demo")` (XML ID removed — create test users explicitly)
- `hr.expense.sheet` (model removed — use `hr.expense` directly)
- `hr.employee.base` (model removed — use `hr.employee`)
- `target.*inline` in ir.actions.act_window (should be `target='main'`)
- `product.product_category_all` (renamed to `product.product_category_goods`)
- `product.group_discount_per_so_line` (moved to `sale.group_discount_per_so_line`)

**Needs manual review:**
- `_search_` methods that check `operator == "="` must also handle `"in"` (optimizer rewrites `=` to `in` before search runs)
- `company_id` in `account.account` create/write — field removed in 19.0; use `.with_company()` on environment instead
- `'consu'` in product.template type — renamed to `'goods'`; `'product'` removed (use `'goods'` + `is_storable=True`)

**Must exist:**
- Version `19.0.x.x.x` in `__manifest__.py`

**Directories must not exist:**
- `migrations/`

### Step 6: Commit (if asked)

Use the OCA commit convention:
```
[MIG] module_name: Migration to 19.0
```

## Important Rules

- NEVER squash historical commits when preparing OCA PRs
- When unsure if a match is a false positive (e.g. `.users` on a non-groups model), ASK the user
- `button_draft`, stored compute side effects, `_sql_constraints`, Domain API changes, and `_search_` methods require manual review — do not auto-fix
- `auto_join`: check field type first — One2many removes it, Many2one/Many2many renames to `bypass_search_access`
- Test files need extra attention: company names must be unique, psycopg imports must handle both v2/v3, demo data is not available
- `self.env.ref()` in tests may break if it references demo data — create test data explicitly instead
- Always read the full context around a match before replacing — at least 5 lines above and below

## Common False Positives (from real migrations)

These rules have high false-positive rates — always verify model context:

- **`category_id`** (rule `field-groups-category-id`): Only affects `res.groups`. Modules with custom category models (e.g. `legal.form.category`, `event.type.category`) will generate many false hits. Check `_name` or `model=` attribute before flagging.
- **`<group string=`** (rule `search-view-group-attrs`): Only affects search views. Form views use `<group string="...">` normally — do NOT change those. Look for the parent `<search>` element to confirm.
- **`self.env.ref()`** (rule `test-demo-data`): Only problematic when referencing demo data XML IDs (`base.res_partner_*`, `base.user_demo`). References to the module's own XML IDs (e.g. `legal_forms.action_report_*`) are safe.
- **`@api.depends`** (rule `method-compute-side-effects`): Only problematic when the compute writes to fields OTHER than the computed field. Simple count/preview computes are safe.
- **`company_id`** (rule `field-account-company-id-removed`): Only affects `account.account`. All other models (`account.move`, `res.partner`, `sale.order`, etc.) still have `company_id`. Always verify the model before flagging.
- **`'consu'`** (rule `field-product-type-consu-removed`): Only affects `product.template.type` selection. Could appear in comments, variable names, or unrelated strings.

## File References

- `README.md` — full migration guide with explanations and code examples (40 sections)
- `CHECKLIST.md` — copy-paste checklist for PR descriptions
- `migration-rules.yaml` — machine-readable detection and fix patterns

## Source References

- [OCA Migration Wiki](https://github.com/OCA/maintainer-tools/wiki/Migration-to-version-19.0)
- All `reference` fields in `migration-rules.yaml` link to the relevant Odoo PRs/commits
