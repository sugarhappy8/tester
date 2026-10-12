# Hearth: design spec

| | |
|---|---|
| **Status** | Draft, waiting for owner review |
| **Date** | 2026-10-12 |
| **Owner** | sugarhappy8 |
| **Repo** | `sugarhappy8/tester`, branch `hearth-v1` |
| **Working name** | Hearth (easy to change; must not use "Claude" in its branding) |

## 1. Summary

Hearth is a self-hosted web app. It turns any local LLM served over an OpenAI-compatible API (Ollama, LM Studio, llama.cpp, vLLM and similar) into an agent like Claude Code or ChatGPT that runs on the user's own PC. It can:

- take in large amounts of files and archives, and search them with citations
- give each project its own folder, where the agent writes, runs and tests code
- search the web, fetch pages, and drive a headless browser
- create projects from the UI and connect to GitHub to clone, branch, push and open PRs
- use **Agent Skills** (the `SKILL.md` format). It ships its own skills, bundles the open-licensed skills we are allowed to redistribute, and imports the user's own Claude skills in place.
- hand self-contained jobs to **helper agents**, either one at a time or several in parallel, depending on what the user's hardware can handle

Hearth's main contribution is a **reliability layer for local models**. It probes each model, picks a tool-calling mode, recovers and repairs malformed tool calls, sets the context size correctly, and compacts the context. This is what lets a mid-size local model finish real multi-step tasks.

**Honest ceiling.** Hearth gives a local model the same kinds of tools Claude has, but the quality of the results is still limited by the model. The target is a 16–24 GB GPU running a 20–35B model, such as `qwen3.6:27b`, `gpt-oss:20b`, `devstral-small-2:24b` or `glm-4.7-flash`. Hearth still works on smaller models and gets better with larger ones.

## 2. Decisions taken during brainstorming

| # | Decision | Choice |
|---|---|---|
| D1 | Audience | **Public on GitHub.** Anyone can install it and use their own local LLM. |
| D2 | Model tier to tune for first | **Mid: 16–24 GB VRAM**, about 20–35B models. Small models degrade gracefully; large models are not held back. |
| D3 | Code execution default | **Tiered.** Commands run on the host after approval by default. An optional Docker or Podman container per project, with one toggle. |
| D4 | v1 acceptance scenarios | All four: (1) coding on an uploaded project, (2) Q&A over a large document dump, (3) web research, (4) producing files through skills |
| D5 | Approach | **Build our own:** Python FastAPI backend plus a prebuilt React/TypeScript UI. We do not fork OpenHands or write an Open WebUI plugin. |
| D6 | Extra requirements | Hearth-native skills for each feature; GitHub connection in the UI; create project folders from the UI; optional helper agents that scale to the hardware |
| D7 | Out of v1 | MCP connectors, image generation, lightweight OS sandboxes (`srt`/bwrap), multi-user or LAN access, voice |

## 3. Success criteria

v1 is **done** when every item below is true:

1. **Install.** A one-line installer on Windows 10/11, macOS 13+ and Linux (x64/arm64) produces a working `hearth` command. The command opens the UI in the browser. The user does not need Python, Node or Docker installed beforehand. Git is installed by the installer if it is missing.
2. **Connect.** Hearth detects and connects to Ollama, LM Studio, llama.cpp and vLLM, and also accepts any OpenAI-compatible URL.
   - The probe classifies the model's tool-calling mode and context size.
   - It shows a fit card.
   - On Ollama, Hearth fixes the 4K-context trap automatically.
   - On LM Studio, the context fix takes one click.
3. **Acceptance scenarios.** All four scenarios (§20.4) pass at least 2 of 3 runs. The runs use a reference mid-tier model (`qwen3.6:27b` on Ollama at 32K context, or the best 24 GB model the probe recommends at release time).
4. **CI.** Unit and end-to-end tests (using the mock model server) pass on GitHub Actions for Windows, macOS and Linux.
5. **Licences.** Every bundled third-party skill ships with its licence and the attribution it requires. No proprietary-licensed skill is bundled, copied or loaded.
6. **Safety.** No file tool can read or write outside the project. Outward-facing actions (push, PR, repo creation) and actions on the danger list always need approval. Model-run commands never see the LLM API key or the GitHub token.

## 4. Architecture

### 4.1 Process model

```
 Browser (React SPA) ──HTTP + WebSocket──▶  hearth (one Python process, 127.0.0.1:7432)
                                            │
                    ┌───────────────────────┼─────────────────────────────────────────┐
                    │ api/   REST + WS, auth token, Origin/Host checks                 │
                    │ agent/ loop · tool-call parsing/repair · context · helpers       │
                    │ llm/   backend adapters · probe · hardware/capacity              │
                    │ tools/ registry + tool implementations                           │
                    │ sandbox/ host runner · container runner · processes · approvals  │
                    │ workspace/ projects · ingestion · conversion · index · snapshots │
                    │ web/   search · fetch · browser (Playwright, own thread)         │
                    │ skills/ discovery · licence guard · adapter · selection          │
                    │ integrations/github                                              │
                    │ store/ SQLite (app DB) + per-project index DB                    │
                    └──────────────────────────────────────────────────────────────────┘
                         │ HTTP                 │ subprocess / docker exec      │ HTTPS
                  Local LLM server        project folder / container      web, GitHub
```

- **One process.** It is bound to `127.0.0.1` on port 7432 by default. If that port is busy, Hearth takes the next free port.
- **The agent keeps running if the browser closes.** Agent turns run as server-side tasks. If the browser disconnects, the turn carries on. When the UI reconnects, it replays the events it missed using sequence numbers.
- **Blocking work stays off the event loop.** Subprocesses, Playwright and ingestion run in worker threads or dedicated threads. This avoids the uvicorn limitation on Windows, where asyncio cannot start subprocesses with a selector event loop.

### 4.2 Tech stack

Versions are pinned at implementation time.

**Backend:** Python 3.12, installed through `uv` as a standalone build.

| Area | Libraries |
|---|---|
| Web server and models | FastAPI, uvicorn, pydantic v2, httpx (async) |
| Storage | `sqlite3` (stdlib, with FTS5), `sqlite-vec` |
| Tool-call repair | `json-repair`, `jsonschema`, `rapidfuzz` |
| Approvals | `bashlex` |
| Processes | `psutil`; `pywin32` on Windows; `ptyprocess` on POSIX, `pywinpty` on Windows |
| Containers | `docker` SDK, which also works with Podman |
| Secrets and paths | `keyring`, `platformdirs` |
| Files | `watchfiles`, `pathspec` |
| Web | `trafilatura`, `markdownify`, `ddgs`, `playwright` (the browser binary is downloaded on first use) |
| Documents | `pypdfium2`, `python-docx`, `openpyxl`, `python-pptx` |
| Skills | `pyyaml` |

**Frontend:** React 19, Vite, TypeScript, Tailwind CSS 4, Radix UI primitives, lucide-react, Zustand, CodeMirror 6, `@xterm/xterm` 6, react-markdown with remark-gfm and a code highlighter, and `diff` (jsdiff).

The UI is built in CI and shipped inside the Python wheel, so end users never need Node.

**External binaries:**
- `git` is required, because it is needed for snapshots and GitHub. The installer ensures it is present.
- `uv` is required. The installer installs it.
- `fnm` is downloaded on demand when a project needs Node.
- Chromium headless shell is downloaded on first browser use, after the user consents. On Windows, Hearth can use the installed Edge instead.
- `rg` is used if it is on PATH; otherwise Hearth falls back to Python search.

### 4.3 Data layout

```
<data dir>  (platformdirs: %LOCALAPPDATA%\Hearth · ~/Library/Application Support/Hearth · ~/.local/share/hearth)
  hearth.db                 app DB: connections, model profiles, projects, chats, events, rules, settings
  skills/                   user-created / installed skills
  cache/                    web cache, browser profile, fnm node versions, sandbox image metadata
  logs/                     server logs (rotated)

<projects root>  (default ~/HearthProjects, configurable; any existing folder can also be opened)
  <project>/                the user's files (cwd for every command)
    .hearth/
      index.db              FTS5 + trigram + sqlite-vec index, converted-text cache
      logs/<id>.log         full untruncated tool/command output (model can re-read)
      snapshots.git/        shadow git dir for per-turn undo (separate from any project .git)
      uploads/              staging for in-flight uploads
```

- **Keeping `.hearth/` out of version control.** If the project is a git repo, Hearth adds `.hearth/` to `.git/info/exclude`, so it never touches the user's `.gitignore`.
- **Secrets.** API keys and the GitHub token go in the OS keychain through `keyring`. If no keychain is available, they are stored in `<data dir>/secrets.json` with user-only permissions, and the UI shows a warning.

## 5. Model layer (`llm/`)

### 5.1 Connections

- **What a connection is.** A connection is a base URL, an optional API key, and a backend type, which is either auto-detected or set manually.
- **Presets.** Ollama (`http://127.0.0.1:11434`), LM Studio (`:1234`), llama.cpp (`:8080`), vLLM (`:8000`), and custom.
- **First-run scan.** Hearth probes these local ports and offers whatever it finds.
- **Model roles:**
  - **main:** required.
  - **helper:** optional. Used by helper agents. Falls back to main.
  - **embedding:** optional. Used by the index. If missing, search uses keyword search only.

### 5.2 Backend detection, in this order

1. `GET /api/version` responds → **Ollama**.
2. `GET /props` responds with `default_generation_settings` → **llama.cpp**.
3. `GET /api/v1/models` responds in LM Studio's shape → **LM Studio**.
4. `GET /v1/models` responds:
   - a `max_model_len` field → **vLLM**
   - anything else → **generic OpenAI-compatible**

### 5.3 Adapters

All adapters share one interface:

```
chat(messages, tools|None, *, stream=True, response_format|None, options) -> AsyncIterator[Delta]
  Delta = text | thinking | tool_call(index, id?, name?, args_fragment|args_obj) | usage | finish(reason)
embed(texts) -> list[vector]
list_models() / model_info(model) -> ModelInfo
```

| Backend | How Hearth talks to it |
|---|---|
| **Ollama** | Native `/api/chat` with `options.num_ctx` set explicitly, `truncate:false`, `shift:false` and `keep_alive:"30m"`. Thinking uses `think` true/false/level and comes back in `message.thinking`. Tool `arguments` arrive as an object. Tool results are sent back as `{role:"tool", tool_name, content}`. |
| **llama.cpp / vLLM / LM Studio / generic** | `/v1/chat/completions`. Streamed `delta.tool_calls` fragments are accumulated per `index` and parsed only at `finish_reason`. `reasoning_content` is captured. `parallel_tool_calls:true` is sent only when llama.cpp reports `chat_template_caps.supports_parallel_tool_calls`. |

