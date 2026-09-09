# AGENTS.md — Guidelines for Coding Agents Working on This Repository

This repository is the **distribution channel** of **Building** («باني
المشاريع»), an Arabic Linux `.deb` desktop tool (PyGObject/GTK3) that turns a
project folder into a distributable artifact: APK / AAB / IPA / Windows EXE /
MSI / `.deb` / AppImage. It detects the project type from its own files,
generates the exact toolchain command, streams a live log with a REAL
progress bar, and ships an update center whose server is this repository.

Follow the rules below exactly. They are inherited from the author's
field-tested PadLink conventions; do not "modernize" them away.

## 1. Repository layout (read this first)

- The git repo tracks **ONLY**: the **single latest** `.deb` + `update.json`
  + this `AGENTS.md`. Nothing else (no source, no assets, no datasets).
  (The professional PNG icon lives outside git in the workspace; the package
  ships a compact SVG so the deb stays small.)
- The **source of truth** is the working tree extracted from the deb:

  ```sh
  cd /home/user && rm -rf building-extract && mkdir building-extract
  dpkg-deb -x building_<ver>-1_amd64.deb building-extract
  dpkg-deb --ctrl-tarfile building_<ver>-1_amd64.deb | tar -x -C building-extract/DEBIAN
  ```

- Main files (inside the package):
  - `usr/lib/building/building.py` — the whole app: pure-stdlib project
    detection, command generation, build runner, real progress bar,
    update engine, centre-of-screen notification, doctor, error reporting.
    **Pure ASCII source** (Arabic lives ONLY in `ui_ar.json`).
  - `usr/share/building/ui_ar.json` — the ONLY Arabic source. Flat JSON,
    **1-space indent**, `ensure_ascii=False`. Every user-facing string here.
  - `usr/share/building/toolchains.json` — pinned "latest known good"
    toolchain versions (data, not code). An update may refresh it alone.
  - `usr/bin/building` — `exec python3 /usr/lib/building/building.py "$@"`.
  - `DEBIAN/control` — `Version: X.Y.Z-1` must match `APP_VERSION` in
    building.py and `update.json`.

## 2. Release flow (every single change ships through this)

1. Bump in **all three places**: `APP_VERSION` (header comment too),
   `DEBIAN/control`, `update.json` (version + deb filename + note).
2. Build:
   ```sh
   cd /home/user/building-extract && find . -name __pycache__ -exec rm -rf {} +
   cd /home/user && dpkg-deb --build --root-owner-group building-extract building/building_<new>-1_amd64.deb
   ```
3. Verify by re-extracting and asserting on extracted files (versions, ui_ar
   key count, toolchains pins, no `__pycache__`) + run the test suites.
4. Git (branch `arena/01a087ba-building`, never switch branches):
   ```sh
   git add building_<new>-1_amd64.deb update.json AGENTS.md
   git rm building_<old>-1_amd64.deb     # "كل تحديث احذف النسخة السابقة"
   git commit && git push origin arena/01a087ba-building
   ```
5. GitHub: `gh release delete <old> --yes --cleanup-tag`, then
   `gh release create <new> --target arena/01a087ba-building` with Arabic
   notes containing the **pinned raw link at the NEW commit SHA**.
6. Verify the blob via `curl -H "Accept: application/vnd.github.raw"` → 200 +
   exact byte size; verify `update.json` content.
7. Delete ALL loose debs from /home/user except the one inside the repo.
8. Reply to the user in **Arabic** with ONE direct clickable link +
   `sudo apt install ~/building_<new>-1_amd64.deb`.

**Invariants:** exactly ONE release and ONE deb at any time; the repo must
never contain two versions; the pinned link must use the commit SHA.

## 3. Update server (auto-update) — do not regress this

- Manifest: `update.json` at repo root, fetched by `UPDATE_ENDPOINTS` in
  order: raw CDN **with per-check cache-buster** (`?t={ts}`) → github.com raw
  redirector → API contents (base64) → raw CDN plain. Headers
  `Cache-Control: no-cache` / `Pragma: no-cache`.
