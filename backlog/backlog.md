# QDVC Alignment Backlog

Actionable tasks to bring the existing QDVC apps into line with
`QDVC_APP_SPEC.md`. The spec itself is reference-only; this file is the
task list. Section numbers below refer to the spec.

The five apps assessed: `qdvc-bibliotheca`, `qdvc-countdowns`, `qdvc-equip`,
`qdvc-logbook`, `qdvc-markdown-notebook`.

---

## Cross-cutting: Application ID (spec §7.1)

The canonical id scheme is **`qdvc.<App>`**. Every app that currently uses a
different scheme MUST be migrated. Note that changing a `Gtk.Application` /
`Adw.Application` id affects single-instance uniqueness and any existing
`.desktop` / dconf associations, so sequence this carefully and update the
`.desktop` file and any stored settings in the same change.

- [ ] **bibliotheca** — change `org.qdvc.Bibliotheca` → `qdvc.Bibliotheca`.
- [ ] **countdowns** — change `org.qdvc.Countdowns` → `qdvc.Countdowns`.
- [ ] **logbook** — change `org.qdvc.Logbook` → `qdvc.Logbook`.
- [ ] **equip** — add an id `qdvc.Equip` (currently sets a prgname only, no
      application id).
- [ ] **markdown-notebook** — add an id `qdvc.MarkdownNotebook` (currently sets a
      prgname only, no application id).

---

## Per-app tasks

### qdvc-bibliotheca

Closest to canonical already. Remaining items:

- [ ] Application id → `qdvc.Bibliotheca` (see cross-cutting, §7.1).
- [ ] Move `MAINTENANCE.md` and `MAINTENANCE_GTK3_GTK4.md` from the repo root
      into a `docs/` folder (§12); update the README links.

### qdvc-logbook

Also close to canonical. Remaining items:

- [ ] Application id → `qdvc.Logbook` (see cross-cutting, §7.1).
- [ ] Move `MAINTENANCE.md` and `MAINTENANCE_GTK3_GTK4.md` from the repo root
      into a `docs/` folder (§12); update the README links.
- [ ] Confirm the keyboard-shortcuts source of truth is a single shared
      `SHORTCUTS` table in the pure `ui_prefs.py` feeding both
      `gtk3_shortcuts.py` and `gtk4_shortcuts.py` (§10); centralise if the two
      modules currently duplicate the definitions.

### qdvc-equip

- [ ] Rename the package folder `qdvcequip_lib/` → `qdvc/` (§2); update all
      imports.
- [ ] Move the flat `gtk3_*.py` / `gtk4_*.py` modules into nested `qdvc/gtk3/`
      and `qdvc/gtk4/` sub-packages (§2); fix intra-package imports to
      `from ..` / `from .gtk3_x`.
- [ ] Replace the positional `gtk3`/`gtk4` argument with the `--gtk3`/`--gtk4`
      flag + `ui_backend` config dispatcher (§3).
- [ ] Add an application id `qdvc.Equip` (§7.1).
- [ ] Add a single shared `SHORTCUTS` table in `ui_prefs.py` and drive both
      front-ends from it (§10).
- [ ] (Docs already live under `docs/` — no change needed for §12.)

### qdvc-countdowns

- [ ] Rename the package folder `qdvc_countdowns_lib/` → `qdvc/` (§2); update
      all imports.
- [ ] Convert config storage from `config.ini` → `config.yml` (§5); include a
      one-time read of any existing `.ini` if in-place migration of user
      settings is desired.
- [ ] Application id → `qdvc.Countdowns` (see cross-cutting, §7.1).
- [ ] Create a `docs/` folder and add `MAINTENANCE.md` there (§12).
- [ ] Add a parallel GTK 4 / libadwaita front-end in `qdvc/gtk4/` (§9); when
      added:
  - [ ] add nested `qdvc/gtk3/` + `qdvc/gtk4/` sub-packages (§2),
  - [ ] add the `--gtk3`/`--gtk4` flag + `ui_backend` dispatcher (§3),
  - [ ] add `docs/MAINTENANCE_GTK3_GTK4.md` (§12),
  - [ ] add the shared `SHORTCUTS` table and GTK4 shortcuts window (§10).

### qdvc-markdown-notebook

- [ ] Rename the package folder `qdvcmdnb_lib/` → `qdvc/` (§2); update all
      imports.
- [ ] Add an application id `qdvc.MarkdownNotebook` (§7.1).
- [ ] Create a `docs/` folder and add `MAINTENANCE.md` there (§12).
- [ ] Add a parallel GTK 4 / libadwaita front-end in `qdvc/gtk4/` (§9); when
      added:
  - [ ] add nested `qdvc/gtk3/` + `qdvc/gtk4/` sub-packages (§2),
  - [ ] add the `--gtk3`/`--gtk4` flag + `ui_backend` dispatcher (§3),
  - [ ] add `docs/MAINTENANCE_GTK3_GTK4.md` (§12),
  - [ ] add the shared `SHORTCUTS` table and GTK4 shortcuts window (§10).
- [ ] (Config is already `config.yml`; the existing custom icon-set support
      already satisfies the optional-custom-icon part of §7.3.)

---

## Alignment matrix

Legend: ✓ = already conforms; blank cell = task listed above.

| Task (spec §) | bibliotheca | logbook | equip | countdowns | markdown-notebook |
| --- | --- | --- | --- | --- | --- |
| Package folder `qdvc/` (§2) | ✓ | ✓ | rename | rename | rename |
| Nested `gtk3/`+`gtk4/` dirs (§2) | ✓ | ✓ | flatten→nest | when GTK4 added | when GTK4 added |
| `--gtk3/4` flag + `ui_backend` (§3) | ✓ | ✓ | positional→flag | add | add |
| Config `config.yml` (§5) | ✓ | ✓ | ✓ | `.ini`→`.yml` | ✓ |
| App id `qdvc.<App>` (§7.1) | rename | rename | add | rename | add |
| Docs in `docs/` (§12) | move | move | ✓ | create | create |
| GTK 4 front-end (§9) | ✓ | ✓ | ✓ | add | add |
| Shared `SHORTCUTS` table (§10) | ✓ | centralise | add | when GTK4 added | when GTK4 added |
