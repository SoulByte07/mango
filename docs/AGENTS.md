# Mango Wayland Compositor — Agent Instructions

## Project Overview
Mango is a Wayland compositor written in C, built on **wlroots 0.20** and **scenefx 0.5**. It forked from **dwl** and adds tags, animations, multiple layouts, window effects, IPC, and hot-reload config.

Three binaries are produced:
- `mango` — the compositor (src/main.c entry point)
- `mmsg` — IPC messaging client (mmsg/mmsg.c)
- `mangonag` — layer-shell notification daemon (mangonag/mangonag.c)

## Build System (Meson)

```bash
# Configure (out-of-tree build)
meson setup build

# Compile
meson compile -C build

# Install (prefix from meson_options.txt)
meson install -C build

# Run from build dir (requires XDG_RUNTIME_DIR)
build/mango

# Check config only (exit code 0 = valid)
build/mango -p
```

### Meson Options
| Option | Type | Default | Notes |
|--------|------|---------|-------|
| `xwayland` | feature | enabled | X11 app support |
| `asan` | boolean | false | AddressSanitizer |

### Key Dependencies (verify with `meson setup build`)
- wlroots-0.20 (≥0.20.0)
- scenefx-0.5 (≥0.5.0)
- wayland-server/client
- libinput (≥1.27.1)
- xkbcommon, libdrm, pixman, pcre2, cjson, pangocairo
- XWayland deps: xcb, xcb-icccm, xcb-randr (if xwayland enabled)

## Development Workflow

### Nix Flake (preferred for dev)
```bash
nix develop          # enters dev shell with clang-tools
nix build            # builds via flake
```

### Formatting
```bash
./format.sh          # clang-format LLVM style, tabs=4, sorts includes
```
`.clang-format`: `BasedOnStyle: LLVM`, `UseTab: Always`, `TabWidth: 4`, `IndentWidth: 4`, `SortIncludes: true`

### Source Layout
```
src/                    # compositor sources (auto-discovered by meson.build)
  main.c                # entry point, setup/run/cleanup
  common/               # server, log, util, scene_node
  config/               # TOML parsing, watcher, hot-reload, conflict check
  layout/               # tile, scroller, dwindle, monocle, grid, deck, etc.
  manage/               # client, monitor, layer, tab, xwayland_primary
  input/                # keyboard, pointer, touch, tablet, trackpad, switch
  animation/            # client, layer, tag, common
  draw/                 # text-node, dim-node, title_message_bar
  dispatch/             # bind (key/mouse/gesture/axis handling)
  ext-protocol/         # custom wayland protocols
  ipc/                  # mmsg IPC server
include/mango/          # public headers mirroring src/
protocols/              # wayland protocol XML (generated at build)
mmsg/                   # IPC client binary
mangonag/               # notification daemon binary
assets/                 # default configs, desktop file, session target
nix/                    # nix modules, options generation
```

### Config System
- Format: **TOML** (`config.toml`)
- Locations (in order): `$XDG_CONFIG_HOME/mango/config.toml` → `/etc/mango/config.toml`
- Hot-reload: `SUPER+r` (default) or `mmsg reload_config`
- Check syntax: `mango -p` (exits 0 on success)
- Config watcher uses inotify via `config_watcher.c`

### Running / Debugging
```bash
# Requires XDG_RUNTIME_DIR set (Wayland requirement)
export XDG_RUNTIME_DIR=/run/user/$(id -u)

# Debug logging
mango -d

# Custom config
mango -c /path/to/config.toml

# Startup command (e.g., for panels)
mango -s "waybar & swaync &"
```

### Key Conventions
- **Tags, not workspaces**: each tag maintains independent layout (9 default)
- **Layouts**: tile, scroller, grid, deck, monocle, center_tile, vertical_tile, vertical_scroller
- **Window states**: swallow, minimize, maximize, global, overlay, fakefullscreen
- **IPC**: Unix socket at `$XDG_RUNTIME_DIR/mango-<instance>/mango.sock` — use `mmsg` to send commands
- **Animations**: open/close/move/tag/fade with configurable curves (cubic-bezier)
- **XWayland**: enabled by default, scale support without blur

### Common Tasks
| Task | Command |
|------|---------|
| Add new layout | Create `src/layout/<name>.c/.h`, add to `src/layout/arrange.c`, register in `src/manage/client.c` |
| Add config option | Define in `src/config/option_defs.c`, parse in `src/config/value_parse.c`/`toml_parse.c` |
| Add IPC command | Extend `src/ipc/ipc.c` handlers, update `mmsg/mmsg.c` CLI |
| Add Wayland protocol | Add XML to `protocols/`, meson.build auto-generates headers |

### Testing
No automated test suite. Manual verification:
- `mango -p` — config syntax check
- `meson compile -C build` — compiles cleanly
- Run compositor in nested Wayland (e.g., `wlr-randr` + `mango` inside sway/Hyprland)

### CI / Release
- No GitHub Actions in repo (check `.github/` — empty)
- Releases tagged manually; version in `meson.build` + git commit hash
- Packaging: Arch (AUR), Nixpkgs, Fedora, others via Repology

### Gotchas
- **Auto-source discovery**: `meson.build` uses `find` to collect `src/**/*.c` — new files picked up on reconfigure
- **Protocol headers**: Generated at build time in `build/`; `.clangd` suppresses "file not found" for layer-shell
- **Systemd integration**: `set_activation_env()` starts `mango-session.target` for portals/panels
- **NOFILE limit**: Raised to hard limit at startup (`increase_nofile_limit()`)
- **SIGPIPE ignored**: IPC send failures shouldn't crash compositor
- **Headless backend**: Always created as secondary for offscreen rendering

### Reference Docs
- Website: https://mangowm.github.io/ (config reference, keybindings, IPC, layouts)
- Wiki: https://github.com/mangowm/mango/wiki/
- `docs/` directory: installation, configuration, bindings, window-management, visuals, IPC