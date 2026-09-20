# LLMBench

A web app for benchmarking local language models across inference engines.
It measures how fast your models and their parameter configurations actually
generate, compares them on an identical prompt set, judges whether the
differences survive the noise, and keeps a history of every run — with the
profile that produced it.

Currently two engines are supported. Choosing one is the app-wide decision the
whole page reads, and it lives in its own **Engine station** — a bold chooser
with a card per engine — not a buried dropdown:

- **Ollama** — models are pulled by tag and served by one long-running server.
- **llama.cpp** — models are `.gguf` files served by a per-model
  `llama-server` process the app spawns and stops for you.

The Engine station also holds everything about the chosen engine: where it is
configured, its server, and the list of models it can run. For llama.cpp that
means pointing it at its installation folder and the folders to scan for
`.gguf` files, and — because the scan is recursive — a model may live in a
subfolder of one of those and it still shows up.

## What it does

- **Four run shapes from one form.** What kind of test a setup is follows from
  the inputs rather than a mode you pick first:
  - one model, no configuration — a plain speed test;
  - one model, several configurations — compare configurations;
  - several models, no (or one) configuration — compare models;
  - several models, several configurations — a tournament, where you pair each
    model with the configuration it races under.
- **Engine-aware configurations.** Each engine publishes its own option
  table, and the configuration editor is built from it. For llama.cpp that
  includes the GPU layer count (`n_gpu_layers`), context size, batch sizes,
  thread counts, flash attention and memory-mapping — the session-scoped
  knobs that decide what fits in VRAM and how fast it runs. Comparing two GPU
  layer counts on one model is just a two-configuration run.
- **Honest numbers.** Every prompt runs a configurable number of times;
  results average the runs and report their spread (standard deviation,
  minimum, maximum). A significance verdict then states whether the gap
  between the top two entries is larger than the measured noise — or says so
  plainly when the run was measured too few times to judge.
- **Machine readings.** When an NVIDIA GPU with `nvidia-smi` is present, each
  generation also records VRAM usage and the hottest GPU's temperature and
  clock, taken right after the generation finished.
- **Shared prompts by construction.** Every configuration in a run answers
  the same prompts, so a difference between two entries describes the
  entries, never the questions.
- **Configurations you can build or import.** A schema-driven editor for the
  engine's options, plus upload/paste of a JSON configuration file (with a
  preview before it is applied), duplication, and exporting the current
  setup back out as a file.
- **Run control.** A background worker with a live progress bar, mid-run
  cancellation that stops generation in flight and releases what the run
  loaded, and a single active run at a time so benchmarks never share the GPU.
- **History with profiles.** Every finished run is saved automatically
  (newest first, capped at 100) together with its profile: which engine, at
  which version, on which machine — so a number can always be traced back to
  the setup that produced it.

## Requirements

- Python 3.10+
- One or both engines:
  - **Ollama** on `http://127.0.0.1:11434` (or elsewhere — see below)
  - **llama.cpp** — a build containing `llama-server`, plus a directory of
    `.gguf` models
- NVIDIA driver with `nvidia-smi` on the PATH — optional; the GPU columns
  simply stay absent without it

## Configure the engines

There are two ways to point the app at its engines, and they combine:

**In the interface (recommended for llama.cpp).** Open the **Engine station**
and pick llama.cpp. Its configuration block asks for three things:

- the **installation folder** — the directory that holds `llama-server.exe`;
  the app runs the server from there and reads the build version from the
  executable;
- the **model folders** — one per line; each is scanned, including its
  subfolders, for `.gguf` files, and every file found becomes a selectable
  model;
- the **server address** — where the spawned server listens (the default is
  fine).

Save with **Apply**: the app validates that the folders exist and that the
installation folder really holds the executable, then persists the values to
`settings.json`. A value saved here wins over the environment, so it survives
a restart and a re-scan picks up whatever the folders now hold.

**Through the environment.** Every value can also come from an environment
variable, which the interface settings fall back to when nothing is saved —
useful for a fresh checkout or a service that is configured out of band:

| Variable | Engine | Meaning | Default |
| --- | --- | --- | --- |
| `OLLAMA_HOST` | Ollama | Root of the Ollama HTTP API | `http://127.0.0.1:11434` |
| `LLAMA_CPP_HOME` | llama.cpp | Directory holding `llama-server.exe` | — |
| `LLAMA_CPP_MODELS` | llama.cpp | Folders scanned (recursively) for `.gguf` files; several may be given, separated by `;` (Windows) or `:` | — |
| `LLAMA_CPP_HOST` | llama.cpp | Address the spawned server listens on | `http://127.0.0.1:8082` |

For example, on Windows:

```bash
set LLAMA_CPP_HOME=D:\AI-Model\llama-cpp\llama-b11056-bin-win-cuda-13.4-x64
set LLAMA_CPP_MODELS=D:\AI-Model\MODELS
python app.py
```

An engine that is not installed stays listed in the interface but simply
reports itself as not found; the other engine keeps working.

## Install and run

```bash
pip install -r requirements.txt
python app.py
```

Then open <http://127.0.0.1:5000>.

## Data and logs

Everything the app persists lives under one per-user directory:

```
%LOCALAPPDATA%\LLMBench          (Linux/macOS: ~/.llmbench)
├── settings.json                engine settings saved in the interface
├── benchmarks/
│   ├── index.json               one small record per saved run
│   └── <id>.json                the full result of one run
└── logs/
    └── llmbench.log             the execution log
```

Each saved run's `<id>.json` carries its **profile** — the engine id, label
and version, the server host, the machine (GPU name, driver, memory) and a
capture timestamp — so a number in the history can be traced back to exactly
what produced it.

## API overview

Every endpoint answers with the same envelope:
`{"ok": bool, "error": str | null, "data": ...}`.

| Endpoint | Method | Purpose |
| --- | --- | --- |
| `/api/engines` | GET | The registered engines, in display order |
| `/api/engines/<id>/status` | GET | Is this engine installed, running, at which version |
| `/api/engines/<id>/start` | POST | Start the engine's server, wait for it to answer |
| `/api/engines/<id>/stop` | POST | Stop the engine's server, wait for it to go quiet |
| `/api/engines/<id>/models` | GET | The installed models with size and details |
| `/api/engines/<id>/models/running` | GET | The models resident in memory, with VRAM |
| `/api/engines/<id>/models/unload` | POST | Unload one resident model |
| `/api/engines/<id>/models/remove` | POST | Delete one installed model (Ollama only) |
| `/api/settings` | GET | Every configurable engine's current values (effective + environment) |
| `/api/settings/<id>` | POST | Validate and save one engine's configuration |
| `/api/benchmark/schema` | GET | The option table of one engine (`?engine=`) |
| `/api/benchmark/parse` | POST | Validate a pasted or uploaded configuration file |
| `/api/benchmark/run` | POST | Start a benchmark in the background (`engine` in the body) |
| `/api/benchmark/status` | GET | Poll the running or last finished run |
| `/api/benchmark/cancel` | POST | Ask the running comparison to stop |
| `/api/benchmark/clear` | POST | Discard a finished run |
| `/api/benchmark/export` | POST | Download the current setup as a configuration file |
| `/api/benchmark/results-json` | POST | Download a finished run as one JSON document |
| `/api/benchmark/results-csv` | POST | Download a finished run's summary table as CSV |
| `/api/history` | GET | The saved runs, newest first |
| `/api/history/<id>` | GET / DELETE | One saved run in full / remove it |
| `/api/logs` | GET | The execution log, filterable by level, component, action |

## Project layout

```
app.py                  entrypoint: build the app and start it
llmbench/
├── __init__.py         app factory: wiring, blueprint registration
├── core/               the domain — no Flask, no HTTP server of its own
│   ├── engines/        the pluggable inference backends
│   │   ├── base.py     the BenchEngine interface every backend implements
│   │   ├── registry.py the named collection of configured engines
│   │   ├── ollama.py   the Ollama backend
│   │   └── llama_cpp.py the llama.cpp backend (llama-server lifecycle)
│   ├── limits.py       the hard caps a run request must respect
│   ├── options.py      input normalisation against an engine's option table
│   ├── plan.py         run planning: a request becomes the matrix to execute
│   ├── interrupt.py    cooperative stop signals for long-running operations
│   ├── telemetry.py    the nvidia-smi samples (identity, VRAM, temperature)
│   ├── suite.py        one benchmark case: a model under one configuration
│   ├── matrix.py       the matrix executor over a whole run plan
│   ├── stats.py        folding, averaging and spread of the measurements
│   ├── verdict.py      the significance judgement on a finished matrix
│   ├── daemon_client.py the Ollama HTTP client (status, models, generate)
│   ├── server_control.py starting and stopping the local Ollama process
│   ├── eventlog.py     the execution event log
│   ├── settings.py     the persisted settings store (one section per engine)
│   └── storage.py      every filesystem location the app uses
├── services/           the stateful middle between core and HTTP
│   ├── run_manager.py  the single background benchmark worker
│   └── archive.py      the persisted run store
└── api/                the HTTP surface
    ├── envelope.py     the JSON envelope and error translation
    ├── engines.py      the per-engine server and model endpoints
    ├── settings.py     the persisted-engine-configuration endpoints
    ├── runs.py         the benchmark workbench endpoints
    ├── history.py      the saved-run endpoints
    ├── events.py       the execution-log endpoints
    └── pages.py        the rendered dashboard
templates/              the single-page dashboard
static/                 the dashboard's CSS and JavaScript
_dev/                   development helpers: e2e suite and mock servers
```

## Adding an engine

An engine is one module in `llmbench/core/engines/` implementing the
`BenchEngine` interface: probe the installation, own the server lifecycle if
there is one, list models, prepare one model under one configuration,
stream generations, and declare its option catalog. Registering it in the
app factory is the only other change — the executor, the worker, the API
and the dashboard pick it up from there.

## Testing

`_dev/` holds an end-to-end suite and mock servers for both engines, so the
full pipeline — status, a run of every shape, progress accounting, the
significance verdicts, history, exports, the log and cancellation — can be
exercised without a real model:

```bash
python _dev/e2e_test.py
```

The suite starts its own mock Ollama and llama-server on throwaway ports and
a throwaway data directory; it never touches a real server or saved history.
