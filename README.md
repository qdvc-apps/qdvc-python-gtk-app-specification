# QDVC Python desktop applications for Linux (GTK3/GTK4) - common specification

A shared specification for building desktop applications in the QDVC family
using **Python 3 + GTK 3** as the primary target, with a **parallel GTK 4 /
libadwaita** implementation for future-proofing.

This document distils the common patterns of the QDVC reference apps into one
canonical design, so that new apps can be built quickly and consistently and so
that all apps present a coherent backend architecture and user experience. It is
**reference material** describing the target design; it does not track the state
of any individual app. Per-app migration tasks are recorded separately in
`BACKLOG.md`.

Throughout, `<app>` is the short app name (e.g. `logbook`), `<App>` is its
title-case form (e.g. `Logbook`), and the repository is `qdvc-<app>`.

## Requirement keywords

The key words **MUST**, **MUST NOT**, **REQUIRED**, **SHALL**, **SHALL NOT**,
**SHOULD**, **SHOULD NOT**, **RECOMMENDED**, **MAY**, and **OPTIONAL** in this
document are to be interpreted as described in RFC 2119. In brief: **MUST** /
**REQUIRED** / **SHALL** denote an absolute requirement; **MUST NOT** / **SHALL
NOT** an absolute prohibition; **SHOULD** / **RECOMMENDED** (and **SHOULD NOT**)
denote a requirement that may be departed from only with a good, documented
reason; **MAY** / **OPTIONAL** denotes a genuinely optional item.

---

## 1. Design philosophy

These principles are shared by every app and constrain everything below.

**Files are the database.** An app MUST NOT use SQLite, an ORM, or any binary
database for its business data. All business data MUST be stored as plain-text,
human-readable, diff-able files in a *workspace* folder. The app is a
viewer/editor over those files plus an in-memory index. Any external edit to the
files MUST be recoverable by rescanning.

**Separation of concerns (pure core vs. toolkit views).** The application MUST
split into a *pure* layer that imports no GTK, and one *view* layer per toolkit.
The pure layer holds the model, file I/O, naming rules, formatting, and settings
access, and MUST be unit-testable without a display. A module that imports `gi`
is a view module and MUST live in a toolkit view package; a module that does not
is a pure module and MUST live in the top-level package. The two view layers
MUST NOT import each other.

**Two front-ends, one core.** GTK 3 is the primary/default front-end and MUST
adopt a classic GNOME2 / MATE-era look. A parallel GTK 4 / libadwaita front-end
SHOULD be provided and MUST follow the GNOME Human Interface Guidelines (HIG).
Both front-ends MUST sit on the same pure core and MUST NOT reimplement model,
file, naming, or formatting logic. Toolkit-independent UI logic (config reads,
spec parsing, the shortcuts table, report formatting) MUST live in a shared pure
module that both front-ends call.

**Lazy where it counts.** Opening a large workspace SHOULD stay fast: a cached
index SHOULD hold only lightweight display fields, and heavy parsing SHOULD
happen per-record, on demand. The index MUST be disposable and regenerable.

---

## 2. Repository & source layout

The canonical layout is:

```
qdvc-<app>/
    qdvc_<app>.py              Thin entry point / backend dispatcher (root).
    qdvc/                       PURE package — no GTK imports anywhere here.
        __init__.py            Version + app constants (APP_ID, APP_NAME, __version__).
        config.py              Load/save YAML config at the XDG location.
        workspace.py           Model aggregate (the Workspace class + index).
        models.py              Dataclasses for the domain records.
        naming.py              id / slug / filename helpers.
        ui_prefs.py            Pure, toolkit-independent UI helpers shared by
                               BOTH front-ends (SHORTCUTS table, sort specs,
                               validation-report formatting, …).
        platform_utils.py      Launch system apps (viewer, editor, file manager).
        <domain>.py            Additional pure modules as the app needs
                               (formatting, import/export, calendar, …).
        gtk3/                  GTK3 front-end sub-package — every module gtk3_*.
            __init__.py
            gtk3_app.py        Gtk.Application subclass; prgname + icon.
            gtk3_main_window.py    MainWindow: menubar + toolbar + content + statusbar.
            gtk3_preferences.py    Preferences dialog (incl. backend selector).
            gtk3_shortcuts.py      GTK3-specific shortcut wiring (see §9).
            gtk3_*.py          Tabs, dialogs, widgets, highlighters, …
        gtk4/                 GTK4 / libadwaita front-end sub-package — every module gtk4_*.
            __init__.py
            gtk4_app.py        Adw.Application; registers win.* accelerators.
            gtk4_window.py     Main window (Adw.ViewStack + header bars + primary menu).
            gtk4_actions.py    Installs the win.* Gio.SimpleActions.
            gtk4_preferences.py    Adw.PreferencesWindow (live-apply + selector).
            gtk4_shortcuts.py      Shortcuts window built from ui_prefs.SHORTCUTS.
            gtk4_*.py          Views, dialogs, factories, …
    docs/
        MAINTENANCE.md         Architecture, module layout, data formats.
        MAINTENANCE_GTK3_GTK4.md   Element-by-element GTK3↔GTK4 comparison.
    README.md                  User-facing: what it does, install, .desktop setup.
    qdvc-<app>.desktop         (Optional) ready-made launcher to copy & edit.
    .gitignore
```

