Migrate a page from legacy AdCenter (CI3) to the new AdCenter (CI4).

You are an expert in AdCenter migration. Follow these rules strictly.

## Input

The user will provide: **$ARGUMENTS**

This is typically a page name or description like "account settings", "campaigns list", "reports page", etc. If unclear, ask what legacy page to migrate.

## Phase 1: Research the Legacy Page

1. **Find the legacy controller** in `advertisers7_new.admedia.com/html/adcenter/controllers/` — read it fully
2. **Find the legacy view(s)** in `advertisers7_new.admedia.com/html/adcenter/views/` — read them
3. **Check legacy validation rules** in `advertisers7_new.admedia.com/html/adcenter/config/form_validation.php` for any relevant rule groups
4. **Check legacy models** in `advertisers7_new.admedia.com/html/adcenter/models/` for any used by the controller
5. **Check for linked user / masterid handling** — legacy `MY_Controller` (lines 46-49) loads `$this->user->linked_user` when `masterid` is set. Port this pattern if the page edits user-specific data.
6. **Find the design mockup** in `adcenter.admedia.com/design/` — read it for target UI

Present a summary to the user:
- Legacy methods to port (with brief description of each)
- DB tables and fields involved
- Design mockup found (yes/no, which file)
- Any linked user / role-based concerns

Ask: "Ready to implement, or any changes to scope?"

## Phase 2: Implement Following CI4 Patterns

### File structure (ALWAYS follow this)

| What | Where | Example |
|------|-------|---------|
| Service | `app/Services/{PageName}/{PageName}Service.php` | `app/Services/AccountSettings/AccountSettingsService.php` |
| Controller | `app/Controllers/{PageName}.php` | `app/Controllers/AccountSettings.php` |
| View | `app/Views/{page-name}/index.php` | `app/Views/account-settings/index.php` |
| Routes | `app/Config/Routes.php` (modify) | Add after last route group |

### Controller rules

- **Extend `BaseController`** — provides `$this->data`, `$this->session`, `$this->loggedIn`
- **Inject service in `initController()`** — follow the Billing controller pattern exactly:
  ```php
  public function initController(
      \CodeIgniter\HTTP\RequestInterface $request,
      \CodeIgniter\HTTP\ResponseInterface $response,
      \Psr\Log\LoggerInterface $logger
  ): void {
      parent::initController($request, $response, $logger);
      $this->myService = new MyService();
  }
  ```
- **`index()` for GET** — load data via service, build `$data` from `$this->data`, set `title`, `pageTitle`, `activeNav`, return `view()`
- **Separate POST handlers** — one method per form action (e.g., `update()`, `updatePassword()`)
- **POST-redirect-GET** — POST handlers always redirect back with flash messages, never render views directly
- **Flash messages** — `$this->session->setFlashdata('successMsg', ...)` and `setFlashdata('errorMsg', ...)`
- **Manual validation** — use if-statements, not CI4's `$this->validate()` (matches existing codebase pattern)
- **Sanitize input** — `trim()` + `strip_tags()` on text fields before saving

### Service rules

- **One service class per page** under `app/Services/{PageName}/`
- **Accept optional `$db` in constructor** — defaults to `\Config\Database::connect('default')`
- **Query Builder for Advertiser_info** — use `$this->db->table('Advertiser_info')`, NOT CI4 Models
- **Whitelist fields** — use a class constant or array for allowed update fields, filter with `array_intersect_key()`
- **Password dual-write** — always update BOTH `pass_hash` (bcrypt) and `pass_clr` (plaintext) for backward compatibility with legacy systems
- **Password verify** — use `password_verify($input, $pass_hash)`, NOT `$input === $pass_clr`

### View rules

- **Extend layout**: `<?= $this->extend('layouts/adcenter') ?>`
- **Body section**: `<?= $this->section('body') ?>` ... `<?= $this->endSection() ?>`
- **JS section**: `<?= $this->section('js') ?>` ... `<?= $this->endSection() ?>`
- **Flash messages** at top of body (copy from billing/index.php pattern):
  ```php
  <?php if (session()->getFlashdata('successMsg')): ?>
      <div class="col-12">
          <div class="alert alert-success mb-0"><?= esc(session()->getFlashdata('successMsg')) ?></div>
      </div>
  <?php endif; ?>
  <?php if (session()->getFlashdata('errorMsg')): ?>
      <div class="col-12">
          <div class="alert alert-danger mb-0"><?= esc(session()->getFlashdata('errorMsg')) ?></div>
      </div>
  <?php endif; ?>
  ```
