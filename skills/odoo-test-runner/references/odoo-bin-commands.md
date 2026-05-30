# `odoo-bin` Command Reference

Source: Odoo 19 CLI docs. Only the commands and flags needed to **run a local DB, install a module, and test development work**. Other `odoo-bin` features (scaffold, populate, cloc, obfuscate, deploy, upgrade_code, i18n) are not in scope here.

Assumes you run from a checkout of Odoo Community where `./odoo-bin` exists. If you installed via package or Docker, swap the binary name accordingly.

## `db init` — create a fresh database

```bash
odoo-bin db init <database>
```

| Flag | Purpose |
|---|---|
| `--force` | Drop the DB first if it exists (use for `tmp_test_*` names; never in shared dev) |
| `--without-demo` (default) / `--with-demo` | Demo data control. Tests run **without** demo. |
| `--language <code>` | Default language (default `en_US`) |
| `--username <name>` / `--password <pw>` | Initial admin credentials (default `admin` / `admin`) |
| `--country <iso>` | ISO code for main company |

Installs only `base`. To install your module, use the server command with `-i` next.

## `db drop` — delete a database (and filestore)

```bash
odoo-bin db drop <database>
```

No interactive prompt. Removes the Postgres DB **and** its filestore directory. This is why you don't use raw `dropdb` — it would leave the filestore behind.

## `db duplicate` — copy a database (for safe experimentation)

```bash
odoo-bin db duplicate <source> <target>
```

| Flag | Purpose |
|---|---|
| `-n`, `--neutralize` | Neutralize the copy (clear cron emails, etc.) |
| `-f`, `--force` | Drop the target first if it exists |

Useful when you want to test a migration against a copy of a real DB without touching the original.

## `server` (default command) — install / update modules and run tests

```bash
odoo-bin -d <database> -i <modules> --stop-after-init
```

`server` is the default command — you omit it and pass options directly. The combination of `-i`, `--test-enable`, and `--stop-after-init` is the test invocation.

### Module options

| Flag | Purpose |
|---|---|
| `-d <db>` | Target database (required) |
| `-i <mod>[,<mod>...]` | **Install** these modules before running |
| `-u <mod>[,<mod>...]` | **Update** these modules (or `all` for all installed) |
| `--addons-path <dirs>` | Comma-separated paths to addons directories |
| `--stop-after-init` | Exit after install/update — the standard test pattern |
| `--without-demo` | Don't install demo data on new modules |

### Test options

| Flag | Purpose |
|---|---|
| `--test-enable` | Run tests after install/update |
| `--test-file <path>` | Run a single Python test file |
| `--test-tags <spec>` | Filter which tests to execute (see below) |
| `--screenshots <dir>` | Where to drop screenshots from failing `browser_js` tests |

### `--test-tags` filter syntax

Format: `[-][tag][/module][:class][.method]`, comma-separated for multiple specs.

| Spec | Meaning |
|---|---|
| `/psae_base` | All tests in module `psae_base` |
| `:TestInvoice` | All tests in class `TestInvoice` |
| `.test_compute_total` | All tests with method name `test_compute_total` |
| `/psae_base:TestInvoice.test_compute_total` | One specific test |
| `-/psae_base` | **Exclude** all tests in `psae_base` |
| `external` | Tests tagged with `@tagged('external')` |

Omitted include tag defaults to `standard`. Omitted exclude tag defaults to `*`.

### Logging for tests

| Flag | Purpose |
|---|---|
| `--log-level=test` | Standard level for test runs |
| `--log-level=debug` | Verbose — use when chasing a failure |
| `--log-handler odoo.models:DEBUG` | Per-logger debug (repeatable) |
| `--logfile <path>` | Send logs to a file instead of stderr |

## Database connection (override `~/.odoorc` if needed)

| Flag | `~/.odoorc` key | Default |
|---|---|---|
| `-r`, `--db_user <user>` | `db_user` | system user |
| `-w`, `--db_password <pw>` | `db_password` | empty |
| `--db_host <host>` | `db_host` | UNIX socket |
| `--db_port <port>` | `db_port` | `5432` |

Put these in `~/.odoorc` once. Don't repeat them in every command.

```ini
[options]
db_user = odoo
db_host = localhost
db_port = 5432
addons_path = /Users/assouma/work/odoo/addons,/Users/assouma/work/psae
```

## `shell` — REPL into the DB for verification

```bash
odoo-bin -d <database> shell
```

Boots Odoo as an interactive Python session with `env` pre-populated. Used for ORM-level verification after install (data shape, computed values, domain results, action effects).

| Flag | Purpose |
|---|---|
| `--shell-interface (ipython\|ptpython\|bpython\|python)` | Pick the REPL flavor |
| `--shell-file <script.py>` | Run a Python script in shell context after startup |

**Transactional by default.** Changes roll back when you exit unless you call `env.cr.commit()`. Useful when you want to poke at data without persisting accidents.

```python
# Inside the shell
>>> env['psae.task'].search([('state', '=', 'open')])
psae.task(7, 12, 18)
>>> env['psae.task'].browse(7).assignee_id.name
'Acme Customer'
>>> # Trigger a method
>>> env['psae.task'].browse(7).action_close()
>>> # Inspect the result (transaction still open — exit to roll back)
```

## Running the UI (for view / wizard / flow verification)

```bash
odoo-bin -d <database>
```

Starts the HTTP server on port `8069` (override with `-p <port>`). Default admin login: `admin / admin` (set by `db init`).

| Flag | Purpose |
|---|---|
| `-p`, `--http-port <port>` | Override default `8069` |
| `--no-http` | Don't start HTTP workers (irrelevant for UI verification) |
| `--workers <count>` | Multiprocessing mode (leave at default `0` for local verification) |
| `--dev xml,reload` | Hot-reload XML templates and Python files (don't combine with `--test-enable`) |

## Useful when debugging tests (`--dev`)

```bash
odoo-bin -d <db> --dev xml,reload
```

| Feature | Effect |
|---|---|
| `xml` | Read QWeb templates from disk, not DB (no re-install needed) |
| `reload` | Restart server when Python files change |
| `qweb` | Break on `t-debug='debugger'` in QWeb |
| `werkzeug` | Full traceback on the frontend on exceptions |
| `access` | Log traceback next to `AccessError` |
| `all` | Alias for `xml,reload,qweb,access` |

Don't use `--dev` in test runs — it changes behavior. Use it only when interactively debugging.

## Canonical test invocation

The full pattern, end-to-end, in one Bash call:

```bash
DB=tmp_test_<module>_$(date +%s) && \
  odoo-bin db init "$DB" --force --without-demo && \
  odoo-bin -d "$DB" \
    -i <module> \
    --test-enable \
    --stop-after-init \
    --without-demo \
    --log-level=test ; \
  odoo-bin db drop "$DB"
```

- `&&` between init and the test run — fail fast if init fails.
- `;` before drop — drop **always** runs, even if tests fail.
- Connection / addons-path come from `~/.odoorc`.

## Commands explicitly **not** covered here

These exist in `odoo-bin` but aren't part of the verification loop. If you reach for them, read the upstream docs:

- `scaffold` — generating a new module skeleton (use editor templates or copy an existing module)
- `populate` — duplicate existing records for load testing
- `cloc` — line-counting for maintenance pricing
- `obfuscate` — anonymize a DB before sharing
- `deploy` — push a module to a remote Odoo server
- `upgrade_code` — bulk source rewrites for major version upgrades
- `i18n` — translation import/export/load
- `neutralize` — strip outbound integrations from a restored DB