- The user's network **cannot reach api.github.com** (ISP/proxy); his proven
  route is the github.com raw redirector → raw.githubusercontent.com. Never
  reduce to a single api.github.com endpoint.
- `DEB_ENDPOINTS` mirrors the same order (`{deb}` placeholder).
- `upd_parse_manifest` tolerates raw JSON and API base64 `content`.
- The "update available" alert is a **centre-of-screen** card (`notify_center`
  → `_center_on_screen`, undecorated + keep-above, moved to the primary
  monitor centre; no RGBA / compositor) titled `up_found` = «يتوفر تحديث».
- Update progress is **REAL**: downloaded/Content-Length bytes during the
  download and apt's own `APT::Status-Fd` percentages during install. The bar
  only **pulses** when no real number exists — never an invented percentage.
- Download is capped (400 MB), truncation-checked and cancellable; the file is
  verified as a Debian `!<arch>` archive before `pkexec apt-get install`.

## 4. Real progress (builds AND updates) — do not regress this

`parse_progress(line)` returns a fraction ONLY from a percentage the toolchain
printed itself (Gradle/Flutter/MSBuild/apt). Otherwise the build bar pulses.
The equivalent command is always shown in the Build page **before** running
(full transparency) and `missing_tools` is reported so the user knows which
toolchain to install.

## 5. Testing (all tests must be green before shipping)

- Suites: `/home/user/building-tests/test_v200.py` (pure engine: detection,
  command generation, legacy compatibility, validation, real-progress parsing,
  toolchain pins, update engine) and `test_ui200.py` (the real UI under the
  `gtk_shim.py` fake `gi`). `/tmp` is wiped by resets; keep suites in
  `/home/user/building-tests` (outside git).
- Run:
  ```sh
  python3 /home/user/building-tests/test_v200.py
  python3 /home/user/building-tests/test_ui200.py
  python3 usr/lib/building/building.py --doctor      # headless toolchain report
  python3 usr/lib/building/building.py --selftest <dir>   # detect + commands
  ```
- Harness pattern: import `building.py` for the module-level engine (it stays
  importable without `gi`); the UI is exercised by installing the fake `gi`
  from `gtk_shim.py`. Add/update tests for every code change.

## 6. Error reporting (automatic)

`report_error(ctx, msg)`: 24 h sha1 dedupe (`~/.config/building/reported.txt`),
body = last 40 log lines + platform, tries `gh api` issue → `0x0.st` paste →
always a local `error_report_*.txt`, toasts the user. Real failure paths
(build fail, update fail, thread exceptions) call it.

## 7. Conventions & commands

| Task | Command |
| --- | --- |
| Rebuild package | `dpkg-deb --build --root-owner-group building-extract building/building_<ver>-1_amd64.deb` |
| Extract for verify | `dpkg-deb -x <deb> verify && dpkg-deb --ctrl-tarfile <deb> \| tar -x -C verify/DEBIAN` |
| Byte-compile | `python3 -m py_compile usr/lib/building/*.py` |
| Headless doctor | `python3 usr/lib/building/building.py --doctor` |
| Detect + commands | `python3 usr/lib/building/building.py --selftest <dir>` |
| Regenerate Arabic | `python3 /home/user/building-tests/gen_locale.py` |
| ui_ar key audit | compare all `tr("...")` literals against the JSON |
| Release list | `gh release list` (must show exactly ONE) |
| Repo debs | `git ls-files '*.deb'` (must show exactly ONE) |

## 8. Do NOT

- Do not add files to git except the single latest deb, `update.json` and
  this `AGENTS.md`. Keep generated artifacts out of git.
- Do not switch branches: this session is fixed to `arena/01a087ba-building`.
- Do not reintroduce: single-endpoint update checks, an update alert not
  centred on the screen, an invented/estimated progress percentage, Arabic
  strings inside `.py`, two releases/debs in the repo, a >2 MB icon in the deb.
- Do not remove the toolchain doctor, the command preview, the route-status
  table, or the automatic error reporting.
- Do not reply to the user in Arabic with more than ONE direct download link
  per release.