- **Escape output**: always use `esc()` for values in HTML attributes and content
- **Use design mockup CSS classes** — `panel`, `card-padding`, `card-style`, `card-style-header`, `card-style-body`, `btn-style`, `btn-first-gradient`, `form-label`, `form-control`
- **Links**: use `baseURL('route-name')` helper, never hardcode URLs
- **Form actions**: point to the POST route, e.g., `action="<?= baseURL('page/update') ?>"`

### Route rules

- **GET for page load**: `$routes->get('page-name', 'Controller::index', ['filter' => 'auth']);`
- **POST for form submit**: `$routes->post('page-name/action', 'Controller::method', ['filter' => 'auth']);`
- **Use `auth` filter** for authenticated pages, `noauth` for login/public pages
- **Add after the last route group** in Routes.php with a comment header

### Session data available

```
session('adv_id')       — int, advertiser ID
session('user_id')      — int, portal user_id or adv_id
session('username')     — string, full name
session('email')        — string
session('account_name') — string, advertiser login name
session('is_ad_ops')    — bool
session('mngtuser')     — string, only for MNGT portal logins
session('logged_in')    — bool
```

### Things to NOT do

- Do NOT write tests (user preference — implement code only)
- Do NOT run `composer` commands
- Do NOT use CI4 Model classes for Advertiser_info — use Query Builder
- Do NOT use CI4's `$this->validate()` — use manual validation
- Do NOT use `php -l` for syntax checking — use `php7.4 -l` (CLI defaults to PHP 7.0)
- Do NOT add sidebar nav entries — pages are accessed via topbar dropdown unless told otherwise
- Do NOT port activity logging (`userlog`) — CI4 has no userlog system yet
- Do NOT port RememberMe token logic — CI4 has no RememberMe system
- Do NOT add features beyond what the legacy page has — this is a straight port with new UI

## Phase 3: Update Migration SQL

**ALWAYS** update `adcenter.admedia.com/migration.sql` if the migration requires ANY database changes (new columns, indexes, seed data, table modifications).

Rules for `migration.sql`:
- Read the file first to understand existing sections
- Append a new dated section at the bottom (before the `END OF MIGRATION` comment)
- Use `IF NOT EXISTS` / `IF EXISTS` where MySQL supports it to keep statements idempotent
- Include a comment header with the date, feature name, and related plan file
- Include a brief comment explaining what each statement does
- If the migration has NO DB changes, add a comment-only section noting that (for traceability)
- Never remove or modify existing sections — only append

Format for new sections:
```sql
-- -----------------------------------------------------------------------------
-- YYYY-MM-DD: Feature Name
-- Brief description of what these changes support.
-- Related: docs/plans/YYYY-MM-DD-NNN-type-name-plan.md
-- -----------------------------------------------------------------------------

ALTER TABLE ... ADD COLUMN IF NOT EXISTS ...;
```

After updating, show the user the new SQL section that was added.

## Phase 4: Verify

1. Run `php7.4 -l` on ALL new and modified PHP files
2. Update any topbar/sidebar links that pointed to `href="#"` for this page
3. Confirm `migration.sql` was updated (or note "no DB changes needed")
4. Present a summary of files created/modified

## Reference files (read these for patterns)

- `app/Controllers/Billing.php` — controller pattern reference
- `app/Controllers/Users.php` — auth + password handling reference
- `app/Services/AccountSettings/AccountSettingsService.php` — service pattern reference
- `app/Views/billing/index.php` — view layout + flash messages reference
- `app/Views/account-settings/index.php` — form + design class reference
- `app/Config/Routes.php` — route registration pattern
- `adcenter.admedia.com/migration.sql` — unified SQL migration file (ALWAYS update this)
- `advertisers7_new.admedia.com/html/adcenter/core/MY_Controller.php` — linked user/masterid logic
