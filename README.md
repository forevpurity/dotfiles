# dotfiles

Personal dotfiles managed with [chezmoi](https://www.chezmoi.io).
Primary machine: EndeavourOS (Arch-based), niri/Wayland desktop.

This README is the **spec**: it records not just *what* the layout is but *why*,
so a fresh install (or future me) can rebuild and extend without re-deriving the
decisions.

## Philosophy

- **Single curated desktop, minimal moving parts.** No templating or secrets
  tooling until a real second machine forces the need. Complexity is added only
  when something concrete demands it.
- **chezmoi over stow** because the source of truth is *copies + explicit apply*,
  not a symlink farm. `$HOME` holds normal files; `chezmoi diff` shows exactly
  what an apply would change before it touches anything.
- **Distro-portable by structure, not by speculation.** The layout is ready for
  Fedora/Debian, but only the Arch package list is actually populated. Untested
  lists rot, so we don't write them until we boot that distro.
- **Config files carry values; this README carries the *why*.** Tracked configs
  stay comment-free, and record **only settings that differ from the app's own
  defaults** (check against its dump — e.g. `ghostty +show-config --default`).
  The two rules compose: every line in a config is then, by definition, a
  deliberate deviation, so the file needs no commentary to explain itself. A
  config that restates defaults reads like intent and drifts silently the day
  upstream changes its mind.

## Theming (deferred, on purpose)

Theme is currently **hardcoded** in `dot_config/ghostty/config` (`theme = Flexoki
Dark`). Ghostty is the only consumer today, and centralizing a value used in one
place means guessing a schema before its other users exist. Lifting it later is
cheap and local — rename `config` → `config.tmpl`, add the data file — so
deferring costs nothing.

**Lift it when the second consumer lands** (tmux or nvim). The shape should be
the *active theme only*, one entry per app:

```yaml
# home/.chezmoidata/theme.yaml
theme:
  ghostty: "Flexoki Dark"     # ghostty's own registry name
  nvim: flexoki-dark          # colorscheme name; also needs the plugin
  tmux: flexoki               # plugin name, or hex values
```

Swapping then means editing one file, and `apply` moves every app in lockstep —
that lockstep is the win, not the line count.

Two things that look tempting and are not:

- **A single theme string shared by all apps.** It doesn't propagate. The apps
  disagree on both name *and* representation: ghostty has a built-in registry
  (one line), nvim needs a *plugin* plus a `colorscheme` call, tmux wants hex or
  its own plugin. Only the decision is portable, never the value — hence a
  per-app mapping.
- **Centralizing the 16-color palette as hex and rendering everything from it.**
  Works for ghostty and tmux; dies at nvim, where a theme colors *hundreds* of
  semantic highlight groups (diagnostics, diff, treesitter, LSP). 16 ANSI colors
  can't express that, and hand-writing highlight groups loses to the plugin.
- **A matrix of every theme with an `active:` selector.** Same failure mode as
  the distro lists above: mappings for themes we don't run are untested, and
  untested lists rot.

## Layout

`.chezmoiroot` points at `home/`, so the repo root stays clean (README, license,
CI) and only `home/` is applied to `$HOME`.

```
~/.local/share/chezmoi/          # the source repo
├── .chezmoiroot                 # contains: home
├── README.md                    # this file (NOT applied — lives above home/)
└── home/                        # everything here maps into $HOME
    ├── .chezmoidata/
    │   └── packages.yaml         # package lists, keyed by distro family
    ├── .chezmoiscripts/          # scripts that run but do NOT create files in $HOME
    │   ├── run_once_before_00-install-paru.sh.tmpl
    │   ├── run_onchange_before_10-install-packages.sh.tmpl
    │   ├── run_onchange_after_20-enable-services.sh.tmpl
    │   └── run_once_after_30-set-shell.sh.tmpl
    ├── .chezmoiignore
    ├── .chezmoiexternal.toml     # assets fetched from URLs (fonts, pinned plugins)
    └── dot_config/               # → ~/.config/
        ├── niri/config.kdl
        ├── ghostty/config        # no extension — ghostty only reads `config`
        └── nvim/                 # hand-rolled, managed inline (see below)
```

## Naming cheat-sheet

The whole mental model is attribute prefixes on source filenames:

| Source name       | Becomes              | Use for                          |
|-------------------|----------------------|----------------------------------|
| `dot_config/`     | `~/.config/`         | everything under XDG             |
| `dot_zshrc`       | `~/.zshrc`           | top-level dotfiles               |
| `private_foo`     | `foo`, chmod 600     | anything holding a token/key     |
| `executable_foo`  | `foo`, +x            | scripts in `~/.local/bin`        |
| `foo.tmpl`        | templated `foo`      | only when a value differs per machine |
| `run_once_*`      | runs once ever       | one-time bootstrap               |
| `run_onchange_*`  | runs when contents change | data-driven actions         |

**nvim** is managed inline under `dot_config/nvim/` (hand-rolled config, not a
distro like LazyVim). Only split it into its own repo via `.chezmoiexternal` if it
grows its own life. Plugin/lockfile install dirs go in `.chezmoiignore` so chezmoi
never tracks downloaded plugins.

## Package management

`home/.chezmoidata/packages.yaml` holds package lists as structured template data,
keyed by distro **family** so it's portable:

```yaml
packages:
  arch:
    pacman:
      - niri
      - ghostty
      - neovim
      - fzf
      - fuzzel
    aur:
      - some-aur-thing
  fedora:
    dnf: []      # placeholder, not a promise — fill in only when we run Fedora
  debian:
    apt: []
```

Use **block style** (one item per line), not flow style (`[a, b, c]`): each
package add/remove becomes a clean one-line git diff, and individual entries can
carry a trailing `# comment`. Empty placeholders stay as `[]`.

The install script dispatches on the running distro. **Dispatch on
`.chezmoi.osRelease.idLike`, not `.id`** — see the gotcha below.

This list is **hand-curated, not generated.** Add a package by hand when you
deliberately adopt it; the file is an intentional manifest of chosen software, not
a snapshot of everything installed. To *audit* for drift — things installed but not
yet in the manifest — compare against `pacman -Qqe` (native) and `pacman -Qqm`
(AUR), but the manifest stays the source of truth.

### ⚠️ EndeavourOS gotcha

On EndeavourOS, `.chezmoi.osRelease.id` is `"endeavouros"`, **not** `"arch"`.
Writing `eq .chezmoi.osRelease.id "arch"` makes the install script silently do
nothing on this very machine. Dispatch on `.chezmoi.osRelease.idLike` (which *is*
`"arch"` on EndeavourOS/Manjaro, and `"debian"` on Ubuntu/Mint/Pop!_OS). Use `.id`
only to single out one specific distro from its family.

## Scripts (`.chezmoiscripts/`)

Scripts are the imperative escape hatch — **minimize them.** Before writing one,
check chezmoi can't do it declaratively (manage the file directly; fetch downloads
via `.chezmoiexternal`; parent dirs are auto-created).