Common to every adapter:
- **Errors are normalised:**
  - unreachable server → `ModelUnavailable`
  - HTTP 400 "does not support tools" → `ToolsUnsupported`
  - `exceed_context_size_error`, or `finish_reason:"length"` with empty content → `ContextOverflow`
- **Images** are sent as base64 data, because Ollama does not accept image URLs.

### 5.4 Probe and model profile

The probe runs when a model is connected and can be re-run on demand. It has a budget of about 30 seconds and builds a **ModelProfile** containing:

- **`ctx_loaded`.** The context size actually in effect.
  - Ollama: `/api/ps` `context_length`, after a warm-up request with the chosen `num_ctx`.
  - LM Studio: `loaded_instances[].config.context_length`.
  - llama.cpp: `/props` `n_ctx`.
  - vLLM: `max_model_len`.
- **`ctx_max`.** The trained maximum.
  - Ollama: `/api/show` `model_info.<arch>.context_length`.
  - LM Studio: `max_context_length`.
  - llama.cpp: `/v1/models` `meta.n_ctx_train`.
- **`caps`.** `tools`, `vision`, `thinking`, `embedding`.
  - Ollama: `/api/show` `capabilities`.
  - LM Studio: `capabilities`.
  - llama.cpp: `chat_template_caps`.
  - Anything else: by test.
- **`tool_mode`.** One of A, B, C or D (§6.2), found with a live test. Hearth offers a single `get_time` tool and a prompt that clearly needs it, then classifies the result:
  - a correct native call → **A**
  - the call appears in content or reasoning text → **B**
  - HTTP 400, or no call → test the text protocol (**C**), and fall back to **D** if that fails too
- **`speed`.** Prompt tokens per second and generation tokens per second, from a 64-token generation.
- **`gpu_fit`.**
  - Ollama: from `/api/ps`, `size_vram == size` means the model is fully on the GPU, otherwise it has spilled to the CPU.
  - Other backends: unknown.
- **`family_preset`.** Sampling defaults chosen by model-name match from `llm/model_presets.json`. For example, Qwen thinking mode uses temp 0.6, top_p 0.95, top_k 20. The user can override these.
- **`tool_profile`.** `small`, `medium` or `full` (§6.6), derived from `ctx_loaded` and `tool_mode`.

**Context target.**
- **Default.** 32K, unless that exceeds `ctx_max`. If GPU info shows the larger setting would spill to the CPU, Hearth suggests 16K.
- **Choices.** The user can pick 16K, 32K or 64K, or a custom value.
- **How the setting is applied:**
  - **Ollama:** `num_ctx` is sent on every request, so the fix is automatic.
  - **LM Studio:** "Fix" calls `POST /api/v1/models/load {model, context_length}`.
  - **llama.cpp and vLLM:** the context is fixed when the server starts, so the UI shows the exact flag to use (`-c 32768` / `--max-model-len 32768`).

**Fit card.** The probe ends with a verdict:

| Label | Condition |
|---|---|
| Good for agent work | tool_mode A or B, and context ≥ 32K |
| Usable with supervision | tool_mode A/B with 16–32K, or mode C |
| Chat + document Q&A only | mode D, or context < 16K |

Each label comes with concrete fixes, such as "Raise context to 32K" or "This model has no tool support; try `qwen3.6:27b`".

### 5.5 Hardware detection and capacity test (for helper agents)

- **Local hardware.** Detected only when the endpoint host is loopback or a local interface:
  - NVIDIA: `nvidia-smi --query-gpu=name,memory.total,memory.used --format=csv,noheader`
  - AMD on Linux: `rocm-smi --showmeminfo vram --json`
  - Apple: `sysctl hw.memsize` and `machdep.cpu.brand_string`. About 75% of unified memory is treated as usable VRAM.
  - RAM: `psutil`
  - If nothing is detected, the GPU is reported as "unknown".
- **Server slots.** llama.cpp reports `/props total_slots`. vLLM batching is assumed to be at least 4. Ollama does not expose `OLLAMA_NUM_PARALLEL`, so Hearth relies on the benchmark.
- **Capacity benchmark.** Run from Settings → Helpers, or offered when helpers are first enabled.
  1. Warm up the model.
  2. Run 1, then 2, then 4 (then 8, only if 4 scaled) concurrent requests. Each generates 64 tokens from a 200-token prompt.
  3. For each level, measure aggregate generation tokens/second (`agg_tps`) and the per-request median.
  4. Stop at the first level where `agg_tps` improves by less than 30% over the previous level, or where per-request tokens/second drops below `max(8, 0.5 × single)`, or where any request errors.
  5. The **recommended N** is the last level that passed.
- **Memory caveat.** Each parallel slot needs its own KV cache. When GPU info is available, the UI estimates the extra memory and warns before N > 1 would push the model off the GPU.
- **Ollama with one slot.** The UI shows how to set `OLLAMA_NUM_PARALLEL` for the user's operating system.

## 6. Agent runtime (`agent/`)

### 6.1 The loop

```
turn(user_msg):
  snapshot project (shadow git)            # §9.6
  build prompt (system + project instructions + skills catalog + todo + history)  # §6.5
  loop step = 1..max_steps:
    ensure context budget (compact if needed)                                    # §6.4
    stream model response → emit text/thinking deltas to UI
    calls = extract_tool_calls(response, tool_mode)                              # §6.2–6.3
    if no calls:
        run final-answer guards (§6.7); if a guard fires → inject nudge, continue
        finish turn
    for call in calls (sequentially, in order; consecutive `task` calls may run concurrently §7):
        validate+repair → approval gate (§9.5) → execute → cap output → append result
        detect repetition (§6.7)
  if max_steps reached → pause; UI offers "Continue" (another max_steps) or "Stop"
```

- **Step cap.** `max_steps` defaults to 40 for the main agent and 20 for helpers. Users can change it.
- **Stop.** The Stop button cancels the model stream, kills any processes started by this turn, and marks the turn `stopped`.
- **Persistence.** Every step is saved to the `events` table as it happens (§16). If the server restarts mid-turn, the turn is marked `interrupted`, and the UI offers "Continue".

### 6.2 Tool-mode ladder

| Level | Mode | Mechanism |
|---|---|---|
| **A** | Native | `tools` are sent in the API request and `tool_calls` are read from the response |
| **B** | Native + recovery | Mode A, and when `tool_calls` is empty the recovery parser (§6.3) scans `content` and `reasoning_content` |
| **C** | Text protocol | Tools are described in the system prompt. The model writes `<tool name="read_file"><arg name="path">src/app.py</arg></tool>`, and several blocks per reply are allowed. Parsing is lenient. Once a tool tag starts, streamed text is no longer shown to the user. |
| **D** | Constrained single action | Each turn returns exactly one JSON object, either `{"thought":str,"tool":str,"args":{…}}` or `{"thought":str,"final":str}`. The format is enforced with Ollama `format` (a JSON schema), `response_format:{type:"json_schema"}`, or vLLM `tool_choice:"required"`. |

- **Retries and downgrades.** After two consecutive unrecoverable parse or validation failures, Hearth retries at a lower temperature (×0.5). If it fails again, Hearth moves down one level for the rest of the session. The new level is written to the model profile, which the user can reset.
- **Sequential calls.** Parallel tool calls from the model are always executed one after another, never concurrently, apart from helper tasks (§7.3).

### 6.3 Parsing and repair

