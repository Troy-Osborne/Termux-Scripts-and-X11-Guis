# Toolbar Manager for Termux X11

A small Tkinter application that renders one or more always-on-top toolbars
at the edges of your X11 screen. Each button runs a shell/Python action,
copies text to the clipboard, or launches an app — all driven by JSON files
you can edit without touching code.

This package ships with:

- `install.sh` — one-shot installer (safe to re-run; wipes previous install)
- `tm` — launcher (installed into `$PREFIX/bin` or `$HOME/bin`)
- `set-mode` — switch between `default` / `decoy` / `personal` toolbars
- `summary.log` — debug build: full install report in your home directory
- `~/.toolbar-manager/runtime.log` — debug build: per-session crash log

---

## 1. Requirements

| Requirement | Install command |
|---|---|
| Termux (or any bash + Python) | — |
| Python 3 | `pkg install python` |
| Tkinter | `pkg install python-tkinter` |
| An X server (Termux X11 / VNC / XSDL) | `pkg install x11-repo termux-x11-nightly` |
| Clipboard backend *(at least one)* | `termux-api` (for `termux-clipboard-set`), or `xclip`, or `xsel` |

```bash
pkg install python python-tkinter termux-api
```

---

## 2. Installation

```bash
# 1. Save the installer somewhere, e.g. ~/install-toolbar.sh
chmod +x ~/install-toolbar.sh

# 2. Start X11 first (the installer warns if DISPLAY is unset)
termux-x11 :0 &
export DISPLAY=:0

# 3. Run it
~/install-toolbar.sh
```

### Installer flags

```
./install-toolbar.sh            # default: debug build (writes summary.log)
./install-toolbar.sh --no-debug # release build: no summary.log, quieter output
./install-toolbar.sh --quiet    # suppress progress lines on stdout
./install-toolbar.sh --help     # show usage
```

You can also control the debug flag via environment variable:

```bash
TM_DEBUG=0 ./install-toolbar.sh
```

### What it does

1. Probes the environment (user, home, shell, `DISPLAY`, `PREFIX`, python version).
2. Checks prerequisites (`python3`, `tkinter`, `DISPLAY`). Missing hard deps → abort.
3. **Wipes** any previous install at `~/.toolbar-manager` and any prior `tm` launcher.
4. Writes `main.py`, configs, action scripts, launcher, and mode helper.
5. Verifies every file (non-empty, valid Python syntax, valid JSON, executable bits).
6. Writes `~/summary.log` (debug only) with the full report.

### Where things get installed

```
~/.toolbar-manager/
├── main.py              # the application
├── mode                 # current mode string ("default" / "decoy" / "personal")
├── runtime.log          # created on first error (debug build)
├── config/
│   ├── toolbars.json    # master list of which toolbars to show per mode
│   ├── toolbar_top.json
│   ├── toolbar_right.json
│   ├── toolbar_bottom.json
│   └── toolbar_decoy.json
├── scripts/             # shell actions each button can call
│   ├── app_terminal.sh
│   ├── app_files.sh
│   ├── project_a.sh
│   └── …
└── bin/
    └── set-mode

$PREFIX/bin/tm           # launcher (or ~/bin/tm if PREFIX is unset)
~/summary.log            # debug-build install report
```

---

## 3. Running it

Make sure X11 is running and `DISPLAY` is exported, then:

```bash
tm
```

Or directly:

```bash
python3 ~/.toolbar-manager/main.py
```

The toolbars appear immediately. They are borderless, always-on-top, and
positioned per their JSON config. Close them by killing the process:

```bash
pkill -f 'toolbar-manager/main.py'
```

To run on login, add this to `~/.bashrc` or `~/.profile` (Termux-X11 only):

```bash
[ -n "$DISPLAY" ] && ! pgrep -f toolbar-manager/main.py >/dev/null && tm &
```

---

## 4. Switching modes

Three modes ship by default:

| Mode | Toolbars shown |
|---|---|
| `default` | top, right, bottom |
| `personal` | top, right, bottom |
| `decoy` | decoy only |

To switch at runtime:

```bash
~/.toolbar-manager/bin/set-mode decoy
```

`main.py` polls the `mode` file every ~0.8 s and rebuilds the toolbars
whenever it changes. You can also echo directly:

```bash
echo personal > ~/.toolbar-manager/mode
```

No restart of `main.py` is needed.

---

## 5. Configuration

### 5.1 `config/toolbars.json` — master list

```json
{
  "mode": "default",
  "toolbars": [
    { "file": "toolbar_top.json",    "modes": ["default", "personal"] },
    { "file": "toolbar_bottom.json", "modes": ["default", "personal"] },
    { "file": "toolbar_decoy.json",  "modes": ["decoy"] }
  ]
}
```

- `mode` is the **default** mode if `~/.toolbar-manager/mode` doesn't exist.
- `modes` lists which runtime modes display this toolbar. Use `["*"]` to
  show in every mode.

### 5.2 Per-toolbar files

