# SeisPilot

**Natural-language control for the Tornado 3D seismic visualization software.**

SeisPilot lets you drive seismic navigation in plain English. You type a command
like *"go to the gas area and make it brighter with rainbow colors"*, an LLM
translates it into precise navigation parameters (crossline / inline / depth,
gain, colormap, etc.), and those commands are executed inside Tornado.

```
"Go to the gas area"  →  crossline 25431, inline 7878, depth 2231
```

Instead of manually entering `X=159738, Y=75000, Z=1750`, adjusting gain, and
selecting a colormap by index, you describe the result you want.

---

## How it works

SeisPilot is split into two processes that communicate **only** through a shared
SQLite database (a constraint imposed by the Tornado runtime's network/security
sandbox):

```
┌─────────────────┐     ┌──────────────────┐     ┌─────────────────┐
│   NLP end       │     │   Shared utils   │     │  Tornado end    │
│  (Windows)      │     │                  │     │   (Linux)       │
├─────────────────┤     ├──────────────────┤     ├─────────────────┤
│ • Chat terminal │     │ • Config loader  │     │ • Bookmark eng. │
│ • LLM parser    │◄───►│ • Coord mapper   │◄───►│ • Seismic nav.  │
│ • Command queue │     │ • Context loader │     │ • State manager │
└─────────────────┘     └──────────────────┘     └─────────────────┘
         │                       │                        │
         └───────────────────────┼────────────────────────┘
                                 │
                       ┌──────────────────┐
                       │   SQLite DB      │
                       │ • Command queue  │
                       │ • State / status │
                       └──────────────────┘
```

1. **NLP end** (Windows) — interactive chat terminal. Sends your text to an LLM,
   which returns JSON-RPC commands. These are written to the SQLite command queue.
2. **Tornado end** (Linux, run inside Tornado) — polls the queue, generates/edits
   the active bookmark, and updates Tornado's view. Writes status back to the DB.
3. **Shared utils** — coordinate mapping, config/context loading, and the JSON-RPC
   protocol used in both directions.

![Architecture](Simplified%20Architecture.png)

---

## Key components

- **Coordinate mapping** — bidirectional linear transform between seismic
  coordinates (crossline/inline/depth) and Tornado's Cartesian space (X/Y/Z),
  configured with two reference points per axis in `config.json`.
- **Domain context** — geological shortcuts ("gas area", "fault zone",
  "reservoir", …) mapped to coordinates in `context.json` and injected into the
  LLM prompt.
- **Multi-provider LLM** — an HTTP endpoint (e.g. a self-hosted Llama model) as
  the primary provider, with the Gemini API as a fallback. See
  `src/shared/llm/llm_provider.py`.
- **Bookmark engine** — Tornado is driven by editing a single bookmark XML
  (`data/bookmarks/TEMP_BKM.html`) built from templates in `data/templates/`.
- **State management** — SQLite-backed command queue with status tracking and a
  20-level undo/redo history.

---

## Project structure

```
.
├── config.json              # Coordinate mappings & data paths
├── context.json             # Domain knowledge for the LLM
├── .env                     # LLM provider configuration (not committed)
├── INSTRUCTIONS.md          # Detailed setup notes
│
├── src/
│   ├── nlp_end/             # Natural-language end (Windows)
│   │   ├── nlp/             #   LLM command parser & validator
│   │   ├── terminal/        #   Interactive chat terminal
│   │   └── gui/             #   Optional Tkinter GUIs
│   │
│   ├── tornado_end/         # Tornado end (Linux)
│   │   ├── core/            #   Bookmark engine, seismic navigation
│   │   └── tornado_listener.py
│   │
│   └── shared/              # Shared between both ends
│       ├── database/        #   SQLite management
│       ├── llm/             #   LLM providers (HTTP + Gemini)
│       ├── protocols/       #   JSON-RPC 2.0 protocol
│       └── utils/           #   Config, context, coordinate mapper
│
├── data/
│   ├── bookmarks/           # Active bookmark loaded into Tornado
│   ├── templates/           # Bookmark view templates
│   └── captures/            # Screenshots produced during view changes
│
└── database/                # SQLite database files
```

---

## Setup

> Full notes are in [INSTRUCTIONS.md](INSTRUCTIONS.md).

The two ends run on different machines and Python versions, so dependencies are
installed into separate virtual environments that are **imported from directly,
not activated** (a workaround for the Tornado sandbox):

- `.win-venv` — Windows, Python 3.13, packages from `win-requirements.txt`
- `.linux-venv` — Linux, Python 3.6.8 (to match Tornado), packages from
  `linux-requirements.txt`

Both ends must point at the **same** directory (e.g. a network share), since they
communicate only through the shared SQLite database.

### 1. Configure the LLM

```env
# .env  (see ".env(example only)")
DEFAULT_LLM_PROVIDER=http_llm
HTTP_LLM_SERVER_URL=http://your-llm-endpoint/v1/chat/completions
FALLBACK_LLM_PROVIDERS=gemini
GEMINI_API_KEY=your_gemini_api_key
```

### 2. Configure coordinate mappings

Edit `config.json` with two reference points per axis from your survey:

```json
"crossline_to_x": {
  "point1": { "crossline": 25519, "x": 159488 },
  "point2": { "crossline": 25599, "x": 159988 }
}
```

### 3. Add domain context

Edit `context.json` to map natural-language locations to coordinates:

```json
{ "domain_context": "When the user asks to go to the gas area, move to crossline 25431, inline 7878, depth 2231. ..." }
```

### 4. Add templates

Place bookmark templates in `data/templates/` (at minimum `default_bookmark.html`).
Each saved XML must contain exactly **one** bookmark.

---

## Usage

Start the Tornado listener from the project root inside an XTerm (Linux):

```bash
tornadoi -script src/tornado_end/tornado_listener.py
```

Start the chat terminal on the Windows machine:

```bash
python src/nlp_end/main.py
```

Then talk to it:

```
seismic> go to the gas area
✅ Moving to crossline 25431, inline 7878, depth 2231

seismic> make it brighter and use rainbow colors
✅ Increasing gain and changing to rainbow colormap

seismic> undo that
✅ Reverted to previous state
```

### Coordinate mapper (standalone)

```python
from shared.utils.coordinate_mapper import get_coordinate_mapper

mapper = get_coordinate_mapper()

x, y, z = mapper.seismic_to_cartesian(crossline=25559, inline=5000, depth=1750)
crossline, inline, depth = mapper.cartesian_to_seismic(x=x, y=y, z=z)
```

---

## Coordinate transform

Each axis uses a two-point linear mapping `cartesian = slope · seismic + intercept`:

```
slope     = (x2 - x1) / (crossline2 - crossline1)
intercept = x1 - slope · crossline1

# e.g. crossline 25519 → X 159488, crossline 25599 → X 159988
# slope = (159988 - 159488) / (25599 - 25519) = 6.25
```

Results are rounded to integers to match Tornado's coordinate space.

---

## Prerequisites

- Python 3.6+ on the Tornado/Linux end, Python 3.8+ on the Windows/NLP end
- SQLite3
- An HTTP LLM endpoint and/or a Gemini API key
- Tornado seismic visualization software (for the Tornado end)

---

## Notes & limitations

- The two ends share a directory rather than copies; both read/write the same
  SQLite database.
- View changes currently require capturing an image (saved to `data/captures/`),
  a side effect of how the Tornado API is driven.
- `.env`, the virtual environments, and the SQLite DB are git-ignored.