**Recovery parser** (`agent/toolcall_recovery.py`)
- **Formats.** It recognises these formats, in this order:
  1. Hermes/Qwen2.5: `<tool_call>{json}</tool_call>`
  2. Qwen3-Coder/3.5/3.6: `<tool_call><function=NAME><parameter=K>v</parameter></function></tool_call>`
  3. GLM: `<tool_call>NAME<arg_key>k</arg_key><arg_value>v</arg_value></tool_call>`
  4. Mistral: `[TOOL_CALLS]name[ARGS]{…}` and `[TOOL_CALLS][{…}]`
  5. LM Studio: `[TOOL_REQUEST]{…}[END_TOOL_REQUEST]`
  6. Llama: `<|python_tag|>`, or a bare `{"name":…,"parameters"|"arguments":…}`
  7. Fenced ```` ```json ```` blocks with name and arguments keys
  8. Leaked Harmony text: `to=functions.NAME … <|message|>{…}`
  9. Hearth's own text-protocol `<tool …>` tags
- **Thinking blocks.** `<think>…</think>` blocks are stripped before parsing, including an unclosed `<think>` at the start.
- **Test corpus.** The parser has a corpus of real broken outputs under `tests/fixtures/toolcalls/`.

**Arguments**
1. Accept either an object or a string.
2. Run `json_repair`.
3. Coerce types to the schema: `"5"`→5, `"true"`→true, a stringified array → array, a single value where an array is expected → `[value]`.
4. Validate with `jsonschema`.

**Tool names**
- Matching is case-insensitive and ignores `_`/`-`. Common aliases are mapped, for example `bash`/`shell`/`execute` → `run` and `open_file` → `read_file`.
- If nothing matches, `rapidfuzz` is tried with a score of at least 85.
- If that also fails, the model gets a tool error that lists the valid names.

**Missing required arguments.** The tool returns an error naming the parameter and showing a one-line example of a correct call.

### 6.4 Context management

- **Budget.** `system + tool defs + history + reserve ≤ ctx_loaded`. The reserve is 4K, or 8K for thinking models.
- **Counting.** Hearth uses `prompt_eval_count` or `usage.prompt_tokens` from the last response. New text is estimated at 3.5 characters per token. On llama.cpp, `/tokenize` is used when the estimate is within 10% of the limit.
- **Compaction, cheapest first:**
  1. **On insert.** Tool output is capped to its head and tail (default 3K + 9K characters; small profile 1.5K + 4K). The full output goes to `.hearth/logs/<id>.log`, and the result says `…[N bytes omitted; full log: .hearth/logs/<id>.log — use read_file with offset]…`.
  2. **Masking.** Results from tool calls older than the last 6 steps are replaced with a one-line placeholder, for example `[read_file src/a.py → 212 lines, omitted]`.
  3. **Summarising.** At 75% of `ctx_loaded`, the same model summarises everything before the last 4 turns into structured progress notes: goal, decisions, files changed, open problems, and **exact error strings**. The notes replace those turns. The UI shows a "context compacted" divider, and the full transcript stays in the database.
- **Context overflow from the server.** Hearth compacts and retries once. If it overflows again, the error goes to the user with "Raise context / start new chat".
- **Thinking.** Reasoning is kept only inside the current tool-call turn. This is required by gpt-oss (Harmony), GLM-4.7 and Qwen3.6/3.8 with `preserve_thinking`. Reasoning from earlier turns is never re-sent.
- **UI meter.** The context meter shows usage and the compaction threshold.

### 6.5 Prompt assembly

The system prompt is built from fixed blocks, each with a token cap:

1. **Identity and rules.** Short and imperative, about 600 tokens, or 350 for the small profile. Covers: use tools rather than describing them, verify before claiming done, cite sources, stay in the project, ask when blocked.
2. **Environment.** OS; shell (bash, zsh or PowerShell); project path; execution tier; network on or off; today's date; available tools.
3. **Project instructions.** The first of `HEARTH.md`, `AGENTS.md` or `CLAUDE.md` in the project root, capped at 2K tokens.
4. **Skills catalogue.** The filtered list (§12.6): `name — description` lines.
5. **Todo list.** If one exists.
6. **Tool definitions.** Native schemas in modes A/B, or a text spec in modes C/D, for the active tool profile.

### 6.6 Tool profiles

| Profile | Used when | Tools exposed |
|---|---|---|
| `small` | ctx < 24K, or mode D | `read_file, write_file, edit_file, list_tree, search, run, web_search, web_fetch, load_skill, todo` |
| `medium` | 24K ≤ ctx < 48K | small + `glob, grep, start_process, process_output, stop_process, browser_*, present_file, ask_user, task, github_*` |
| `full` | ctx ≥ 48K | everything, plus `read_skill_file`, `list_processes`, `task_batch` |

- **Pinned groups.** Users can pin extra tool groups, which costs context.
- **Browser tools on demand.** In the small profile, browser tools are added only after the model calls `web_fetch` on a page whose extracted text is shorter than 500 characters, which is usually a JavaScript-heavy page.

### 6.7 Guards against faked work

These run deterministically, so no extra model call is needed:

- **Repetition.** The same tool with identical arguments three times in a row triggers the nudge: "You already ran this 3× with the same result; change approach or ask the user."
- **Described, not called.** In modes A/B/C, if the final text contains phrases like "I will now run…" or "Let me read…" with no tool call, Hearth sends one nudge: "If you intended to call a tool, call it now."
- **Unverified claim.** If the final answer claims tests pass, a build succeeds or something was "fixed", and the turn had no successful `run` or `process_output` after the last edit, Hearth sends one nudge: "Run the tests to verify before claiming success."
- **Uncited claim.** In document Q&A (any `search` used this turn), if the final answer has no citation, Hearth sends one nudge.
- **Nudge limit.** At most two nudges per turn, then the answer is accepted and flagged ⚠ in the UI.

## 7. Helper agents (`agent/helpers.py`)

### 7.1 Tools

- **`task`** `{description: str (≤60 chars), prompt: str, profile: "explore"|"edit"|"research"|"full", model: "main"|"helper"}`:
  1. Starts a fresh agent in the same project, with no chat history.
  2. Its system prompt says it is a helper and must finish with a report.
  3. It gets the tool subset for `profile`:

     | Profile | Tools |
     |---|---|
     | `explore` | read-only: read, list, glob, grep, search |
     | `edit` | explore + write and edit + `run` |
     | `research` | read-only + web |
     | `full` | everything except `task` |

  4. Returns the final report (capped at 2K tokens), the files it changed, and its status.
- **`task_batch`** `{prompt_template: str, items: [str], profile, model}` runs one helper per item, with `{item}` substituted into the template. Returns a list of reports. It exists because local models are bad at making many separate parallel calls. The `bulk-summarize` skill uses it for map-reduce.
- **No nesting.** Helpers cannot call `task` or `task_batch`.

### 7.2 Modes

Set in Settings → Helpers:

| Mode | Behaviour |
|---|---|
| **Off** | `task` and `task_batch` are not exposed |
| **One at a time** | Default when the capacity test gives N=1 or has not been run |
| **Parallel up to N** | N is the capacity-test recommendation. The user can override it, with a hard cap of 8. |

### 7.3 Scheduling and constraints

- **Concurrency.** A global semaphore of size N covers helper model calls. The main agent's own calls are not counted, because the main agent waits while helpers run.
- **Batching.** Consecutive `task` calls in one model response, and every `task_batch`, are dispatched together.
- **Using a second model.** If the helper role uses a different model, Hearth checks whether both models fit at once:
  - Ollama: `/api/ps`, plus a warning if loading the helper model would evict the main one.
  - LM Studio: loaded instances.
  - If they do not both fit, helpers are forced to run "One at a time" and the UI explains why.
- **Shared rules.** Helpers share the project's approval rules. Approval requests from helpers show in the main chat, labelled with the helper's name.
- **Editing.** Two `edit` helpers that edit the same file are serialised with a per-file lock. A write fails if the file changed since the helper last read it, and the helper must re-read it.
- **Transcripts.** Each helper's transcript is stored and can be viewed from its card.
- **Skill mapping.** `model:` frontmatter in skills maps to roles: `haiku` → helper; `sonnet`/`opus`/`inherit` → main.

## 8. Tools reference (`tools/`)

Every tool declares a JSON schema, an **approval class** (§9.5) and an output cap. Tool results are text, optionally with image attachments for vision models.

### 8.1 Files

All file tools are confined to the project (§9.1).

| Tool | Args | Behaviour | Class |
|---|---|---|---|
| `read_file` | `path, offset?=1, limit?=400` | Returns lines with numbers. Documents (pdf/docx/xlsx/pptx/html) return converted text with page/sheet/slide markers (§10.3). Images: attached for vision models, otherwise metadata only. Binary files: refused with type and size. | read |
| `write_file` | `path, content` | Creates or overwrites. If the file exists and has not been read in this chat, the write is refused ("read it first"). Parent directories are created. Returns the line count. | edit |
| `edit_file` | `path, old_string, new_string, replace_all?=false` | Exact replace, and `old_string` must be unique unless `replace_all`. If there is no exact match, tries again ignoring trailing whitespace, then with relative indentation normalised; the second match must still be unique. Returns the edited region with line numbers. | edit |
| `list_tree` | `path?=".", depth?=3` | Tree with sizes. Ignores .git, node_modules, .venv, `__pycache__`, `.hearth` and gitignored paths. Capped at about 2K tokens, with a "…N more" marker. | read |
| `glob` | `pattern` | gitignore-aware; at most 200 results | read |
| `grep` | `pattern, path?, glob?, ignore_case?, context?=0` | Uses `rg` if available, otherwise Python. At most 100 matches, each shown as file:line. | read |
| `search` | `query, k?=8, path_prefix?` | Hybrid index search (§10.5). Returns snippets with citations. | read |
| `present_file` | `path, title?` | Shows the file to the user as a deliverable card (open, download, preview) | read |

### 8.2 Execution

| Tool | Args | Behaviour | Class |
|---|---|---|---|
| `run` | `command, timeout?=120 (max 600), cwd?` | One-shot command in the active tier (§9.2/§9.3). Returns the capped output, exit code, duration and whether it timed out. | exec |
| `start_process` | `command, name?` | Background process with its own process group or job and a ring-buffer log. Detects ports (§9.4). Returns `{id, url?}`. | exec |
| `process_output` | `id, since?` | New output since an offset. Also reports status and any port. | read |
| `stop_process` | `id` | Kills the whole process tree | read |
| `list_processes` | | Running processes for this project | read |

### 8.3 Web

| Tool | Args | Behaviour | Class |
|---|---|---|---|
| `web_search` | `query, n?=8` | §11.1 | web |
| `web_fetch` | `url, offset?=0, max_chars?=12000` | §11.2. Returns paged markdown. | web |
| `browser_open` | `url` | Opens the page and returns the title plus an interactive snapshot (page 1) | web |
| `browser_snapshot` | `scope?="interactive"\|"all", page?=1` | §11.3 | web |
| `browser_click` / `browser_type` / `browser_select` | `ref`, plus `text, submit?` / `value` | Acts on the element with `aria-ref` `ref`, then returns a new snapshot | web |
| `browser_scroll` / `browser_back` | `direction` / – | | web |
| `browser_read` | | Current page as clean markdown, via trafilatura | web |
| `browser_screenshot` | | Vision models only: a PNG attachment | web |

### 8.4 Agent and meta

| Tool | Args | Behaviour | Class |
|---|---|---|---|
| `load_skill` | `name, args?` | §12.7 | read |
| `read_skill_file` | `name, path, section?` | Reads a file inside a skill directory | read |
| `todo` | `items:[{text, status:"pending"\|"in_progress"\|"done"}]` | Replaces the to-do list. It is shown pinned in the UI and fed back each turn. | read |
| `ask_user` | `question, choices?` | Pauses the turn and shows a question card; the answer comes back as the result | read |
| `task` / `task_batch` | §7.1 | | inherits from the helper's tools |

### 8.5 GitHub

GitHub tools are available when GitHub is connected (§13).

| Tool | Args | Class |
|---|---|---|
| `github_clone` | `repo ("owner/name" or URL), path?` | outward-read (auto) |
| `git_push` | `remote?="origin", branch?, set_upstream?=true` | outward |
| `git_pull` | `remote?, branch?` | exec |
| `github_create_repo` | `name, private?=true, description?` | outward |
| `github_create_pr` | `title, body, head, base?, draft?=true` | outward |
| `github_list_issues` / `github_get_issue` / `github_get_pr` | filters / number | read |
| `github_comment` | `number, body` | outward |

Local git operations, such as `git status`, `add`, `commit`, `branch` and `diff`, go through `run` and do not need credentials (§13.3).

## 9. Execution, sandbox and approvals (`sandbox/`)

### 9.1 Path confinement for file tools

- **Check.** Each path is resolved with `Path(project, p).resolve()` and must satisfy `is_relative_to(project_root)`.
- **Links.** Symlinks and junctions that resolve outside the project are rejected.
- **Hearth's own folder.** Writes under `.hearth/` are rejected, except to `.hearth/scratch/`. Reads of `.hearth/logs/` are allowed.
- **Reserved names.** Windows reserved names (CON, PRN, AUX, NUL, COM1–9, LPT1–9) are rejected on every OS, so projects stay portable.

### 9.2 Host runner (T1, default)

- **Spawning.** `subprocess.Popen` with argv, `stdin=DEVNULL`, and reader threads feeding a head buffer plus a tail ring buffer. The full output streams to the log file, with a 50 MB cap.
- **Shell.**

  | OS | Shell used |
  |---|---|
  | POSIX | `bash -lc` (or `zsh` when that is the user's shell and bash is missing) |
  | Windows | **Git Bash** when installed, which it is because the installer ensures Git. Otherwise `pwsh` or `powershell -NoProfile -NonInteractive -Command`. |

  The prompt tells the model which shell is active.
- **Process-tree kill.**

  | OS | Mechanism |
  |---|---|
  | POSIX | `start_new_session=True`, then `killpg` with SIGTERM, a 3-second wait, and SIGKILL |
  | Windows | A Job Object with `JOB_OBJECT_LIMIT_KILL_ON_JOB_CLOSE` per command, plus one app-wide job so nothing is orphaned if Hearth crashes |
  | Fallback | Recursive kill with `psutil` |

- **Environment.** The environment is built from an **allowlist**:
  - **Kept:** PATH, HOME/USERPROFILE, LANG/LC_*, TERM, TMP/TEMP, and on Windows SystemRoot, ComSpec, PATHEXT and WINDIR.
  - **Dropped:** anything matching `*KEY*`, `*SECRET*`, `*TOKEN*`, `*PASSWORD*`, `AWS_*`, `OPENAI_*`, `ANTHROPIC_*`, `GH_*` or `GITHUB_*`.
  - **Added:** `CI=1`, `GIT_TERMINAL_PROMPT=0`, `PIP_NO_INPUT=1`, `PYTHONUNBUFFERED=1`, `HOST=127.0.0.1`, `FORCE_COLOR=0`.
  - **Never present:** the LLM API key and the GitHub token, in any tier.
- **Toolchains:**
  - **Python.** On first need, Hearth runs `uv venv .venv --python 3.12`. `VIRTUAL_ENV` and PATH point at it, so `python`, `pip` and `uv pip` work inside the project.
  - **Node.** On first need, Hearth installs Node with `fnm`, using `.node-version`/`.nvmrc` or else the LTS release. Commands run with that Node on PATH.
- **Output.** ANSI codes are stripped, and output is capped per §6.4.

### 9.3 Container runner (T3, optional)

- **Detection.** `docker.from_env().ping()`. Podman works through `DOCKER_HOST`. If neither responds, the toggle is greyed out with install hints (Docker Desktop, Podman Desktop, Rancher Desktop or Colima).
- **One long-lived container per project:**
  - image `ghcr.io/<owner>/hearth-sandbox:<ver>`, built from `docker/sandbox.Dockerfile`: python 3.12-slim, node LTS, uv, git, ripgrep, build-essential
  - `init=True`, `cap_drop=["ALL"]`, `security_opt=["no-new-privileges"]`
  - `pids_limit=512`, `mem_limit` 4 GB and 2 CPUs by default (configurable)
  - the project bind-mounted at `/workspace`
  - non-root user (Linux: the host uid/gid)
  - ports 3000, 5173, 8000 and 8080 pre-published to random `127.0.0.1` host ports
  - network `bridge` or `none`, switched at runtime with network connect/disconnect
- **Running commands.** `exec_run(["bash","-lc","timeout -k 5 <t> bash -lc <quoted>"], stream=True, demux=True)`.
- **Keeping host and container separate.**
  - The container's Python venv lives in a named volume, set with `UV_PROJECT_ENVIRONMENT=/home/agent/.venv`. `node_modules` is installed by the container inside the container.
  - When a project switches tier, Hearth warns that dependencies will be reinstalled.
- **Windows.** The polling variables `CHOKIDAR_USEPOLLING=1` and `WATCHPACK_POLLING=true` are set. The UI advises keeping container projects inside WSL for speed.
- **Isolation guarantees.** GitHub network operations and the browser always run on the host, inside Hearth. File tools operate on the bind-mounted folder from the host.

### 9.4 Processes, ports and preview

- **Registry.** Each project has a process registry with ring-buffer logs. Every process is killed when the project closes or Hearth exits.
- **Ports.** Hearth passes `PORT` (a free port), which dev servers usually honour. A port counts as detected when either:
  - the output matches the regex `https?://(localhost|127\.0\.0\.1|0\.0\.0\.0|\[::\]):(\d+)`
  - a LISTEN socket is owned by a descendant process (`psutil`)

  In container mode, Hearth maps the published port.
