# Module Structure Reference

Use when creating or restructuring a module's files and directories. The canonical layout (Odoo 19 Coding Guidelines) — match it exactly. Filenames use `[a-z0-9_]` only. Directories `755`, files `644`.

## Manifest (`__manifest__.py`)

The keys a PSDU customer module is expected to carry:

| Key | Convention |
|---|---|
| `name` | MMC convention `<Customer> <Project> \| <Domain>`, e.g. `Phoenix Fashion \| Product` |
| `summary` | one real line of what it does (not a restatement of `name`) |
| `description` | meaningful; cite the `project.task` id(s) the work traces to |
| `license` | `OEEL-1` for customer code — **not** `LGPL` |
| `version` | full `<series>.<x.y.z>`, e.g. `18.0.1.0.0`; bump the last segments when shipping a migration |
| `depends` | smallest set that works (see the "`depends` is a contract" stance) |
| `data` | every XML/CSV the module ships, in load order; demo files go in `demo` |
| `installable` | `True`; `auto_install` only for glue/bridge modules |

```python
{
    "name": "Phoenix Fashion | Product",
    "summary": "Product extensions for the Phoenix Fashion catalogue",
    "description": "Implements task-2451234: size/colour variants on the website.",
    "version": "18.0.1.0.0",
    "license": "OEEL-1",
    "depends": ["product"],
    "data": [
        "security/ir.model.access.csv",
        "views/product_template_views.xml",
    ],
    "demo": ["data/product_demo.xml"],
    "installable": True,
}
```

## Directory layout

| Dir | Purpose |
|---|---|
| `data/` | Demo and data XML (split by purpose) |
| `models/` | Model definitions (one file per main model) |
| `controllers/` | HTTP routes |
| `views/` | Backend views and templates |
| `static/` | Web assets (css, js, img, lib, scss, xml) |
| `security/` | Access rights, groups, record rules |
| `wizard/` | TransientModels and their views |
| `report/` | Printable + SQL-view-based reports |
| `tests/` | Python tests |

## File naming

### `models/`
One file per main model, named after the model. Inherited models get their own file named after the inherited model. One model (or one wizard) per file — don't pile several into one `.py`.

```
models/
├── plant_nursery.py       # main model: plant.nursery
├── plant_order.py         # main model: plant.order
└── res_partner.py         # inherited from base
```

### `__init__.py`
One import per line (PEP 8). No `from . import a, b, c`, no loops over module names — both hurt diffs and cause merge conflicts.

```python
from . import models
from . import wizard
```

### `security/`
Three files, fixed names:

```
security/
├── ir.model.access.csv             # access rights
├── <module>_groups.xml             # res.groups
└── <model>_security.xml            # ir.rule per main model
```

### `views/`
Backend views suffixed `_views.xml`. Templates (QWeb portal/website) suffixed `_templates.xml`. Optional `<module>_menus.xml` for top-level menus.

```
views/
├── plant_nursery_menus.xml         # optional, main menus only
├── plant_nursery_views.xml
├── plant_nursery_templates.xml
├── plant_order_views.xml
└── res_partner_views.xml
```

### `data/`
Split by purpose. `_data.xml` for production data, `_demo.xml` for demo only.

```
data/
├── plant_nursery_data.xml
├── plant_nursery_demo.xml
└── mail_data.xml                   # data targeting another module
```

### `controllers/`
One controller per file. The module's main controller is `<module_name>.py`. Inheriting a controller from another module: name the file after the source module.

```
controllers/
├── plant_nursery.py                # the module's own controller
└── portal.py                       # inherits portal/controllers/portal.py
```

`main.py` is deprecated — don't create new files with that name.

### `wizard/`
Same as models, but co-located with their views.

```
wizard/
├── make_plant_order.py
└── make_plant_order_views.xml
```

### `report/`
Two distinct kinds:

**Stats reports** (Python + SQL view + classic views):
```
report/
├── plant_order_report.py
└── plant_order_report_views.xml
```

**Printable reports** (data prep + QWeb templates):
```
report/
├── plant_order_reports.xml         # report actions, paperformat
└── plant_order_templates.xml       # QWeb report templates
```

### `static/`
```
static/
├── lib/<libname>/                  # third-party libs (don't link, copy in)
├── src/
│   ├── js/
│   │   └── tours/                  # end-user tutorials, NOT tests
│   ├── scss/
│   ├── xml/                        # QWeb templates rendered in JS
│   └── img/
└── tests/
    └── tours/                      # tour tests
```

Never reference external URLs for assets — copy the file into `static/`.

## Complete example

```
addons/plant_nursery/
├── __init__.py
├── __manifest__.py
├── controllers/
│   ├── __init__.py
│   ├── plant_nursery.py
│   └── portal.py
├── data/
│   ├── plant_nursery_data.xml
│   ├── plant_nursery_demo.xml
│   └── mail_data.xml
├── models/
│   ├── __init__.py
│   ├── plant_nursery.py
│   ├── plant_order.py
│   └── res_partner.py
├── report/
│   ├── __init__.py
│   ├── plant_order_report.py
│   ├── plant_order_report_views.xml
│   ├── plant_order_reports.xml
│   └── plant_order_templates.xml
├── security/
│   ├── ir.model.access.csv
│   ├── plant_nursery_groups.xml
│   ├── plant_nursery_security.xml
│   └── plant_order_security.xml
├── static/
│   ├── lib/
│   └── src/{js,scss,xml,img}/
├── views/
│   ├── plant_nursery_menus.xml
│   ├── plant_nursery_views.xml
│   ├── plant_nursery_templates.xml
│   ├── plant_order_views.xml
│   └── res_partner_views.xml
└── wizard/
    ├── make_plant_order.py
    └── make_plant_order_views.xml
```

## Modifying existing files

- **Stable version:** never restyle. Original file style supersedes guidelines. Keep diff minimal.
- **Master:** apply guidelines only when most of the file is being revised. Move-commit first, then changes.