Rules:

- The substantive code MUST live in a subfolder named exactly `qdvc/` (not
  `qdvc_<app>_lib/` or similar).
- GTK 3 and GTK 4 code MUST live in nested `qdvc/gtk3/` and `qdvc/gtk4/`
  sub-packages, and every module within them MUST be prefixed `gtk3_` / `gtk4_`
  respectively.
- The root entry point MUST be named `qdvc_<app>.py` (underscores).
- View modules MUST reach pure modules with `from ..`
  (e.g. `from ..workspace import Workspace`) and reach siblings with
  `from .gtk3_x` / `from .gtk4_x`.

---

## 3. Entry point & backend dispatch

The root `qdvc_<app>.py` MUST be a thin dispatcher that selects the UI toolkit
*before importing any GTK*, so that only the chosen front-end is loaded.

The backend selection order MUST be:

1. An explicit CLI flag `--gtk3` or `--gtk4` (consumed wherever it appears in
   argv);
2. the `ui_backend` key saved in the config;
3. the default `gtk3`.

The dispatcher:

- MUST preserve `argv[0]` in the argv handed to the backend (GApplication
  expects a program name there);
- MUST pass through all other arguments (e.g. a workspace path) to the backend's
  `main()`;
- SHOULD, if the GTK 4 import fails (e.g. libadwaita absent), print a note to
  stderr and fall back to GTK 3;
- MUST expose the backend selector from both UIs' Preferences, with changes
  taking effect on the next launch.

`Config.ui_backend` MUST validate and lower-case the stored value and MUST fall
back to `gtk3` for anything unrecognised, so a hand-edited config can never
leave the dispatcher without a valid backend.

---

## 4. Runtime requirements

- The app MUST target **Python 3.10+** (union syntax `str | None`, `list[...]`
  generics).
- The app MUST require **PyGObject** and exactly **one** of the two toolkits at
  runtime:
  - GTK 3 (`python3-gi`, `gir1.2-gtk-3.0`) — the default front-end; or
  - GTK 4 + libadwaita (`gir1.2-gtk-4.0`, `gir1.2-adw-1`) — the modern
    front-end.
- The app MUST require **PyYAML** (config and most data files are YAML).
- Any additional third-party library SHOULD be optional: the app SHOULD guard
  its import and provide a built-in fallback so it runs without it.

---

## 5. Configuration (preferences data)

- Preferences MUST be stored as **YAML** at
  `$XDG_CONFIG_HOME/qdvc-<app>/config.yml`, falling back to
  `~/.config/qdvc-<app>/config.yml`.
- `config.py` MUST be a thin wrapper exposing `get(key, default)` /
  `set(key, value)` over a `DEFAULTS` dict. Because every read supplies a
  default, the app MUST NOT require a schema migration when a new key is added.
- The config MUST hold application preferences only and MUST NOT hold business
  data. Typical keys: `last_workspace`, `recent_workspaces`, `window` (size),
  `reopen_last`, plus app-specific and UI keys (`toolbar_style`, `ui_backend`,
  fonts, …).
- Any key with a constrained value set SHOULD have a validated accessor
  (e.g. `Config.ui_backend`, `Config.toolbar_style`).

---

## 6. Business data — the workspace

- All domain data MUST live in a user-chosen **workspace folder** of
  plaintext-serialisable files. The format SHOULD be chosen per data need:
  - **YAML** (`.yml`) for structured records with a small, stable schema;
  - **Markdown** (`.md`) with **YAML frontmatter** for free-text content plus
    structured metadata;
  - **CSV** (`.csv`) for flat tabular/log data;
  - domain-specific text formats where they are the natural fit (e.g. BibTeX
    `.bib`, iCalendar `.ics`).