- **Preview tab.** An iframe pointing at `http://127.0.0.1:<port>`. It disappears when the process exits. The agent can look at its own app with `browser_open` on the same URL.
- **Terminal tab.** A PTY in the project, using `pywinpty` or `ptyprocess`, connected to xterm.js over WebSocket. In container mode it is an exec PTY. This terminal is the user's own and is never driven by the model.

### 9.5 Approvals

**Approval classes**

| Class | Examples | Ask every time | Smart (default) | Autonomous (container only) |
|---|---|---|---|---|
| read | read_file, grep, search, todo | auto | auto | auto |
| web | web_search, web_fetch, browser_* | auto | auto | auto |
| edit | write_file, edit_file in project | **ask** | auto (diff shown, undoable) | auto |
| exec | run, start_process, git_pull | **ask** | auto only for read-only commands or user-saved allow rules; otherwise **ask** | auto unless danger |
| outward | git_push, github_create_repo/pr, github_comment | **ask** | **ask** | **ask** |
| danger | see list below | **ask (red)** | **ask (red)** | **ask (red)** |

**Rule engine.** Rules are stored per project and evaluated **deny > ask > allow**. Patterns look like `run: pytest *` or `run: npm test`. The approval card offers four answers:
- **Approve once**
- **Always allow `<prefix>` in this project.** Hearth suggests the prefix, e.g. `pytest`, and never a bare interpreter such as `python` or `bash`.
- **Deny**
- **Deny with a note.** The note goes back to the model.

**Command parsing fails closed**
- **POSIX shells.** `bashlex` parses the command, which is split on `&&`, `||`, `;`, `|`, `&` and newlines. Every simple command must be allowed on its own. Wrappers such as `timeout`, `nice`, `nohup` and `env` are stripped. Any parse failure, `$(…)`, backticks, `eval`, `source`, or `sh|bash|zsh -c` → ask.
- **PowerShell.** Tokenised conservatively. Any `;`, `|`, `&`, `$(`, `iex`, `Invoke-Expression` or `-EncodedCommand` → ask.
- **Built-in read-only allowlist** (auto in Smart mode):
  - listing and reading: `ls`, `dir`, `pwd`, `cat`, `type`, `head`, `tail`, `wc`, `tree`, `stat`, `file`, `which`, `where`, `echo`
  - searching: `grep`, `rg`, and `find` without `-exec`, `-delete` or `-ok`
  - git: `status`, `diff`, `log`, `show`, `branch` (list), `rev-parse`, `remote -v`
  - version checks: `<tool> --version`
  - None of their path arguments may resolve outside the project.

**Danger list (always a red prompt, never auto-approved)**
- **Recursive deletes:** `rm -r`/`-rf`, `rmdir /s`, `del /s`, `Remove-Item -Recurse`
- **Destructive git:** `git push --force`/`-f`, `git reset --hard`, `git clean -fdx`
- **Permissions and disks:** `chmod -R`/`chown -R`, `dd`, `mkfs`, `format`, `diskpart`, `reg delete`
- **Privilege escalation:** `sudo`, `runas`, `doas`
- **Download piped to a shell:** `curl …|sh`, `wget …|bash`, `iex (irm …)`
- **Publishing and pruning:** `npm publish`, `twine upload`, `docker system prune`, `shutdown`, `reboot`
- **Outside the project:** any redirect or path argument that resolves outside the project
- **Sensitive files:** reading `~/.ssh`, `~/.aws`, `.env` files outside the project, or browser profile folders
- **Global installs:** `pip install --user`, `npm i -g`

**Taint (prompt-injection guard).** In Smart mode, auto-approval of the **exec** class is suspended for the rest of the turn once any of these happens:
- a result from `web_fetch`, `browser_*` or `web_search` enters the turn
- a script from a **third-party** skill (not Hearth-native) is executed

During the suspension, every exec needs approval except the built-in read-only allowlist, and saved allow rules are paused too. A banner explains why. The user's own project files do **not** count as taint.

**Visible tier.** Every approval card and the chat header show a tier badge, either "Host (unsandboxed)" or "Container · network off/on".

### 9.6 Snapshots and undo

- **Before every agent turn.** Hearth snapshots the project into `.hearth/snapshots.git`, using `GIT_DIR` with the project as `GIT_WORK_TREE`, so the user's own `.git` is never touched. The snapshot:
  - respects `.gitignore`
  - always excludes `.git`, `.hearth`, `node_modules`, `.venv`, `__pycache__` and `dist`
  - skips files larger than 50 MB, listing them in the turn's notes
- **Undo turn.** Restores every file the turn changed to its snapshot state, and deletes files the turn created.
  - If the user edited a file after the turn, Hearth asks before overwriting it.
  - Undo is available for the last 20 turns.
  - Snapshots older than 30 days are pruned, using `git gc`.
- **Changes tab.** Shows `git diff` between the turn's snapshot and the current files.

## 10. Projects, ingestion and index (`workspace/`)

### 10.1 Projects

**New project dialog:**

| Option | What it does |
|---|---|
| **Blank** | An empty folder |
| **Template** | `python-app` (uv, pytest), `web-app` (Vite + React + TS), `api-server` (FastAPI + pytest), `data-analysis` (uv, pandas, a `data/` folder), `documents` (an empty `files/` folder; the document-qa skill is pinned) |
| **Upload** | Files, a folder or archives (§10.2) |
| **Clone from GitHub** | Searchable list of the user's repos, including private ones, or a URL. Optional branch. |
| **Open existing folder** | A path picker on the server side, because a browser cannot pick local paths |

- **Names.** Project names are filename-safe. Folders go under `~/HearthProjects/` unless the user chose an existing folder.
- **Settings per project:** tier, network on or off, approval mode, pinned skills, rules.
- **Quick chat.** A project-less chat backed by a scratch project in `<data>/scratch/<chat-id>`.

