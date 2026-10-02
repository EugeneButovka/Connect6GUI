# ConnectMore

ConnectMore is a GUI for Connect6 game engines, written in Python 3.
Engines speaking the command protocol below are supported (default: `TIA.Connect6` in `engines/main`).

Runs on Linux, macOS and Windows.

About the game: [Connect6]. Original default engine: [Cloudict].

---

## Contents

- [Setup & Run](#setup--run)
- [Engine Selection](#engine-selection)
- [Engine Protocol](#engine-protocol)
- [Tournaments](#tournaments)
- [Change Notes / Known Issues](#change-notes--known-issues)
- [Project Structure](#project-structure)
- [License](#license)

---

## Setup & Run

The project is managed with [uv](https://docs.astral.sh/uv/) and requires Python 3.12+ with Tcl/Tk.

```sh
# one time: install uv
curl -LsSf https://astral.sh/uv/install.sh | sh

# create .venv and install the uv-managed Python 3.12
uv sync

# run the GUI
uv run python ConnectMore.py
```

> **Why a uv-managed Python?** It bundles a working Tcl/Tk. Homebrew Pythons may lack the `_tkinter` module, and we pin 3.12 because its Tk build is verified with this app.

After replacing the engine binary, no extra steps are needed — the GUI spawns it as a subprocess at runtime.

## Engine Selection

The default engine is picked per platform by `defaultEnginePath()` in `tournament.py`:

| OS | Candidates (first existing wins) |
| --- | --- |
| Windows | `engines/main.exe`, `engines/cloudict.exe` |
| macOS / Linux | `engines/main`, `engines/cloudict.app`, `engines/cloudict.linux` |

> Windows: build the TIA engine on Windows (`uv run pyinstaller main.spec` in
> the engine project) and copy `dist/main.exe` to `engines/main.exe`.
> Engine binaries are platform-specific — a macOS/Linux build cannot be
> executed on Windows (WinError 193).

To change it in the source, edit the candidate list in `defaultEnginePath()`:

```python
if os.name == 'nt':
    candidates = ['main.exe', 'cloudict.exe']
else:
    candidates = ['main', 'cloudict.app', 'cloudict.linux']
```

Alternatives at runtime:

| Method | How |
| --- | --- |
| Per-player override | `Load Black Engine` / `Load White Engine` buttons |
| Per-tournament | Tournament file: one engine path per line |

## Engine Protocol

The GUI runs the engine as a subprocess: commands go to the engine's **stdin**, replies come back on **stdout**. The board is 19×19 with coordinates `A`–`S`.

| Command | Meaning |
| --- | --- |
| `name` | Print the name of the game engine |
| `new xxx` | Start a new game and set the engine to White |
| `black XXXX` | Place the black stone(s) on position XXXX |
| `white XXXX` | Place the white stone(s) on position XXXX |
| `next` | Engine searches and replies with its move |
| `depth d` | Set the alpha-beta search depth (AI Level: Low=2, Medium=3, High=4) |
| `vcf` / `unvcf` | Enable / disable VCF search |

The engine answers with lines such as `name <name>` and `move XXXX`; all other output (stats, help text) is ignored by the GUI.

> **Performance note:** TIA.Connect6's search has no pruning and evaluates the
> whole board at every node, so runtime grows ~40x per depth level. Measured
> per move: depth 2 ≈ 2–3s, depth 3 ≈ 4–5s, depth 4 ≈ 100s+ (GUI timeout is
> 30s). That's why Low/Medium map to 2/3.

## Tournaments

- `Load Tournament` — select a file with one engine path per line; each engine is test-spawned on load.
- `Start Games` — round-robin: every engine plays every other engine (both colors).
- `Save results` — writes a report: classification (2 pts win / 1 pt draw), Buchholz tie-breaker, cross table, per-game moves and times.

A working sample is provided in [`tournaments/sample.txt`](tournaments/sample.txt):

```text
engines/main
engines/main
```

Paths are resolved from the launch directory, so run the GUI from the project root
(e.g. `uv run python ConnectMore.py`). Duplicate paths are fine — they make an
engine play itself. Point the lines at different engine executables to compare
them head-to-head. Your own tournament files live in `tournaments/` (gitignored;
only `sample.txt` is tracked).

## Change Notes / Known Issues

- **uv migration** — the project uses `pyproject.toml` / `.python-version` / `uv.lock`; run via `uv run python ConnectMore.py`.
- **Tk 9 thread safety** — Tk 9.0 (bundled with the uv Python) aborts the process when Tk is touched from a non-main thread. All UI work from the engine search thread is marshalled to the main thread (`runInMainThread` / `processUiQueue` in `ConnectMore.py`). **Any new Tk call added to the search thread must go through `runInMainThread()`.**
- **Restart race** — restarting a game mid-play used to freeze the new game (a stale exception from the old game declared a win). A game generation counter (`gameGen`) now ignores stale exceptions.
- **Engine process cleanup** — engines are spawned in their own process group and killed as a whole tree on release (`killpg` / `taskkill`). Previously `terminate()` only killed the PyInstaller bootloader and orphaned the real engine child process.
- **Engine EOF crash** — fixed in the engine source (`Connect6Engine/game_engine.py`: `input()` now handles `EOFError` so the engine exits cleanly when the GUI closes its stdin).

## Project Structure

```
ConnectMore.py   # Tk GUI: board, buttons, search thread, UI queue
tournament.py    # players, games, round-robin scheduling, scoring/report
engine.py        # engine subprocess wrapper + move parsing (A–S protocol)
engines/         # engine executables (default: main)
imgs/            # board / stone / emote images
patterns.in      # legacy patterns file for Cloudict
pyproject.toml   # uv project config
.python-version  # pinned interpreter (3.12)
```

## License

Copyright (c) 2014, Liang Li \<ll@lianglee.org; liliang010@gmail.com\>.
All rights reserved. BSD-style — see [LICENSE.txt](LICENSE.txt).

![screenshot](http://i.imgur.com/OL2kxZf.png)

> Have fun! :-)

[Cloudict]: https://github.com/lang010/cloudict
[Connect6]: http://en.wikipedia.org/wiki/Connect6