- The app MAY maintain a disposable index/cache file (e.g. `.qdvc-index.json`)
  holding lightweight display fields for fast load. Such a cache MUST be safe to
  delete and regenerate, and MUST carry an `INDEX_VERSION` that MUST be bumped
  whenever the cached per-record schema changes.
- Writes MUST be atomic (write a temp file, then `os.replace`).
- Where two writers touch the same file (e.g. free-text body vs. frontmatter),
  the app MUST define and preserve a strict write ordering (flush the body
  before rewriting frontmatter) so they do not clobber each other.
- The model SHOULD provide a `validate()` method returning categorised problem
  lists (orphans, dangling references, missing linked files, …) that the UI can
  render as a report.

---

## 7. Application identity, icon & desktop integration

### 7.1 Application ID

- Every app MUST define an application id of the form **`qdvc.<App>`**
  (e.g. `qdvc.Logbook`), exposed as `APP_ID` in `qdvc/__init__.py` and used for
  the `Gtk.Application` / `Adw.Application` id.

### 7.2 Program name / WM_CLASS

- At startup the app MUST call `GLib.set_prgname("qdvc-<app>")` so the X11
  `WM_CLASS` matches the `.desktop` `StartupWMClass`. This is the load-bearing
  line that lets the MATE/GNOME panel associate the running window with its
  launcher.

### 7.3 Icon policy

- The app MUST use, by default, a standard freedesktop themed icon name present
  on a typical GNOME/MATE install (e.g. `accessories-dictionary`,
  `appointment-soon`, `package-x-generic`), and MUST NOT require a bundled image
  file for that default case.
- The app MUST set the themed icon as both the app-wide default and the
  per-window icon:
  - `Gtk.Window.set_default_icon_name(ICON_NAME)` (GTK3 `do_startup`); and
  - `self.set_icon_name(ICON_NAME)` on the main window, so the icon shows before
    any `.desktop` matching.
- The app SHOULD allow a user-selected custom icon (a preference pointing at an
  absolute `.png`/`.svg`) and MUST NOT require one.
- The themed name MUST be defined once as `ICON_NAME` in `gtk3_app.py` (and the
  matching GTK4 module) so it is trivial to override.

### 7.4 `.desktop` launcher

The README MUST document a launcher installed at
`~/.local/share/applications/qdvc-<app>.desktop`, and the app MAY ship a
ready-made copy:

```ini
[Desktop Entry]
Type=Application
Name=QDVC <App>
Comment=<one-line description>
Exec=python3 /full/path/to/qdvc_<app>.py %U
Path=/full/path/to
Icon=<themed-icon-name>
Terminal=false
Categories=<freedesktop categories>;
StartupNotify=true
StartupWMClass=qdvc-<app>
```

- `StartupWMClass=qdvc-<app>` MUST equal the `set_prgname` value.
- `Exec` MUST be an absolute path; `Path` MUST be the directory containing the
  entry point and the `qdvc/` package. The entry SHOULD use `%U`/`%F` so a
  workspace folder can be dropped on the launcher.
- The README SHOULD include the refresh/validate commands
  (`update-desktop-database`, `desktop-file-validate`).

---

## 8. GTK 3 front-end design (primary)

The GTK 3 UI MUST adopt a classic GNOME2 / MATE-era look: title bar, menubar,
toolbar, content area, status bar — top to bottom.

**Application & window bootstrap.** `gtk3_app.py` MUST define a
`Gtk.Application` (id `qdvc.<App>`, `HANDLES_OPEN` where a workspace path is
accepted); `do_startup` MUST set the default icon; `do_activate` MUST build the
single main window; `do_open` MUST route a folder argument.
`GLib.set_prgname(...)` MUST run at import.

**Menubar.**
- The menubar SHOULD use the canonical top-level menus **File, Edit, View,
  Tools, Help**, in that order, including only those the app needs; an app MAY
  add a domain menu (e.g. *Task*).
- **Edit → Preferences** MUST be the home of the preferences command.
- Menu items SHOULD carry icons where appropriate. Items with icons MUST be
  built as a `Box(Image + Label)` inside a plain `Gtk.MenuItem` and MUST NOT use
  `Gtk.ImageMenuItem` (removed in GTK4, warns in GTK3). The app SHOULD provide a
  shared helper (e.g. `_menu_item(label, icon)`).
- Help SHOULD include an About item and MAY include an entry opening the
  shortcuts reference.

**Toolbar.**
- The toolbar MUST be a subset of the menubar's most-used actions.
- The app MUST provide a `toolbar_style` preference switching between
  labels-below-icons and labels-beside-icons, and MUST apply it live from
  Preferences.