### 10.2 Upload and safe extraction

- **Streaming uploads.** Uploads are chunked multipart streamed to `.hearth/uploads/`, with progress shown in the UI. The default cap is 4 GB per upload, configurable.
- **Archives:**
  - `.zip` via `zipfile` with our own checks
  - `.tar`, `.tar.gz`, `.tgz`, `.tar.bz2` and `.tar.xz` via `tarfile` with `filter="data"`
  - `.7z` via `py7zr`, an optional extra
- **Checks on every entry:**
  - the resolved destination must stay inside the target directory
  - symlinks and hardlinks are rejected
  - absolute paths and drive letters are rejected
  - Windows reserved names are rejected
  - **Limits:**

    | Limit | Default |
    |---|---|
    | Total uncompressed size | 20 GB, or the free disk space minus 2 GB if that is smaller |
    | Per-file size | 2 GB |
    | Entry count | 200,000 |
    | Compression ratio per entry | > 200 is rejected |
    | Nested-archive depth | 2 |

  - Every violation is reported per entry, and the rest of the archive still extracts.
- **Where uploads land.** Uploads go to `uploads/<archive-name>/` by default.
  - If the project is empty and the archive looks like a codebase, Hearth offers "Use as project root". A codebase has a root marker such as `package.json`, `pyproject.toml`, `go.mod`, `Cargo.toml`, `.git` or `src/`. A single top-level folder is unwrapped first.
  - Users can drop files onto a specific folder in the Files tree.

### 10.3 Conversion

Converted text is cached in `index.db`, keyed by path and content hash, and is used by `read_file` and the indexer.

| Type | Library | Output anchors |
|---|---|---|
| PDF | pypdfium2 text per page | `[page N]` |
| DOCX | python-docx (paragraphs, headings, tables → markdown) | heading path |
| XLSX/XLSM/CSV | openpyxl (values; formulas shown as `=…` in an extra column) / csv | `[sheet Name]`, row ranges |
| PPTX | python-pptx (titles, text, notes) | `[slide N]` |
| HTML | trafilatura → markdown | headings |
| Code/text/JSON/MD | as-is | line numbers |
| Images | metadata; description on demand if vision model; optional OCR extra (`rapidocr`) | – |
| Scanned PDF (no text layer) | flagged "needs OCR"; OCR extra if installed | `[page N]` |

### 10.4 Index

**Storage.** Everything lives in `index.db`:
- `chunks(id, path, anchor, breadcrumb, text, hash)`
- an FTS5 table using the `porter unicode61` tokenizer
- an FTS5 trigram table for code identifiers
- a `sqlite-vec` table when an embedding model is configured

**Chunking**
- **Size.** 300–500 tokens, split along structure: headings, pages, slides, sheets, or functions and classes in code.
- **Breadcrumb.** Each chunk is prefixed with a breadcrumb, e.g. `docs/report.pdf > p.12 > Revenue`.
- **Code.** Hearth uses regex-based symbol detection for Python, JS/TS, Go, Rust, Java, C/C++ and C#. Tree-sitter is an optional extra.

**Embeddings**
- **Endpoint.** The `embedding` role is called through `/v1/embeddings` or Ollama `/api/embed`, in batches of 32.
- **Recommended model.** `nomic-embed-text`. The fit card suggests installing it when it is missing.

**Background job**
- **Progress.** The progress bar shows files converted, chunks indexed and embeddings computed.
- **Incremental.** A `watchfiles` watcher, together with the agent's own edits, triggers re-indexing of changed files only.

**Project map**
- **File tree overview.** About 1.5K tokens, regenerated on change.
- **Code map.** The top symbols per file for code projects.
- **Usage.** `list_tree` and the `codebase-onboarding` skill use the map.

### 10.5 Search

`search(query)` works in four steps:
1. Run BM25 over the FTS5 table, with trigram matching added when the query looks like an identifier.
2. Run vector KNN when embeddings exist.
3. Merge the two with Reciprocal Rank Fusion (k = 60).
4. Return the top k snippets, about 600 characters each, as `[path anchor] text`.

**Citations.** The model is told to cite as `[path p.N]` or `[path#L10-20]`. The UI turns these into links that open the file viewer at that anchor.

## 11. Web (`web/`)

### 11.1 Search providers

Providers are tried in this order:
1. **SearXNG**, if a URL is configured. It must have the `json` format enabled; the docs explain how.
2. **A keyed provider**, if a key is set: Tavily, Brave or Serper.
3. **`ddgs`**, trying backend `auto`, then `bing`, then `brave`.

**Handling results**
- **Filtering.** Only absolute `http(s)` URLs are kept, and duplicates are removed.
- **Caching.** Results are cached for 24 hours in `<data>/cache/web.db`.
- **Pacing.** At most one query every 2 seconds per provider, with exponential backoff and jitter on rate limits.
- **When everything fails.** The tool error says so, and the UI suggests configuring SearXNG or a key.

### 11.2 Fetch

1. Fetch with `httpx`: follow redirects, 20-second timeout, 10 MB cap, a browser-like user agent, and robots.txt is not enforced. This is a user-initiated agent, the same model as a browser.
2. Extract with `trafilatura.extract(output_format="markdown", include_tables=True)`.
3. **Fallback.** If the extracted text is under 500 characters, or the content type suggests a JavaScript app, render the page in the browser and extract again.
4. **PDF URLs** go through the PDF converter.
5. **Paging.** Results are paged by `offset` and `max_chars`, and the source URL is always included for citation.

### 11.3 Browser

- **Engine.** Playwright Python with headless Chromium in a dedicated thread. Every call is marshalled to that thread.
- **Install.** On first use Hearth asks: "Download browser (~115 MB)?". It then runs `playwright install chromium --only-shell`. On Windows, Hearth offers `channel="msedge"` instead, which needs no download.
- **Profile.** A persistent but isolated profile in `<data>/cache/browser-profile`. It never uses the user's real browser profile. "Clear browser data" is in Settings.
- **One page per chat.** It closes after 10 minutes idle.
- **Snapshots.**
  - Built from `page.aria_snapshot(mode="ai")`, filtered to interactive roles (link, button, textbox, combobox, checkbox, radio, menuitem, tab, option, searchbox, slider, switch) plus headings for orientation.
  - Shown as numbered lines like `e12 button "Sign in"`, in pages of 60 elements.
  - `scope="all"` gives the full tree, depth-limited and paged.
- **Actions.** `page.locator("aria-ref=eN")`. After each action, Hearth waits for `networkidle` (3-second cap) and returns a new snapshot.
- **Screenshots.** Vision models get screenshots with numbered set-of-marks overlays, drawn from `boxes=True`.
- **Downloads** are saved into the project's `downloads/` folder.
- **UI.** The Browser tab shows a live screenshot that refreshes after each action, plus the current URL.

## 12. Skills (`skills/`)

### 12.1 Format

Hearth uses standard Agent Skills: a directory containing `SKILL.md` with YAML frontmatter (`name`, `description`, and optionally `license`, `allowed-tools`, `model`, `argument-hint`, `disable-model-invocation`, `user-invocable`), plus optional `scripts/`, `references/` and `assets/`. Unknown frontmatter keys are ignored.

### 12.2 Discovery sources

When two skills share a name, the source higher in this list wins:

1. **Project:** `<project>/.agents/skills/*`, `<project>/.claude/skills/*`
2. **User:** `<data>/skills/*`, created or installed through the UI
3. **Hearth-native:** `hearth/skills_builtin/*`
4. **Bundled packs:** `hearth/skills_packs/<pack>/*`
5. **Claude import (opt-in, read in place):**
   - `~/.claude/skills/**/SKILL.md`, including `synced/*`
   - `~/.claude/plugins/**/skills/*/SKILL.md`
   - `~/.agents/skills/*`

Skill IDs are namespaced `<source-or-pack>:<name>`. A bare `name` resolves by the precedence above. The Skills page shows duplicates.

### 12.3 Licence guard

Every skill is classified when it is discovered:

| Class | Rule | Effect |
|---|---|---|
| **open** | A LICENSE file in the skill or plugin root, or a frontmatter `license:`, matches a known permissive licence (MIT, Apache-2.0, BSD-2/3, ISC, MPL-2.0, CC-BY-4.0, Unlicense) | Loadable, exportable and bundleable |
| **local-only** | No licence found | Loadable **only from the user's own disk**. Never copied, exported, bundled or shared. Badge: "No licence: local use only". |
| **blocked** | Licence text contains a proprietary marker, e.g. "All rights reserved" together with "ADDITIONAL RESTRICTIONS" or prohibitions on extraction or retention outside a service | **Not loaded.** Shown greyed out with the reason. |

The blocked class covers Anthropic's docx, pdf, pptx, xlsx, chrome-browser, built-in-browser and computer-use skills. Hearth-native skills (§12.9) replace them.

### 12.4 Bundled packs

**How packs are vendored**
- **Source of truth.** `scripts/vendor_skills.py` and `skills_packs/manifest.yaml`, which records each pack's upstream repo, pinned commit, licence, and include and exclude lists.
- **Fallback.** Packs are fetched from upstream at the pinned commit. If upstream cannot be reached at build time, they are copied from a local `~/.claude` install, with the provenance recorded.
- **Files stay unmodified.** All adaptation happens at load time (§12.5), so the Apache-2.0 "mark modified files" requirement does not apply. Any deleted file is listed in the pack's `NOTICE`.

**Each pack directory contains:**
- `LICENSE`, copied verbatim from upstream
- `NOTICE`, with attribution, the upstream URL and commit, and the files removed
- the skills themselves

**Packs**

| Pack | Licence | Default |
|---|---|---|
| `superpowers` (obra/superpowers) | MIT | on |
| `engineering-pocock` (mattpocock/skills) | MIT | on |
| `napkin-math` | MIT | off |
| `anthropic-engineering`, `anthropic-data`, `anthropic-design`, `anthropic-productivity` (anthropics/knowledge-work-plugins) | Apache-2.0 | on |
| `anthropic-marketing`, `anthropic-finance`, `cowork-plugin-management` | Apache-2.0 | off |
| `anthropic-skills-open`: skill-creator, theme-factory, web-artifacts-builder, learn | Apache-2.0 | on (skill-creator, web-artifacts-builder); off (others) |
| `remotion` | MIT | off |
| `fastly`, `prefect`, `product-tracking` | per upstream LICENSE (MIT / Apache-2.0) | off |
| `typeui` design skills (bold, futuristic, minimal, premium, sleek) | MIT, declared in frontmatter | off |

