```
███████╗ █████╗ ███████╗██████╗
██╔════╝██╔══██╗██╔════╝██╔══██╗
█████╗  ███████║███████╗██║  ██║
██╔══╝  ██╔══██║╚════██║██║  ██║
██║     ██║  ██║███████║██████╔╝
╚═╝     ╚═╝  ╚═╝╚══════╝╚═════╝
```

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![zshrs plugin](https://img.shields.io/badge/zshrs-native%20plugin-blue.svg)](https://github.com/MenkeTechnologies/zshrs)

### `[FILE + DIRECTORY FRECENCY — COMPILED]`

> *"a, s, d, f, j — the whole filesystem, ranked by frecency, in Rust."*

## `[NATIVE ZSHRS PLUGIN]`

[fasd](https://github.com/clvv/fasd) — frecency for **files and directories** (it tracks every path argument of every command, so `f <part>` finds a recently-used file and `j <part>` cd's to the best directory) — ported to a **native [zshrs](https://github.com/MenkeTechnologies/zshrs) plugin**. A faithful reimplementation in Rust: the `~/.fasd` datafile format, the add/aging rule, the regex + fuzzy matching, the frecency scoring, and the command options are all reproduced.

Where [zsh-z](https://github.com/MenkeTechnologies/zshrs-zsh-z) tracks only directories you `cd` into, fasd tracks **files too** — the two compose.

### [`zshrs`](https://github.com/MenkeTechnologies/zshrs) &middot; [`znative`](https://github.com/MenkeTechnologies/zshrs/blob/main/docs/ZNATIVE.md) &middot; [`upstream`](https://github.com/clvv/fasd)

---

## Table of Contents

- [\[0x00\] Overview](#0x00-overview)
- [\[0x01\] Install](#0x01-install)
- [\[0x02\] Usage](#0x02-usage)
- [\[0x03\] How it works](#0x03-how-it-works)
- [\[0xFF\] License](#0xff-license)

---

## [0x00] OVERVIEW

```text
f cfg           → best file matching "cfg"
d proj          → best directory matching "proj"
a part          → best file OR directory
j proj          → cd to the best directory matching "proj"
v main          → open the best file match in $EDITOR
s foo           → list scored matches
f -l cfg        → list matching paths (no scores)
fasd -A path…   → add paths        fasd -D path…  → forget paths
```

---

## [0x01] INSTALL

```sh
znative load MenkeTechnologies/zshrs-fasd
```

Put that one line in your `.zshrc`. [znative](https://github.com/MenkeTechnologies/zshrs/blob/main/docs/ZNATIVE.md), zshrs's package manager, installs the plugin on the first shell start — clones it, runs `cargo build --release`, and `zmodload -R`s the resulting `libfasd` — then loads it from the store, zero-network, on every start after. Tracking begins on the first command; `f`/`d`/`a`/`j`/`v` search once there is history.

### Manual build

```sh
cargo build --release
zmodload -R ./target/release/libfasd.dylib   # .so on Linux
d <partial-dir>
```

---

## [0x02] USAGE

| Command        | Matches            | Action                                   |
| -------------- | ------------------ | ---------------------------------------- |
| `a [q…]`       | files + dirs       | best match                               |
| `d [q…]`       | directories        | best match                               |
| `f [q…]`       | files              | best match                               |
| `s [q…]`       | files + dirs       | scored list                              |
| `sd` / `sf`    | dirs / files       | scored list                              |
| `j [q…]`       | directories        | `cd` to the best match                   |
| `v [q…]`       | files              | open best match in `$EDITOR` (vim)       |
| `fasd [q…]`    | files + dirs       | the underlying command                   |
| `fasd_cd [q…]` | directories        | `cd` to the match — what `j` calls       |

Flags mirror fasd: `-a`/`-d`/`-f` kind, `-s` scores, `-l` list, `-i` interactive select, `-r` rank / `-t` recency ordering, `-R` reverse, `-e <cmd>` execute on the result, `-N` select the Nth entry, `-A`/`-D` add/delete.

`fasd` also takes the sub-commands the shell hook and completion use directly:

| sub-command       | effect                                                     |
| ----------------- | ---------------------------------------------------------- |
| `--add PATH…`     | record paths (alias `-A`) — this is what the `preexec` hook calls |
| `--delete PATH…`  | drop paths from the datafile (alias `-D`)                  |
| `--proc CMD…`     | feed a command line through the blacklist/shift/ignore filters, then record its path arguments |
| `--complete Q…`   | emit completion matches for `Q`                            |
| `--version`       | prints `1.0.1`, the upstream fasd version this reimplements — not the plugin's own crate version |

### Environment

| variable            | default          | effect                                                    |
| ------------------- | ---------------- | --------------------------------------------------------- |
| `_FASD_DATA`        | `~/.fasd`        | datafile path                                             |
| `_FASD_MAX`         | `2000`           | Σrank ceiling; past it every rank ages `* 0.9`            |
| `_FASD_FUZZY`       | `2`              | chars the fuzzy pass may skip between consecutive query characters; `0` disables it |
| `_FASD_TRACK_PWD`   | `1`              | set to `0` to stop recording `$PWD` on every command       |
| `_FASD_BLACKLIST`   | `--help`         | drop the whole command line if any word matches           |
| `_FASD_SHIFT`       | `sudo busybox`   | leading words to skip before reading the real command     |
| `_FASD_IGNORE`      | `fasd ls echo`   | commands whose path arguments are never recorded          |

---

## [0x03] HOW IT WORKS

The datafile (`$_FASD_DATA`, default `~/.fasd`) holds `path|rank|time` rows. A `preexec` hook feeds each command's path arguments (plus `$PWD`) to `fasd --add`: an existing entry re-ranks `rank + 1/rank` and touches its time, new paths enter at rank 1, and when the total exceeds `$_FASD_MAX` (2000) every rank ages `*0.9`. Queries match terms in order with the last term landing in the basename — case-sensitive, then case-insensitive, then fuzzy (`$_FASD_FUZZY`, 2). Scoring is `Σrank * frecent(dx)` where `frecent` is 6 (`<1h`), 4 (`<1d`), 2 (`<1w`), else 1 — `-r` weights by rank alone, `-t` by recency alone. The `cd` performed by `j` is delegated to the shell so `$PWD` and hooks stay correct — identical to fasd.

---

## [0xFF] LICENSE

MIT. Ported from [clvv/fasd](https://github.com/clvv/fasd) (MIT/WTFPL). See [LICENSE](LICENSE).
