# Finder

![Linux](https://img.shields.io/badge/platform-Linux-blue?style=for-the-badge&logo=linux)
![Rust](https://img.shields.io/badge/built_with-Rust-orange?style=for-the-badge&logo=rust)
![License](https://img.shields.io/badge/license-GPL--3.0-green?style=for-the-badge)
![AppImage](https://img.shields.io/badge/distro-AppImage-red?style=for-the-badge)

• Strategic Recommendation: Frames Lubuntu as the primary native development environment for guaranteed stability.
• Technical Breakdown: Bullet points highlighting the streamlined configuration, low resource footprint, and its ability to revive older hardware with zero lag.
• Veteran Insight: Concludes with an engaging nod to the fun of managing multiple Linux environments.

> A compact, dependency-light neon file search and full-binary duplicate finder for Linux.
> One AppImage. No daemon. No telemetry.

Finder is a single-binary desktop utility that answers two questions quickly: *where is this
file?* and *which files are byte-for-byte identical?* It ships as an icon-equipped AppImage
with a custom-drawn, borderless neon UI and runs on both X11 and Wayland.

## Highlights

- **Two modes, one binary** — ranked fuzzy name search (with Windows-style wildcards) and
  duplicate detection over complete file contents.
- **Content, not names** — candidates are pre-filtered by size, then verified by streaming
  BLAKE3 over every byte. A same-size, different-content file is never reported as a duplicate.
- **Windows-style globs** — `o*.deb`, `?pen.deb`, `*.log`. Literal and fuzzy matching remain
  available when no wildcard is present.
- **Actionable results** — every row exposes `LOCATION` (reveal parent) and `OPEN` (launch with
  the default handler). Duplicate rows add `DELETE`; duplicate groups add `DELETE REST` — both
  behind a confirmation dialog with recoverable Trash and permanent-delete options.
- **Low idle cost** — no resident indexer, no background service, no periodic repaint. Scanning
  runs on worker threads, hashing is capped at two workers, and cancellation is instant via
  `STOP SCAN` or `Esc`.
- **Portable** — built against glibc 2.17; X11, Wayland and GL are loaded at runtime. Works under
  UEFI/EFI or legacy boot, on older and current distributions, with no kernel modules.
- **Local only** — no network calls, no uploads, no telemetry, no account.


## Quick start

```bash
chmod +x Finder-1.0.0-x86_64.AppImage
./Finder-1.0.0-x86_64.AppImage
```

If FUSE is unavailable:

```bash
APPIMAGE_EXTRACT_AND_RUN=1 ./Finder-1.0.0-x86_64.AppImage
```

## Controls

| Control | Action |
|---|---|
| Yellow `−` (top left) | Minimize |
| Green `□` (top left) | Maximize / restore |
| Red `○` (top left) | Close |
| `CONTACT` (top right) | Open the author's repository page in the default browser |
| `DEEP FIND` | Ranked name-search mode |
| `DUPLICATE VAULT` | Full-binary duplicate mode |
| `BROWSE` | Folder picker (`kdialog` / `zenity` / `yad`) |
| `CLEAR` | Clear results in the active mode |
| `DELETE` / `DELETE REST` | Remove one duplicate / all copies except the first, after confirmation |
| Drop a folder | Set scan scope |
| `Enter` | Start scan from a focused field |
| `Esc` | Stop an active scan |

Window: 1000×660 default, 880×600 minimum, native decorations disabled.

## Search semantics

Query handling is ordered and deterministic:

1. **Wildcard present** (`*` or `?`) — glob match against the filename; fallback to the full
   path. `*` spans any sequence, `?` matches exactly one character.
2. **Exact name** → full score.
3. **Prefix** → high score, with a length penalty.
4. **Word boundary** — matches after `_`, `-`, `.` or space.
5. **Fuzzy subsequence** — non-adjacent characters in order.
6. **Path substring** → low score.

Examples: `report.pdf`, `rep`, `mnf` (matches `my_neon_finder.rs`), `o*.deb`, `?pen.deb`.

## Duplicate verification pipeline

1. Recursively index files under the scan root; skip virtual filesystems (`/proc`, `/sys`,
   `/dev`) and do not follow symlinks.
2. Group candidates by exact byte length. Files below `Minimum size (KB)` are ignored.
3. Stream every candidate through BLAKE3 using at most two workers.
4. Report only groups that share both size and full 256-bit digest. Reclaimable space is
   computed as `(copies - 1) × size`.

Deletion is explicit: nothing is removed until the confirmation dialog is accepted. `TO TRASH`
uses `gio trash`; `DELETE FOREVER` calls `remove_file` directly.

## Build from source

```bash
# Mandatory security gate: runs twice in the packaging script as well
cargo audit

cargo build --release --locked
./target/release/finder
```

Toolchain: Rust 2021 edition, `rust-version = "1.85"`.

```bash
cargo fmt --check
cargo clippy --all-targets -- -D warnings
cargo test --locked
```

`cargo audit` reports **zero vulnerabilities**. It emits one informational `unmaintained`
notice for `ttf-parser`, a transitive dependency of egui's font stack; no advisory describes a
vulnerability in it, and the command exits successfully.

## Packaging

```bash
./packaging/build_appimage.sh
```

The script:

1. Runs `cargo audit`.
2. Builds with `cargo zigbuild --target x86_64-unknown-linux-gnu.2.17` when Zig and
   `cargo-zigbuild` are on `PATH`; otherwise falls back to the host toolchain.
3. Runs `cargo audit` again immediately before packaging.
4. Assembles an AppDir with the binary, desktop entry, metainfo, 512×512 icon and licenses.
5. Produces `dist/Finder-1.0.0-x86_64.AppImage` and verifies its payload by extraction.

For maximum distro coverage, install Zig and `cargo-zigbuild` before building. Without them the
binary is linked against the host glibc.

## Project layout

```
src/
  main.rs       eframe bootstrap, viewport, embedded icon
  app.rs        state machine, layout, actions, confirmation flow
  scan.rs       search scoring + BLAKE3 duplicate engine
  platform.rs   default opener, folder dialog, trash/delete helpers
  theme.rs      fonts, palette, custom widgets, icons, glow effects
packaging/      AppRun, .desktop, metainfo, AppImage build script
assets/         PNG icon, SVG source, bundled Lato fonts
docs/           screenshot used by this README
```

## Design notes

- **On-demand traversal over persistent index** — zero idle cost and no stale database; the
  trade-off is a fresh walk per scan. Results are capped at 2000 displayed matches.
- **Size prefilter is an optimization, not the verdict** — only full digests decide duplicates.
- **Hidden files are included** — dotfiles and dot-directories are scanned like any other entry.
- **Deletion is guarded** — a modal requires an explicit choice; Trash is preferred, permanent
  delete is opt-in. Trash is unavailable on system mounts such as `/tmp`.
- **All visuals are vector-painted** — no runtime image assets beyond the embedded icon.

## License

MIT for the application. Bundled Lato fonts are licensed under the SIL Open Font License 1.1;
see [THIRD_PARTY_LICENSES.md](THIRD_PARTY_LICENSES.md).