**Fixes and exclusions, in the manifest**
- **Removed or fixed:**
  - superpowers `writing-skills/anthropic-best-practices.md` is removed (it is a copy of Anthropic documentation) and replaced with a link
  - an MIT notice is added for the shadcn tarball in web-artifacts-builder
  - an MIT notice is added for the typeui skills and Remotion
  - the `pr` skill's "Summary" section is checked against humanlayer/skills and excluded if it is unclear
  - remotion-maps sample geodata gets ODbL attribution, or is excluded
- **Excluded:** `implement-eval` (Prefect-internal), `diagnosing-superpowers`, `scaffold-exercises`, `setup-matt-pocock-skills`, `git-guardrails-claude-code` (Claude Code specific), and the AccelByte skills (not redistributable).
- **Overlapping pairs.** One of each overlapping pair is off by default, e.g. `tdd` vs `test-driven-development`.

**Small-model safety.** The `using-superpowers` bootstrap is **off by default** when the profile is `small` or `medium`, because its "MUST" wording over-triggers on small models.

### 12.5 Adaptation at load time

Skill files are never modified on disk. Hearth adapts them when it loads them:

- **Tool-name map.** Hearth injects this preamble when a skill is loaded:

  | Claude name | Hearth tool |
  |---|---|
  | Read / Write / Edit | `read_file` / `write_file` / `edit_file` |
  | Bash | `run` |
  | Glob / Grep | `glob` / `grep` |
  | WebSearch / WebFetch | `web_search` / `web_fetch` |
  | Skill | `load_skill` |
  | Agent, Task (subagent) | `task` |
  | TodoWrite, TaskCreate/TaskUpdate/TaskList | `todo` |
  | AskUserQuestion | `ask_user` |
  | SendUserFile, present_files | `present_file` |
  | NotebookEdit | `edit_file` |
  | ToolSearch, EnterPlanMode/ExitPlanMode | not needed (ignore) |
  | `mcp__*`, `~~connector` placeholders | "Not connected in Hearth; use the skill's fallback or ask the user" |

- **Substitutions:**
  - `$ARGUMENTS`, `$1…$9` → the user's arguments
  - `${CLAUDE_PLUGIN_ROOT}` → the plugin directory
  - `${CLAUDE_SKILL_DIR}` → the skill directory
  - `@path` includes → inlined, capped at 4K tokens
- **Project instructions.** `CLAUDE.md` is treated as project instructions, alongside `AGENTS.md` and `HEARTH.md`.
- **Frontmatter:**
  - `disable-model-invocation: true` → the skill appears only in the `/` menu
  - `user-invocable: false` → the skill is offered only to the model
  - `allowed-tools` → while the skill is active, only those tools plus `load_skill` and `todo` are exposed
  - `model` → mapped to a role (§7.3)
- **Sibling bundling.** Skills that reference siblings get them too. For example, executing-plans loads together with subagent-driven-development and requesting-code-review.
- **Generated reference for superpowers.** Hearth generates `using-superpowers/references/hearth-tools.md` in memory, using the same mechanism superpowers uses for other harnesses.

### 12.6 Selection for the system prompt

- **Pinned skills.** Pinned skills, per project or globally, are always listed.
- **Relevance ranking.** The remaining enabled skills are ranked by BM25 over their name and description against the latest user message and the project type. The top K are listed:

  | Profile | K |
  |---|---|
  | small | 5 |
  | medium | 10 |
  | full | 25 |

  In the full profile the catalogue is capped at 100 lines.
- **Description length.** Each description is truncated to 250 characters.
- **Slash commands.** Typing `/name args` in the chat loads that skill directly into the turn, as if `load_skill` had been called.

### 12.7 `load_skill`

- **Short skills.** If `SKILL.md` is at most the profile limit, it is returned in full, after substitutions and with the tool-map preamble. The limits are 3K tokens (small), 6K (medium) and 12K (full).
- **Long skills.** Otherwise Hearth returns the heading outline, the first section, and a note to use `read_skill_file(name, "SKILL.md", section="<heading>")` for the rest.
- **What else is listed.** Bundled files in `scripts/` and `references/`, with one-line descriptions where available.

### 12.8 Skill scripts

- **Where they run.** Scripts run through `run` in the active tier. Their directory is exposed read-only: the host tier enforces this by path checks, and the container tier mounts it at `/skills/<id>` with `:ro`.
- **Approval.** A third-party skill script counts as **exec** and triggers taint (§9.5).
- **Dependencies.** Python dependencies named in the skill (e.g. `pip install pypdf`) are installed into the project venv after the normal exec approval.

### 12.9 Hearth-native skills

These live in `hearth/skills_builtin/` and are MIT-licensed. They are written from scratch for mid-size models: short numbered steps, explicit tool calls, done-criteria checklists, and few words. Each one ships `evals.yaml` with 3–5 test prompts and checks (§20.4).

| Skill | Purpose |
|---|---|
| `coding-loop` | explore → plan (todo) → edit → run tests → fix; never claim done without a passing run; small diffs |
| `codebase-onboarding` | map an uploaded codebase: entry points, build/test commands, architecture notes → `HEARTH.md` draft |
| `project-setup` | detect stack, create venv/node env, install deps, find and run the test command |
| `github-workflow` | branch naming, commits, `git_push`, PR description, never force-push |
| `document-qa` | search first, read cited chunks, answer with citations, say "not found in the provided files" when absent |
| `bulk-summarize` | map-reduce over many files with `task_batch` (or sequentially), then synthesise |
| `web-research` | plan queries, read ≥3 independent sources, cross-check, cite URLs, date-stamp facts |
| `browser-automation` | snapshot → act by ref → re-snapshot; handle logins/forms; when to use `web_fetch` instead |
| `word-docs` | create/edit .docx with python-docx (headings, tables, images, styles), verify by re-reading |
| `spreadsheets` | create/edit .xlsx with openpyxl (formulas, formatting, charts), verify by loading |
| `slides` | create .pptx with python-pptx (layouts, text, images, charts), verify slide count/content |
| `pdf-tools` | read/extract (pypdfium2), merge/split/rotate (pypdf), fill forms (pypdf), create (reportlab) |
| `charts-and-data` | pandas analysis + matplotlib charts saved as files + `present_file` |
| `web-app-builder` | scaffold a Vite app or static page, run dev server, check it in the browser, present |
| `delegation` | when to use `task`/`task_batch`, how to write helper prompts, how to merge reports |
| `skill-authoring` | how to write a good SKILL.md for local models (used by the skill creator) |

The four document skills (`word-docs`, `spreadsheets`, `slides`, `pdf-tools`) are **clean-room**. They are written only from the open libraries' own documentation, with no reference to Anthropic's proprietary skills.

### 12.10 Skill management UI

- **Skills page.** Lists every skill with source, licence badge, enabled and pinned toggles, and duplicate markers. The full skill can be viewed rendered.
- **Import from Claude.** Scans the paths in §12.2 item 5 and shows what was found, grouped by licence class. Enabling a skill does not copy it.
- **Install from GitHub URL or zip.** Treated as installing code: Hearth shows the file list and licence, then asks for approval before extracting to `<data>/skills/`.
- **New skill (creator).** A four-step flow:
  1. The user describes the skill.
  2. The main model drafts `SKILL.md`, guided by the `skill-authoring` skill and the Apache-licensed skill-creator.
  3. The user edits it in CodeMirror with a live preview.
  4. The user runs it against sample prompts in a scratch project, then saves it to `<data>/skills/`.

## 13. GitHub (`integrations/github/`)

### 13.1 Connecting

**Device flow ("Sign in with GitHub")**
1. Hearth calls `POST https://github.com/login/device/code` with `client_id` and `scope=repo`.
2. The UI shows the user code and a link to `https://github.com/login/device`.
3. Hearth polls `POST https://github.com/login/oauth/access_token` at the given interval until approved, expired or denied.

**The `client_id` setup step**
- **What it needs.** The `client_id` belongs to a GitHub OAuth App with device flow enabled. It is public, so no secret ships.
- **Who registers it.** The project owner registers it once; the README has step-by-step instructions. It goes in `hearth/integrations/github/oauth.py` or the `HEARTH_GITHUB_CLIENT_ID` environment variable.
- **Until it exists,** the device-flow button is hidden.

**Token paste (alternative)**
- **Token types.** A fine-grained token with Contents RW, Pull requests RW, Issues RW, Metadata R and Administration RW (to create repos), or a classic token with `repo`.
- **Validation.** Hearth validates it with `GET /user`.

**Storage and disconnecting**
- **Storage.** The token goes in the keychain (§4.3). The UI shows the login and avatar.
- **Disconnect.** Deletes the token from the keychain.

### 13.2 Repository listing and cloning

- **Listing.** `GET /user/repos?per_page=100&sort=updated`, paginated and searchable, in the New project dialog.
- **Cloning.** Done by Hearth itself, not the model's shell: `git clone` with credentials injected through environment variables `GIT_CONFIG_COUNT=1`, `GIT_CONFIG_KEY_0=http.https://github.com/.extraheader`, `GIT_CONFIG_VALUE_0="AUTHORIZATION: basic <b64(x-access-token:TOKEN)>"`. The token is never written to `.git/config`.

### 13.3 Agent git operations

- **Local operations.** `status`, `add`, `commit`, `branch` and `diff` go through `run`, with no credentials needed.
  - Commits use the user's git identity when configured. Otherwise Hearth uses the GitHub login's noreply address, after asking once.
- **Network operations.** `git_push`, `git_pull` and `github_clone` are **dedicated tools that Hearth executes** with the credential injection described above.
  - If the model tries `git push` through `run`, it fails without credentials. The tool error then points to `git_push`.
  - The model never sees the token.
- **API operations.** Repos, PRs, issues and comments use the GitHub REST API through httpx, with the approval classes in §8.5.
- **Force pushes** are never offered by `git_push`. A force push through `run` is on the danger list and would also fail without credentials.

## 14. API (`api/`)

### 14.1 Authentication and origin checks