```json
{
  "name": "top",
  "width": 900,
  "height": 44,
  "anchor": "top-center",
  "x_offset": 0,
  "y_offset": 0,

  "bg": "#1e1e2e",
  "fg": "#cdd6f4",
  "border_color": "#45475a",
  "border_width": 1,

  "layout": "horizontal",
  "button_bg": "#313244",
  "button_active_bg": "#585b70",
  "button_pad": 3,
  "font_family": "Monospace",
  "font_size": 10,

  "buttons": [
    { "label": "Term",    "flag": "app_terminal.sh" },
    { "label": "Browser", "flag": "app_browser.sh" },
    { "label": "Email",   "flag": "copy:me@example.com" }
  ]
}
```

**Fields**

| Key | Meaning |
|---|---|
| `anchor` | `top-left`, `top-center`, `top-right`, `left-center`, `right-center`, `bottom-left`, `bottom-center`, `bottom-right` |
| `x_offset` / `y_offset` | pixels added to the anchor position (may be negative) |
| `layout` | `horizontal` or `vertical` — direction buttons pack |
| `border_width` | inset of the inner frame; `0` for a flat look |
| `width_chars` (per-button) | fixed width in characters; omit for auto |

---

## 6. Flags — what a button does when clicked

The `flag` string determines the action. Prefix decides the behavior:

| Flag form | Effect |
|---|---|
| `app_terminal.sh` | If `~/.toolbar-manager/scripts/app_terminal.sh` exists, run it with `sh`. |
| `shell:<command>` | Run the raw command with `sh -c`. |
| `py:<path>` | Run the file with the same Python interpreter. |
| `copy:<text>` | Copy `<text>` to the X11 clipboard. |

**Newlines in `copy:`** — JSON string `\n` is literal, so
`"copy:Best regards,\n-- Me"` becomes a two-line clipboard value.

**Clipboard backends** — `main.py` tries in order: tkinter's X11 clipboard,
then `termux-clipboard-set`, then `xclip`, then `xsel`. Install at least
one for reliable behavior when tkinter's clipboard ownership is lost
(e.g. app closed by `kill -9`).

---

## 7. Logs & debugging

### `~/summary.log` (debug build, install time)

Written only when `DEBUG=1` (the default). Contains:

- environment snapshot
- per-step status/duration
- full manifest of every file written with byte counts
- aggregated errors and warnings

Delete it any time — it is not read by the app.

### `~/.toolbar-manager/runtime.log` (debug build, run time)

Appended on every warning, error, or crash. Empty if nothing has failed.
Look here first if toolbars disappear, a button stops responding, or
clipboard copy silently fails.

Tail it while diagnosing:

```bash
tail -f ~/.toolbar-manager/runtime.log
```

### Switching summary.log off in future releases

Once you're satisfied, either:

- run `./install-toolbar.sh --no-debug`, or
- edit `DEBUG=${TM_DEBUG:-1}` → `DEBUG=${TM_DEBUG:-0}` at the top of
  `install-toolbar.sh`.

The runtime logger always stays on — it's the crash breadcrumb trail.

---

## 8. Uninstall

```bash
rm -rf  ~/.toolbar-manager
rm -f   "$PREFIX/bin/tm"  ~/bin/tm
rm -f   ~/summary.log
```

Nothing else is written anywhere.

---

## 9. Troubleshooting

**"tkinter missing"** — run `pkg install python-tkinter`. The installer
aborts cleanly and does not leave partial state.

**"DISPLAY not set"** — X11 isn't running or isn't exported. Start it:

```bash
termux-x11 :0 &
export DISPLAY=:0
```

**Toolbars don't appear, but `tm` runs** — check the log:

```bash
tail -n 40 ~/.toolbar-manager/runtime.log
```

Common causes: a JSON file is malformed (the loader logs the exact path),
a toolbar references a nonexistent file, or `DISPLAY` changed between
launch and GUI creation.

**Buttons do nothing** — the flag was dispatched but the script errored.
Inspect `runtime.log`. For `shell:` actions you can add debugging:

```json
{ "label": "Test", "flag": "shell:echo clicked >> /tmp/tm-test.log" }
```

**Clipboard copies nothing** — install at least one backend:
`pkg install termux-api`, or `pkg install xclip`. The logger records
each backend that failed.

**Mode switch not taking effect** — confirm the watcher is alive:

```bash
ps aux | grep toolbar-manager
cat ~/.toolbar-manager/mode
ls  -la ~/.toolbar-manager/mode   # check mtime changes when you set-mode
```

**Re-running the installer wipes my custom toolbars** — that's intentional;
re-run is "clean slate". Back up `~/.toolbar-manager/config` and
`~/.toolbar-manager/scripts` first:

```bash
cp -r ~/.toolbar-manager/config  ~/tm-config-backup
cp -r ~/.toolbar-manager/scripts ~/tm-scripts-backup
```

Then restore after reinstall.

---

## 10. Quick reference

```bash
# first install
chmod +x install-toolbar.sh
termux-x11 :0 &
export DISPLAY=:0
./install-toolbar.sh

# every-day use
tm
~/.toolbar-manager/bin/set-mode personal

# diagnose
tail -f ~/.toolbar-manager/runtime.log
less    ~/summary.log

# reinstall cleanly
./install-toolbar.sh

# uninstall
rm -rf ~/.toolbar-manager "$PREFIX/bin/tm" ~/bin/tm ~/summary.log
```