**Status bar.** The window SHOULD carry a `Gtk.Statusbar` (or equivalent) at
the bottom for transient status.

**Action sensitivity.** Enable/disable logic SHOULD be centralised in one method
(e.g. `_update_actions_sensitivity`). Workspace-scoped actions MUST track
whether a workspace is open; context-scoped actions MUST recompute on the
relevant selection/tab-switch signals. Where a command appears in both menu and
toolbar, both widgets MUST be toggled together.

---

## 9. GTK 4 / libadwaita front-end design (parallel)

The GTK 4 UI MUST follow the GNOME HIG and MUST reuse the pure core unchanged;
only the view mechanics differ. The full element-by-element map MUST be recorded
in `docs/MAINTENANCE_GTK3_GTK4.md`. The key substitutions:

- The entry point MUST be an **`Adw.Application`**, and accelerators MUST be
  registered via `set_accels_for_action("win.<name>", [...])`.
- Commands MUST use the **`Gio.Action`** system: each command MUST be a
  `Gio.SimpleAction` under the `win.` scope, and menu items and header buttons
  MUST reference it by name, so one action drives every surface and
  `set_action_enabled(name, bool)` greys every bound item and its shortcut at
  once.
- The GTK 4 UI MUST NOT use a menubar or toolbar. Instead it MUST use a single
  **`Adw.HeaderBar`** and a **primary menu** (`Gtk.MenuButton`,
  `open-menu-symbolic`, `set_primary(True)`) whose `Gio.Menu` MUST end with a
  **Preferences / Keyboard Shortcuts / About** section per the HIG; the most
  common actions SHOULD be promoted to header buttons.
- A multi-view app MUST use an **`Adw.ViewStack`** + **`Adw.ViewSwitcher`** in
  place of a `Gtk.Notebook`.
- Preferences MUST be an **`Adw.PreferencesWindow`** with live-apply (no
  Save/Cancel). The toolbar-style control MUST be omitted (there is no toolbar);
  the backend selector MUST be present as an `Adw.ComboRow` with a "takes effect
  after restart" subtitle.
- The UI MUST NOT rely on `dialog.run()`: every modal flow MUST be asynchronous
  with a continuation callback (`Adw.MessageDialog` / `Gtk.FileDialog` /
  `Adw.Window` + `on_*` callback).
- Lists and tables SHOULD use `Gtk.ColumnView` / `Gtk.ListView` over a
  `Gio.ListStore` of row GObjects, with `Gtk.SignalListItemFactory`,
  `Gtk.FilterListModel` + `Gtk.CustomFilter`, and `Gtk.SingleSelection`.

---

## 10. Keyboard shortcuts (shared)

- The shortcut set MUST be defined once in the pure layer — a `SHORTCUTS` table
  in `ui_prefs.py` — and both front-ends MUST be driven from it, so GTK 3 and
  GTK 4 stay consistent.
- In GTK 3, shortcuts MUST be menubar-driven: accelerators attached via a
  `Gtk.AccelGroup` and shown next to their menu items, with window-level
  bindings (e.g. F2) via `accel_group.connect`. `gtk3_shortcuts.py` MUST wire
  these from the shared table.
- In GTK 4, the app MUST provide a keyboard-shortcuts window
  (`gtk4_shortcuts.py`) built from the same `SHORTCUTS` table, plus
  `set_accels_for_action` registration.
- Toolkit-specific exceptions are permitted: where a shortcut is meaningful in
  only one toolkit, it MUST be marked as GTK3-specific or GTK4-specific in the
  shared table and handled accordingly, and the app MUST NOT invent artificial
  parity.
- The app SHOULD use common defaults across the family: e.g. `Ctrl+O` open
  workspace, `Ctrl+Q` quit, `Ctrl+,` preferences, `Alt+1..N` view switching,
  `F2` rename.

---

## 11. Cross-platform helpers

The app SHOULD provide a pure `platform_utils.py` for launching system apps,
branching on `sys.platform` / `os.name`:

- `open_with_default_app(path)` — open in the default viewer;
- `open_with_text_editor(path)` — open in the default editor
  (`xdg-open` on Linux, `open -t` on macOS);
- `reveal_in_file_manager(path)` — honour a configurable `file_manager`
  template with `{dir}`/`{file}` placeholders.

---

## 12. Documentation

Every app MUST ship two maintenance documents inside a `docs/` folder, plus a
root README:

- **`docs/MAINTENANCE.md`** MUST cover design philosophy, runtime requirements,
  directory & file layout, data formats, the model/load pipeline, query &
  mutation API, validation, the UI layer, testing approach, a "common
  maintenance tasks — where to touch" section, and deployment.