- **Token.** At startup Hearth generates a random 32-byte token and opens `http://127.0.0.1:<port>/?t=<token>`. The server exchanges the token for an `HttpOnly`, `SameSite=Strict` cookie and redirects to strip it from the URL.
- **Every HTTP and WebSocket request** must have:
  - the cookie, or `Authorization: Bearer <token>` for the CLI
  - a `Host` header of `127.0.0.1:<port>` or `localhost:<port>`
  - an `Origin` header matching that, or none for non-browser clients
- **Cross-origin requests** are rejected, which blocks DNS rebinding and malicious-page attacks.
- **`hearth url`** prints a fresh login URL.

### 14.2 REST endpoints

All endpoints are under `/api`.

| Area | Endpoints |
|---|---|
| System | `GET /health`, `GET /system` (OS, tiers available, docker, git, browser installed, hardware) |
| Connections | `GET/POST /connections`, `PATCH/DELETE /connections/{id}`, `POST /connections/{id}/probe`, `GET /connections/{id}/models`, `POST /connections/{id}/fix-context` |
| Models | `GET/PATCH /model-roles` (main/helper/embedding), `GET /model-profiles/{key}`, `POST /capacity-test` |
| Projects | `GET/POST /projects` (`kind: blank\|template\|upload\|clone\|open`), `GET/PATCH/DELETE /projects/{id}` (delete = forget; deleting files is a separate confirmed action) |
| Files | `GET /projects/{id}/tree`, `GET/PUT /projects/{id}/file?path=`, `POST /projects/{id}/upload` (streaming multipart), `POST /projects/{id}/extract`, `GET /projects/{id}/index/status`, `GET /projects/{id}/download?path=` |
| Chats | `GET/POST /projects/{id}/chats`, `GET/PATCH/DELETE /chats/{id}`, `GET /chats/{id}/events?after=<seq>`, `POST /chats/{id}/undo` (`{turn_id}`) |
| Rules | `GET/POST/DELETE /projects/{id}/rules` |
| Processes | `GET /projects/{id}/processes`, `POST /projects/{id}/processes/{pid}/stop` |
| Skills | `GET /skills`, `PATCH /skills/{id}` (enabled, pinned), `POST /skills/scan-claude`, `POST /skills/install`, `POST /skills` (create), `GET/PUT /skills/{id}/files?path=` |
| GitHub | `POST /github/device/start`, `POST /github/device/poll`, `POST /github/token`, `GET /github/me`, `GET /github/repos?q=`, `DELETE /github` |
| Settings | `GET/PATCH /settings` |
| Browser | `POST /browser/install`, `GET /chats/{id}/browser/screenshot` |

### 14.3 WebSocket protocol

There are two sockets:
- `GET /api/chats/{id}/ws` for the chat
- `GET /api/projects/{id}/terminal` for the terminal, a raw PTY stream

**Client → server messages**
```json
{"type":"user_message","text":"…","attachments":["path"],"skill":"name?"}
{"type":"stop"}
{"type":"approval","request_id":"…","decision":"approve|always|deny","rule":"pytest *?","note":"…?"}
{"type":"answer","request_id":"…","text":"…"}
{"type":"resume","after_seq":123}
```

**Server → client events**
- **Fields.** Every event has `seq`, `turn_id`, `ts`, `type` and `data`.
- **Persistence.** Every event except deltas is persisted. Deltas are coalesced into the final message events.

| `type` | data |
|---|---|
| `turn.started` / `turn.finished` | status: `done\|stopped\|error\|interrupted\|paused_max_steps` |
| `text.delta` / `thinking.delta` | text |
| `message` | final assistant message (text, thinking) |
| `tool.call` | id, name, args, approval_class |
| `tool.result` | id, ok, summary, output (capped), log_path, attachments |
| `approval.request` / `approval.resolved` | id, tool, args, class, tier, danger, suggested_rule / decision |
| `ask.request` / `ask.resolved` | question, choices / answer |
| `todo.updated` | items |
| `helper.started` / `helper.progress` / `helper.finished` | helper id, description, status, report |
| `context.usage` | used, limit, threshold |
| `context.compacted` | summary |
| `nudge` | kind, text (guard fired) |
| `snapshot` | turn_id, files_changed |
| `error` | code, message, hint |

## 15. Frontend (`frontend/`)

### 15.1 Layout

- **Left sidebar:**
  - Projects (+ New), each with its chats; Quick chat
  - Skills and Settings at the bottom
  - **Model chip:** model name, fit label colour, context meter, tool mode, and helper mode with N
- **Centre, the chat:**
  - **Messages.** Markdown with code highlighting and copy buttons. Thinking is collapsible.
  - **Tool cards.** Icon, name, a one-line argument summary, a status spinner or ✓/✗, the result summary, and an expandable raw output with a link to the full log.
  - **Approval cards.** The command or diff, the tier badge, a danger styling when relevant, and the Approve / Always / Deny buttons with an optional note.
  - **Pinned at the top:** the to-do checklist and the taint banner.
  - **Helper cards.** Live status, expandable to the helper's transcript.
  - **Deliverables and citations.** Deliverable cards, clickable citations, and "context compacted" dividers.
  - **Input box.** Attach by drag-and-drop or button, a `/` skill menu, `@` file mentions, an approval-mode selector, the tier badge, and Send/Stop.
- **Right panel tabs** (resizable and collapsible):

  | Tab | Contents |
  |---|---|
  | **Files** | Tree, viewer for code, markdown, images, PDF, docx/xlsx/pptx as converted text, and a CodeMirror editor |
  | **Changes** | Per-turn diff and **Undo turn** |
  | **Terminal** | The user's terminal |
  | **Preview** | Detected ports in an iframe, plus HTML, PDF and image preview |
  | **Browser** | The agent's browser screenshot and URL |
  | **Index** | Ingestion and indexing progress and errors |

- **Settings pages:**
  - Models: connections, probe results, fit card, context fix, roles, sampling
  - Helpers: mode, N, capacity test
  - Execution: tier, Docker status, defaults
  - Web: providers and keys, browser install or clear
  - GitHub
  - Skills packs
  - Storage: data dir, projects root
  - About: version, licences

### 15.2 Behaviour

- **Responsive.** Below 900 px the right panel becomes a drawer, and the sidebar collapses below 640 px.
- **Themes.** Light and dark, following the OS by default.
- **Keyboard shortcuts.** `Enter` sends, `Shift+Enter` adds a new line, `Esc` stops, `Ctrl/Cmd+K` jumps to a project or chat, `/` focuses skills.
- **Accessibility.** Radix primitives, visible focus states, and WCAG AA contrast in both themes.
- **First-run wizard:**
  1. Detect local LLM servers.
  2. Choose a model and run the probe.
  3. Show the fit card with fixes.
  4. Optionally connect GitHub.
  5. Optionally install the browser.
  6. Create the first project.

## 16. Storage schema (app DB)

Schema migrations are applied at startup and versioned in `store/migrations/NNN_*.sql`.

```
connections(id, name, base_url, backend, api_key_ref, created_at)
model_roles(role PK, connection_id, model)
model_profiles(key PK /* connection_id:model */, json, probed_at)
projects(id, name, path, tier, network, approval_mode, pinned_skills_json, created_at, last_opened_at)
chats(id, project_id, title, created_at, updated_at, archived)
turns(id, chat_id, idx, status, snapshot_ref, started_at, finished_at)
events(chat_id, seq, turn_id, ts, type, data_json, PRIMARY KEY(chat_id, seq))
messages(chat_id, idx, role, content_json, tokens, compacted /* bool */)  -- model-visible history
rules(id, project_id, tool, pattern, decision /* allow|ask|deny */, created_at)
skills_state(skill_id PK, enabled, pinned_global)
helper_runs(id, chat_id, turn_id, description, profile, model_role, status, report, transcript_json)
settings(key PK, value_json)
```

## 17. Security model

| Threat | Mitigation |
|---|---|
| Another site or LAN device drives Hearth | Loopback bind, random token cookie, Host and Origin checks on HTTP and WS (§14.1) |
| Model escapes the project through file tools | Resolve-and-confine paths, symlink checks (§9.1) |
| Model runs destructive host commands | Approval modes, fail-closed parsing, danger list, visible tier, per-turn undo, optional container (§9) |
| Prompt injection from web pages, docs or third-party skills | Taint suspends exec auto-approval; outward actions always ask; tokens never in the model's environment (§9.5, §13.3) |
| Secret exfiltration | Environment allowlist; keychain storage; GitHub network ops run by Hearth; secrets never logged (a redaction filter on logs) |
| Zip slip or zip bomb | Safe extraction with caps (§10.2) |
| Malicious skill install | Treated as a code install: licence and file list shown, approval needed; scripts flagged as third-party |
| Proprietary content redistribution | Licence guard: blocked and local-only classes (§12.3) |
| Supply chain | Pinned dependencies with hashes (`uv.lock`, `package-lock.json`); the sandbox image is pinned by digest; no telemetry |

## 18. Error handling

| Situation | Behaviour |
|---|---|
| Model server unreachable | A banner with a retry button. The current turn pauses and resumes on retry, or Stop. |
| `ToolsUnsupported` mid-session | Switch the profile to mode C and continue, with a notice in the UI |
| Context overflow | Compact and retry once, then show an error with fixes (§6.4) |
| Tool throws | Caught; the model receives `{"error": "...", "hint": "..."}` as the tool result; the UI shows ✗. A turn is never aborted because a tool failed. |
| Approval timeout | None; the turn waits. The user can Stop. |
| Hearth restart mid-turn | Processes from that turn are killed. The turn is marked `interrupted`, and the UI offers Continue. |
| Index or conversion failure for a file | Logged per file in the Index tab; ingestion continues |
| Docker disappears | The project falls back to the host tier only after the user confirms. Until then, exec tools return an error. |
| Disk full | Writes and uploads fail with a clear message; extraction stops cleanly; partial files are removed |

## 19. Packaging and distribution

### 19.1 Repository layout

```
hearth/                  Python package (backend)
  __main__.py cli.py server.py config.py
  api/ agent/ llm/ tools/ sandbox/ workspace/ web/ skills/ integrations/github/ store/
  skills_builtin/        Hearth-native skills (MIT)
  skills_packs/          vendored third-party packs (each with LICENSE + NOTICE) + manifest.yaml
  web_dist/              built UI (gitignored; produced by CI; included in wheel)
frontend/                React + Vite + TS source
tests/                   pytest: unit/, integration/, e2e/ ; fixtures/ ; mockllm/
evals/                   acceptance scenarios (§20.4) + fixtures
scripts/                 vendor_skills.py, build_ui.py, make_fixtures.py
installers/              install.sh, install.ps1
docker/                  sandbox.Dockerfile
docs/                    user docs + superpowers/specs + superpowers/plans
pyproject.toml uv.lock LICENSE (MIT) THIRD_PARTY_NOTICES.md README.md
```

