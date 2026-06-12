# Qt 6.8.3 upgrade — build instructions for the Windows agent

**Branch:** `dpi_fix` (build from this branch)
**Written:** 2026-06-13 by the WSL-side agent
**Status: BUILD OK — 6.8.3 exe built & launches, awaiting human multi-monitor test** ← update this line when you start/finish (see "Reporting back" at the bottom)

## Why

The dropdown-menus-detaching-from-the-menu-bar bug on multi-monitor systems is a
known Qt regression introduced in **exactly Qt 6.2.4** (the version we currently
build with): QTBUG-101203 — Windows screen manager treats two monitors with the
same device name (identical models, as on this machine) as one physical device.
Screen geometry for the "lost" monitor (#2, the left external) is then wrong and
every QMenu popup positions against the wrong screen. Fixed in Qt 6.2.5; we are
jumping to Qt 6.8.3 (LTS) which also includes later popup-placement fixes
(QTBUG-118695, QTBUG-120011) and the reworked Windows DPI handling.

CI workflows (`Build.yml`, `Release_tag_stable.yml`) are already bumped to 6.8.3
on this branch. Your job: reproduce that locally and produce a test build.

## Install the new kit (next to the existing `6.2.4\` one — do not delete it yet)

From `D:\GitHub\LibreVNA` in PowerShell or cmd:

```
pip install aqtinstall
python -m aqt install-qt windows desktop 6.8.3 win64_mingw -O D:\GitHub\LibreVNA
python -m aqt install-tool windows desktop tools_mingw1310 -O D:\GitHub\LibreVNA
```

This creates `D:\GitHub\LibreVNA\6.8.3\mingw_64\` and
`D:\GitHub\LibreVNA\Tools\mingw1310_64\`.

Qt 6.8 binaries are built with GCC 13.1 — the old `Tools\mingw1120_64`
compiler is too old for it. Use `mingw1310_64` for everything below.

## Copy libusb into the new kit

The .pro expects libusb headers in the Qt include dir (`LIBS += -lusb-1.0`,
library already at `Software\PC_Application\LibreVNA-GUI\libusb-1.0.a`).
The repo already has `libusb\` extracted at the root:

```
Xcopy /E /I /Y D:\GitHub\LibreVNA\libusb\include D:\GitHub\LibreVNA\6.8.3\mingw_64\include
```

(Mirrors what was done for the 6.2.4 kit and what CI does.)

## Build

```
set PATH=D:\GitHub\LibreVNA\6.8.3\mingw_64\bin;D:\GitHub\LibreVNA\Tools\mingw1310_64\bin;%PATH%
cd D:\GitHub\LibreVNA\Software\PC_Application\LibreVNA-GUI
mingw32-make distclean   (ignore errors if no previous build)
qmake LibreVNA-GUI.pro
mingw32-make -j8
```

Output: `Software\PC_Application\LibreVNA-GUI\release\LibreVNA-GUI.exe`.

Then deploy the Qt DLLs next to the exe so it runs standalone (and so the old
6.2.4 DLLs on PATH can't be picked up by mistake):

```
cd release
windeployqt.exe LibreVNA-GUI.exe
copy D:\GitHub\LibreVNA\6.8.3\mingw_64\bin\libwinpthread-1.dll .
copy D:\GitHub\LibreVNA\6.8.3\mingw_64\bin\libgcc_s_seh-1.dll .
copy D:\GitHub\LibreVNA\6.8.3\mingw_64\bin\libstdc++-6.dll .
copy D:\GitHub\LibreVNA\6.8.3\mingw_64\bin\Qt6OpenGL.dll .
```

**Important:** make sure `windeployqt` is the one from `6.8.3\mingw_64\bin`
(check with `where windeployqt`). Mixing 6.2.4 DLLs with a 6.8.3 exe will
crash at startup or silently reintroduce the bug we're testing for.

## Possible build issues

- Qt 6.8 is stricter about deprecated APIs than 6.2. If compilation fails,
  note the errors in this file (don't fix blindly) — the WSL-side agent will
  pick them up.
- If `aqt` can't find 6.8.3, run `python -m aqt list-qt windows desktop` and
  use the closest 6.8.x available; record which one you used.

## Verification (the actual point of all this)

The human will test, but sanity-check first:
1. Launch `release\LibreVNA-GUI.exe` (no device needed).
2. Help → About should report Qt 6.8.3.
3. Drag the window to monitor **#2** (left external — the bad one) and open
   File / View menus. Dropdowns should attach directly below the menu bar.
   On 6.2.4 they reliably detach on this monitor.

## Reporting back

Update the **Status** line at the top and append a section here:

```
## Build agent result (YYYY-MM-DD HH:MM)
- Qt version installed: ...
- Build: OK / FAILED (errors below)
- Menus on monitor #2: FIXED / STILL BROKEN / NOT TESTED
- exe at: ...
```

The WSL-side agent and the human are polling this file.

## Build agent result (2026-06-13 10:19)
- Qt version installed: 6.8.3 (win64_mingw, GCC 13.1 via tools_mingw1310). Kit at
  `D:\GitHub\LibreVNA\6.8.3\mingw_64`, compiler at `Tools\mingw1310_64`. Old 6.2.4
  kit left in place untouched.
- Build: **OK** — 0 errors, 5 pre-existing benign warnings (third-party
  `Util\QMicroz\miniz.c` -Wtype-limits x2, `tools.cpp` -Wparentheses, `scpi.cpp`
  -Wmaybe-uninitialized x2). None are Qt 6.8 deprecation breaks; code compiled
  clean against 6.8.3 with no source changes.
- Built from commit c00b4ef (branch dpi_fix). GITHASH embedded.
- Deployed: ran `windeployqt` from the 6.8.3 kit + copied libwinpthread-1.dll,
  libgcc_s_seh-1.dll, libstdc++-6.dll, Qt6OpenGL.dll. Verified
  `release\Qt6Core.dll` reports FileVersion 6.8.3.0, so the exe runs against 6.8.3
  (not the old 6.2.4 DLLs). `platforms\qwindows.dll` present.
- Smoke test: launches without crash; main window title "LibreVNA-GUI v1.6.4".
- Menus on monitor #2: **NOT TESTED** (needs human — left running for the test).
- exe at: `Software\PC_Application\LibreVNA-GUI\release\LibreVNA-GUI.exe`

Build notes for reproducing:
- `mingw32-make distclean` did NOT actually remove the stale 6.2.4 `.o`/exe
  (make then reported "Nothing to be done" and the old binary was kept). Had to
  delete `release\`, the Makefiles, and root-level moc_*/ui_*/*.o by hand before
  qmake+make produced a real 6.8.3 binary. If rebuilding, nuke `release\` first.
- libusb: Windows build uses `#include <libusb-1.0/libusb.h>`, so the header must
  be at `6.8.3\mingw_64\include\libusb-1.0\libusb.h` (the `Xcopy /E` of
  `libusb\include` places it there correctly — no need for a flat `include\libusb.h`).