- **`docs/MAINTENANCE_GTK3_GTK4.md`** MUST provide the element-by-element
  GTK3↔GTK4 comparison and the list-model/data-binding cheat-sheet. An app that
  has not yet gained a GTK 4 front-end MAY omit this file until it does.
- **`README.md`** (root) MUST be user-facing: what the app does, requirements,
  install & run (including the `--gtk4` / Preferences backend switch), and the
  `.desktop` launcher instructions.

---

## 13. Testing without a display

Because CI/sandbox environments often lack a GTK runtime, the app SHOULD support
two display-free techniques:

1. **Model tests** run directly against the pure layer — building a temp
   workspace on disk and asserting on `Workspace` behaviour (derivation, import,
   rename, validate, relative paths) with no display.
2. **Import smoke test** — a permissive fake `gi` stub (returning a catch-all
   object for any attribute) lets every module under `qdvc/gtk3/` and
   `qdvc/gtk4/` be imported, exercising class bodies, `__gsignals__`
   definitions, and top-level code. Because such a stub cannot evaluate real
   enum values, any enum-derived constant (e.g. a Pango style int) MUST be
   computed through a guarded helper rather than at class scope.

The minimum pre-ship check SHOULD be:

```sh
python3 -m py_compile qdvc/*.py qdvc/gtk3/*.py qdvc/gtk4/*.py qdvc_<app>.py
# then the stub-import of every qdvc.* module
```

with a cross-check that every `self.<tab>.<method>` call has a matching
definition and every `.connect("signal")` has a declared `__gsignals__` entry.

---

## 14. Adding a feature — the parity rule

When adding a user-facing feature:

1. Toolkit-independent logic (config reads, formatting, spec parsing, shortcut
   entries) MUST be placed in the pure layer (`ui_prefs.py` or a domain module)
   so both front-ends call one implementation.
2. The view change SHOULD be implemented in both `gtk3_*` and `gtk4_*`; if one
   toolkit is excepted, the reason MUST be documented (commit/PR note).
3. Where a command is involved, it MUST be added to the GTK 3 menu/toolbar and a
   corresponding `win.*` `Gio.SimpleAction` MUST be installed in `gtk4_actions`
   and referenced by name from the GTK 4 menu/header; its accelerator MUST be
   added to the shared `SHORTCUTS` table.
4. Sensitivity MUST be wired into the GTK 3 `_update_actions_sensitivity` and the
   GTK 4 action-enabled updates.

---

## 15. New-app checklist

To scaffold a new `qdvc-<app>`:

- [ ] Root `qdvc_<app>.py` thin dispatcher (`--gtk3`/`--gtk4` → `ui_backend` →
      `gtk3`), preserving `argv[0]`, with GTK4→GTK3 fallback.
- [ ] `qdvc/` pure package: `__init__.py` (`APP_ID = "qdvc.<App>"`, `APP_NAME`,
      `__version__`), `config.py` (YAML at `~/.config/qdvc-<app>/config.yml`),
      `workspace.py`, `models.py`, `naming.py`, `ui_prefs.py` (incl.
      `SHORTCUTS`), `platform_utils.py`.
- [ ] `qdvc/gtk3/` (all `gtk3_*`): `gtk3_app.py`, `gtk3_main_window.py`
      (menubar File/Edit/View/[Tools]/Help with icons; Edit→Preferences;
      toolbar subset with `toolbar_style` below/beside), `gtk3_preferences.py`,
      `gtk3_shortcuts.py`, plus tabs/dialogs/widgets.
- [ ] `qdvc/gtk4/` (all `gtk4_*`): `gtk4_app.py`, `gtk4_window.py`,
      `gtk4_actions.py`, `gtk4_preferences.py` (Adw, live-apply, backend
      selector), `gtk4_shortcuts.py`, plus views/dialogs/factories.
- [ ] `GLib.set_prgname("qdvc-<app>")`; themed `ICON_NAME` set as default +
      per-window; optional custom-icon preference.
- [ ] Workspace on disk: plaintext files (YAML / Markdown+frontmatter / CSV /
      domain formats), atomic writes, disposable index with `INDEX_VERSION`,
      `validate()`.
- [ ] `docs/MAINTENANCE.md` + `docs/MAINTENANCE_GTK3_GTK4.md`; root `README.md`
      with `.desktop` block (`StartupWMClass=qdvc-<app>`).
- [ ] Display-free tests: pure-model tests + fake-`gi` import smoke test;
      `py_compile` all modules.