### Which `run_` verb

| Verb             | Runs                               | Use for                                   |
|------------------|------------------------------------|-------------------------------------------|
| `run_once_`      | once per unique content hash, ever | true one-time bootstrap (paru, chsh)      |
| `run_onchange_`  | whenever (rendered) contents change| data-driven (package install, `fc-cache`) |
| `run_`           | every `apply`                      | almost never                              |

### Rules

- **Everything must be idempotent.** Any edit to a script re-fires it, so it must
  be safe to run twice. `pacman --needed` and `systemctl enable` already are;
  guard `chsh`/`mkdir`/symlinks explicitly.
- **Data in `.chezmoidata`, logic in the script.** Reference `.packages.arch.pacman`;
  don't hardcode lists. Keeps the rendered script stable so `onchange` only fires
  when the data actually changes.
- **Keep templated scripts deterministic.** The `onchange` hash is over rendered
  output — never interpolate timestamps/`now`, or the script runs every apply.

### `before_` / `after_` (relative to file application)

A script's attribute places it relative to when chezmoi writes your dotfiles:

- `run_before_` — runs before *any* files are written. Use when the rest of the
  apply depends on it (paru must exist; packages must be installed).
- `run_after_` — runs after *all* files are written. Use when it depends on a file
  chezmoi just deployed (enable a service whose unit override you just wrote).
- **neither** — a plain `run_` runs *interleaved* with file writes, ordered by
  path. Legal, but the interleave point is meaningless for `.chezmoiscripts`
  entries (they have no `$HOME` path to sit next to).

**Convention: in `.chezmoiscripts`, always tag `before_` or `after_`.** chezmoi
doesn't require it, but it makes the file-write boundary explicit. Decide by asking:
*does this script depend on a file the apply writes?* Yes → `after_`. No → `before_`.

### Ordering: numeric prefixes

chezmoi runs scripts in **alphabetical order of name** (within the `before_` group,
then files are written, then the `after_` group). Alphabetical rarely matches the
order you need — e.g. `install-packages` sorts before `install-paru` (`c` < `r`),
which would try to install AUR packages before the AUR helper exists.

Numeric prefixes override that with an explicit sequence you derive from
dependencies:

1. List the scripts.
2. Draw "must run before" arrows: `paru → packages → services`.
3. Lay them in a line respecting the arrows.
4. Number left-to-right in tens.

```
00-install-paru      # AUR helper must exist first
10-install-packages
20-enable-services   # packages must be installed first
30-set-shell         # independent — number is free
```

Only *relative* order matters (00/10/20 ≡ 100/200/300). Gaps of 10 leave room to
insert later without renaming. Where two scripts have no dependency, the number is
free — don't overthink it. Numbers only compete *within* the same `before_`/`after_`
group; the attribute controls the file-application boundary independently.

## Common commands

```bash
chezmoi edit ~/.config/niri/config.kdl   # edit the source copy
chezmoi diff                             # preview what apply would change
chezmoi apply                            # write changes into $HOME
chezmoi re-add <file>                    # capture a live $HOME edit back into source
chezmoi cd                               # drop into the source repo (then git ...)
```

**Source of truth is the source repo, not `$HOME`.** Live edits in `$HOME` aren't
captured until `chezmoi re-add`. Working loop: fiddle live → `re-add` → commit.

### Debugging scripts

```bash
chezmoi state delete-bucket --bucket=scriptState   # forget which scripts have run
chezmoi apply -v --dry-run                          # see what would fire, verbosely
```

## Bootstrap a fresh machine

```bash
sudo pacman -S --needed chezmoi
chezmoi init --apply <this-repo-url>
```
