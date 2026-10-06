> Sanitized mirror of Forgejo `serhii/dnd-agent2`. Source code is not published here.
>
> Commit texts: `commits/`. Need the code? Email: sergeyalpatov1@gmail.com
> Source: Forgejo `serhii/dnd-agent2` | Synced: 2026-10-06T04:24:08Z

---

# D&D AI Dungeon Master

An AI-powered Dungeon Master agent for playing Dungeons & Dragons via a CLI chat interface. The DM agent uses an LLM to narrate the game, manages character state, runs combat encounters, and persists sessions.

## Architecture

| Project | Purpose |
| --------- | --------- |
| **Dnd.Core** | LLM client, prompt building, context budget estimation, retry policies, agent streaming |
| **Dnd.DM** | Domain logic: character management, dice rolling, combat state, game state machine, DM agent tools |
| **Dnd.CLI** | Spectre.Console chat loop, command registry, session lifecycle, input/output rendering |
| **Dnd.Infrastructure** | Session persistence (save/load), session models |
| **Dnd.Tooling** | Build and analysis tooling |
| **Dnd.Core.Tests** | Unit tests for core LLM/prompting logic |
| **Dnd.DM.Tests** | Unit tests for character, dice, combat, and state machine logic |
| **Dnd.Infrastructure.Tests** | Unit tests for session persistence |

## Setup

```bash
# Build
dotnet build

# Run all tests
dotnet test

# Run the CLI
dotnet run --project Dnd.CLI
```

See [`Makefile`](Makefile) for common build targets.

## LLM profiles (local llama.cpp vs. cloud)

The DM talks to an OpenAI-compatible endpoint. The base `LLM` section in
[`Dnd.CLI/appsettings.json`](Dnd.CLI/appsettings.json) defaults to the local llama.cpp
server (Qwen3.6-27B). An optional **profile** overlays the keys it declares on top of that
base section, so you can switch backends without editing the file.

| Profile | Provider | Model | Endpoint |
| --------- | --------- | ------- | --------- |
| *(default)* | `llama.cpp` | `Qwen3.6-27B` | `http://100.110.77.11:8080/v1` |
| `opencode-go-deepseek` | `opencode-go` | `deepseek-v4.1-flash` | `https://opencode.ai/zen/go/v1` |

The cloud profile is handy for running tests and game sessions in parallel: it does not
compete for the local GPU.

### Selecting a profile

```bash
# 1) Command-line flag (highest precedence)
dotnet run --project Dnd.CLI -- --profile opencode-go-deepseek

# 2) Environment variable
export DND_LLM_PROFILE=opencode-go-deepseek
dotnet run --project Dnd.CLI

# 3) One-off value overrides on top of the profile
dotnet run --project Dnd.CLI -- --profile opencode-go-deepseek --set LLM:ThinkingLevel=low
```

Precedence (lowest → highest): `appsettings.json` base `LLM` section → selected profile →
environment variables (`LLM__ModelName=…`, `__` is the section separator) → `--set
LLM:Key=value` command-line overrides.

### Supplying the API key

The key **never** lives in git. `appsettings.json` keeps the `REPLACE_WITH_REAL_API_KEY`
placeholder; the real key is read from the environment:

```bash
export DND_LLM_API_KEY=sk-...          # preferred
# or, provider-specific fallback:
export OPENCODE_GO_API_KEY=sk-...
```

`DND_LLM_API_KEY` beats `OPENCODE_GO_API_KEY` and `LLM__ApiKey`. Selecting a cloud provider
without a usable key fails loudly at startup (before any HTTP request) instead of surfacing
as a 401 in the middle of a session. The local llama.cpp profile keeps its historical
`sk-dummy` stub.

opencode-go also requires two request headers, which the client sends for that profile only:
`x-opencode-session` (the stable game session id, used for routing and prompt caching) and
`User-Agent: dnd-agent2/1.0`. The local llama.cpp requests are left untouched.

### Cost

The `opencode-go-deepseek` profile bills per token: **$0.15 / 1M input tokens** and
**$0.6 / 1M output tokens** (DeepSeek V4.1 Flash). Reasoning tokens count as output.

### Quick start

```bash
export DND_LLM_API_KEY=sk-...
make run-cloud
```

`make run-cloud` runs the CLI against `opencode-go-deepseek` and refuses to start when no
key is set.

### Reasoning (thinking) display

The model's reasoning block is **hidden by default** so the narration is not buried under a
stream of English chain-of-thought. The model still reasons — hiding is display-only, and
the full text is always recorded in `reports/llm-turns`. Show it at startup with the
`--thinking` flag:

```bash
dotnet run --project Dnd.CLI -- --thinking
```

or flip it mid-session with `/thinking on|off` (the block is capped at 3000 displayed
characters per turn and runs of blank lines collapse to a single separator, so a
paragraph-gap-happy model cannot flood the screen).

## Features

- **Character Import** — JSON-based character creation with validation (name, race, class, level, ability scores, HP, AC, speed)
- **Spell Slot Calculation** — Full caster, half caster, warlock pact magic, and non-caster support
- **Dice Rolling** — `NdM` notation with advantage/disadvantage, critical hit/failure detection for d20
- **Combat State** — Initiative ordering, turn management, damage/healing, conditions, auto-skip dead combatants
- **Game State Machine** — Exploration → Combat → Rest → Exploration transitions with XP tracking
- **Session Persistence** — Save/load game state with full character data and chat history

## Development Guidelines

See [`AGENTS.md`](AGENTS.md) for code rules (C#/.NET conventions, DI, SOLID, interfaces, project structure).
See [`CONTEXT.md`](CONTEXT.md) for domain documentation and architecture decisions.

## Bundled SRD data

`Dnd.Infrastructure/data/dnd5e_srd.json` is a bundled dataset of D&D 5e rules content
(spells, equipment, magic items, conditions, monsters, class/racial features) built by
`dotnet run --project Dnd.Tooling -- build-srd` from two upstream sources (provenance,
including the exact commit SHAs, lives in the dataset's `_meta`):

- **5e-bits/5e-database** (MIT + OGL) — spells, equipment, magic items, conditions, monsters.
- **foundryvtt/dnd5e** (MIT + CC-BY-4.0) — class features and racial traits (2024 SRD).

Both upstreams are OGL/CC-licensed SRD content; this project adds no copyrighted
material of its own. Rebuild the dataset with `--no-fvtt` to skip the FoundryVTT
features, or `--status` to print the current counts and provenance.