### 19.2 CLI

```
hearth                         start server + open browser
hearth --port N --no-browser   options
hearth url                     print a fresh login URL
hearth doctor                  diagnose: git, uv, docker, browser, LLM servers, ports
hearth eval [--scenario X]     run acceptance evals against configured model → report card
hearth version
```

### 19.3 Building and installing

- **Build.** The wheel is built with hatchling, and a build hook runs `npm ci && npm run build` into `hearth/web_dist`. CI builds wheels on every tag and attaches them to a GitHub Release.
- **`installers/install.sh`** (macOS and Linux):
  1. Install `uv` if it is missing, using Astral's official installer.
  2. Make sure `git` is present. On macOS this means `xcode-select --install` or Homebrew; on Linux, print the right package-manager command.
  3. Resolve the latest release wheel URL through the GitHub API.
  4. Run `uv tool install --python 3.12 <wheel-url>`, then `hearth doctor`.
- **`installers/install.ps1`** (Windows): the same steps. Git is installed with `winget install Git.Git` after the user agrees, or the script points to git-scm.com.
- **Platforms.** Windows 10 22H2+ and 11 (x64); macOS 13+ (arm64 and x64); Linux glibc 2.31+ (x64 and arm64). Python 3.12 is provided by uv.
- **Updating.** `uv tool upgrade hearth` reinstalls from the latest release, using the same wheel URL resolution through `hearth update`.

## 20. Testing

### 20.1 Unit tests (pytest)

- `toolcall_recovery`: a fixture corpus of at least 60 real or realistic outputs across every format in §6.3, including malformed ones
- argument repair and coercion; name fuzzing
- context budgeter and compaction ordering
- approval engine: a table-driven test of at least 150 commands across bash and PowerShell, covering classification, danger detection and fail-closed cases
- path confinement, including symlinks, junctions on Windows, `..`, absolute paths and reserved names
- safe extraction against malicious fixtures: zip slip, symlink, bomb, deep nesting, absolute paths, reserved names
- converters and chunker against sample documents; RRF merge
- skill discovery, precedence, licence classification (using copies of real licence texts), frontmatter handling, substitutions
- capacity-test recommendation logic, using synthetic timings
- GitHub credential injection: the token never appears in argv, the environment of `run`, logs or events

### 20.2 Integration and end-to-end tests with the mock model server

**The mock server** (`tests/mockllm/`) is a FastAPI app that:
- implements Ollama (`/api/version`, `/api/chat`, `/api/show`, `/api/ps`, `/api/embed`), llama.cpp (`/props`, `/v1/chat/completions`) and generic `/v1/*`
- plays scripted response queues for each test, including streaming fragments, leaked tool calls, malformed JSON, `length` finishes and errors

**End-to-end tests** run the real Hearth server against the mock:
- a full coding turn that edits and runs `pytest` on a tiny fixture project
- approval round-trips
- undo
- upload and extraction, followed by search with citations
- helper agents, run sequentially and with N=2
- tool-mode downgrade
- reconnect and replay

**Frontend tests**
- **Component tests** with Vitest.
- **Browser tests** with Playwright drive the built UI against Hearth plus the mock: first-run wizard, chat, approval, Files, Changes and undo.

**CI** runs on GitHub Actions on `ubuntu-latest`, `windows-latest` and `macos-latest`. Docker-tier tests run on Linux only.

### 20.3 Real-model smoke tests

- **What.** A small CPU-friendly model, e.g. `qwen3.5:4b` on Ollama, run through a reduced eval set.
- **Purpose.** This checks the wiring against a real server, not the quality of the results.
- **When.** Run manually, or in a scheduled CI job when a runner with Ollama is available.

### 20.4 Acceptance evals (`evals/`, `hearth eval`)

Each scenario has fixtures, a prompt, and **automatic checks**:

1. **Coding on an uploaded project.**
   - **Fixture.** A small Python package with a pytest suite. It has 2 bugs and one missing function described in the README.
   - **Prompt.** "Make all tests pass."
   - **Pass:** after the run, the harness's own `pytest` exits 0; the transcript shows the agent ran the tests itself; there were no writes outside the project and no edits to test files.
2. **Big document dump Q&A.**
   - **Fixture.** About 40 files (PDF, DOCX, XLSX, PPTX, CSV, MD), some inside a nested zip, with 10 planted facts.
   - **Prompt.** 10 questions.
   - **Pass:** at least 8 of 10 answers contain the expected value and cite the correct file (and the correct page for PDFs); "not in the files" questions (2 of the 10) are answered as not found.
3. **Web research.**
   - **(a) Live web.** A question that needs at least 3 sources. **Pass:** the report cites at least 3 distinct domains, and **every cited URL appears in a `web_fetch` or `browser_*` result in the transcript**, so no citations are invented.
   - **(b) Deterministic browser.** A local fixture site served by the harness, with a JavaScript-rendered page, a multi-step form and pagination. **Pass:** the extracted answer matches the expected value.
4. **Deliverables via skills.**
   - **Tasks:**
     - an XLSX budget with formulas and a chart
     - a DOCX report with headings and a table built from `data.csv`
     - a 5-slide PPTX
     - merging two PDFs and adding page numbers
   - **Pass:** each file opens with openpyxl, python-docx, python-pptx or pypdf and meets structural assertions (formulas present, chart count ≥ 1, heading levels, table row count, slide count, page count, page-number text).

**Bar for v1:** every scenario passes at least 2 of 3 runs on the reference model (§3.3).

**Report card.** `hearth eval` writes `eval-report.html` with pass/fail per check, timings and a link to each transcript. It doubles as a model report card for users.

## 21. Milestones

Each milestone gets its own implementation plan in `docs/superpowers/plans/`. Each ends with its exit tests green in CI and a usable build.

| # | Scope | Exit criteria |
|---|---|---|
| **M1 Core** | Package skeleton and CLI; server, auth and API foundations; SQLite store; connections, detection, adapters (Ollama native and OpenAI-compatible), probe and profile; agent loop with modes A–D, recovery and repair, guards, context management; file tools; `run` on the host runner with approvals, rules, danger list and taint; snapshots and undo; projects (blank, upload without index, open folder); UI shell with chat, tool and approval cards, Files and Changes, Settings → Models, first-run wizard; mock model server and CI matrix | Integration/e2e: coding turn on the fixture project passes against the mock on all 3 OSes; approval and undo e2e pass |
| **M2 Documents** | Safe extraction; converters; index (FTS5, trigram, vectors), RRF search with citations; `read_file` for documents; Index tab; file viewer for documents; `document-qa` and `bulk-summarize` skills (without `task_batch`; sequential map-reduce) | Doc Q&A e2e against the mock; real-model smoke |
| **M3 Web** | Search providers and cache; fetch; Playwright browser thread, snapshot, actions and install flow; Browser tab; `web-research` and `browser-automation` skills | Fixture-site browser e2e; live-search smoke |
| **M4 Skills** | Discovery, licence guard, adapter, selection, `load_skill`/`read_skill_file`; bundled packs vendored with LICENSE and NOTICE; Claude import; install from URL/zip; skill creator; all Hearth-native skills including clean-room document skills, with evals | Licence-guard tests; deliverables eval checks pass against the mock-scripted run; Skills UI e2e |
| **M5 Power** | Process manager, port detection, Preview tab; PTY terminal; container tier (image, runner, network toggle, Autonomous mode); helper agents (`task`, `task_batch`, scheduler, capacity test, helper model); GitHub (device flow, token, repos, clone, `git_push`/`git_pull`, API tools); templates; `codebase-onboarding`, `project-setup`, `github-workflow`, `delegation` skills | Helper e2e (sequential and N=2); Docker-tier tests on Linux; GitHub tools tested against recorded API fixtures |
| **M6 Release** | Installers; `hearth doctor`/`update`/`eval` with HTML report; docs and README; THIRD_PARTY_NOTICES; release workflow; acceptance runs on the reference model | §3 success criteria all true |

## 22. Out of scope for v1

- MCP client and connectors (Slack, Notion, Google and others). Skills referring to them show "not connected".
- Image generation; voice in or out.
- Lightweight OS sandbox tier (T2: `srt`, bubblewrap or Seatbelt).
- Multi-user, LAN access or remote access; accounts.
- Packaging as a desktop app (Tauri or Electron).
- Persistent long-term memory across projects.
- Fine-tuning or model downloading inside Hearth. Users manage models in Ollama or LM Studio; the fit card links to them.

## 23. Risks

| Risk | Mitigation |
|---|---|
| Local model quality disappoints users | Probe and fit card; honest labels; `hearth eval` report card; model recommendations in the docs |
| Host-tier commands plus prompt injection | Smart-mode defaults, taint, danger list, undo, visible tier, Docker one toggle away |
| Scope (the feature set of two large projects) | Six milestones, each shippable; strict out-of-scope list; mock-driven tests keep velocity |
| Cross-platform process handling (Windows jobs, PTY, uvicorn loop) | Thread-based runners; Windows CI from M1; `hearth doctor` |
| Free web search is fragile | Provider chain, caching, pacing; SearXNG and key options; clear error hints |
| Runtime and template churn in Ollama and llama.cpp | Recovery-parser corpus as regression suite; mode ladder auto-downgrade; mock-server tests per backend |
| Licensing mistakes | Manifest-driven vendoring with recorded licences; licence-guard tests; THIRD_PARTY_NOTICES generated from the manifest |
| Context pressure from long skills and tools | Tool profiles, skill outline loading, compaction, BM25 skill preselection |

## 24. Licensing

- **Hearth's own code and Hearth-native skills:** MIT.
- **Bundled packs** keep their upstream licences (MIT or Apache-2.0) with LICENSE and NOTICE files per pack. `THIRD_PARTY_NOTICES.md` is generated from `skills_packs/manifest.yaml` and the Python and JS dependency licences.
- **Proprietary-licensed skills** are never included. Unlicensed skills are only ever read in place from the user's own disk.
- **Branding.** Hearth is not branded as Claude or Anthropic. Attribution to Anthropic appears only in NOTICE and THIRD_PARTY_NOTICES, as Apache-2.0 §4(d) requires.
