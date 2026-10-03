# The minefield doctor on this fork's lane — findings, fixes, retest

[`Blackwellboy/model-serving-minefield`](https://github.com/Blackwellboy/model-serving-minefield) is a registry of
serving-path traps: request-shaped defects that are **not** the weights. Run against Strata's OpenAI surface it
audits the layer between the client and the model — template rendering, the request field set, tool gating,
thinking/routing and the budget arithmetic — which is exactly the layer this fork changes.

Run (cloned fresh, from the source, not a local adaptation):

```sh
git clone --depth 1 https://github.com/Blackwellboy/model-serving-minefield.git
python3 -m minefield quick --base-url http://127.0.0.1:18110/v1 --model <model-id> \
        --api-key "$STRATA_API_KEY" --hf-repo ISTA-DASLab/Qwen3.8-Flash-Next-GSQ-RCO-GGUF \
        --hf-revision ed59f92082b1e93c0e96d60a8b11aab089b52f09 --json doctor.json
```

13 requests, 20 checks, the pinned checkpoint passed so its configuration checks fire.

## What the doctor found

| trap | what it means on this lane | verdict |
|---|---|---|
| **78** | `tool_choice: "none"` was accepted and **ignored**: a control with tools and no `tool_choice` called a tool, and the identical request with `"none"` called one too. **Fails open** — an agent loop with a side-effecting tool can act on a turn the caller believed read-only. | **problem, fixed** |
| **77** | The request surface was unvalidated: an invented top-level field was accepted with `200`, so a misspelling was silent and a status code confirmed nothing. | **problem, fixed** |
| **12** | A hard task at `max_tokens=512`: `finish=length`, **empty content**, 1966 chars of reasoning. Honest truncation — but a harness that scores that zero is measuring its own budget, not the model. | **problem, fixed** |
| **21** | The GGUF repo ships **no `generation_config.json`**: there is no such thing as "model defaults" here, sampling must be set explicitly per request/mode. | **problem, fixed** |
| **04/25** | History rendered **2 empty think blocks** for prior turns: a prior assistant turn whose reasoning was not resent got `<think>\n\n</think>`. Nudges thinking collapse on later turns, and makes equivalent histories render differently (a conversation-cache miss). | **problem, fixed** |
| **01, 02, 03, 20, 23, 29, 19, 26, mm** | Reasoning arrives under `reasoning_content` (and `reasoning` is a dead write name); no orphaned `</think>`; streamed deltas and real structured `tool_calls` on the forced probe; a text-only lane rejects an image part with a 400 naming the modality. | **clean** |
| **07** | `reasoning_effort` deadness — one render cannot show it; needs a two-render diff (`checks/preflight_template.py`). | inconclusive |
| **10, 17, 141, 22** | Coverage/scoping: config.json 404 (a GGUF repo), no shipped generation config, an SGLang-only entry, and "one budget cannot characterise the floor". | inconclusive |

The clean rows matter as much as the problems: they say the answers themselves are sound (no dequant corruption,
no orphaned tags, real tool calls), so every finding above is a serving-path defect and **none of them needs a
weight change, a re-quant, or a different checkpoint**.

## What changed in the fork

All server-side or template-side; the engine and the weights are untouched.

| trap | fix | where | knob |
|---|---|---|---|
| 78 | `tool_choice` implemented: `"none"` **omits the tools payload** from what the engine is offered (a lane never offered a tool cannot call one, whatever the template does), a named function narrows the offer to that one, `"required"` is offered and **reported unenforced**, and any value that cannot be honoured is a **400**. Anthropic's `{"type": "none"│"any"│"tool"}` too. | `serve/frontend.py` (`tool_choice_of`, `offer_tools`), `serve/server.py` (both API paths), and MCP tools are not even collected under `"none"` | per request |
| 77 | Unknown fields are named **once each** in the log (the API and the field), and **every response carries a `strata` block** with the effective settings, so a client asserts on the response per request rather than on the status code. `"strict_params": true` turns an unknown field into a **400** listing it. | `serve/frontend.py` (`unknown_params`, the field sets), `serve/server.py` (`check_params`, `effective_settings`, `openai_collect`) | `strict_params` (config or `POST /props`) |
| 12 | A reply whose whole budget went on thinking is **named**: `strata.cap_hit = "reasoning"` when `finish=length` with empty content and non-empty reasoning. The budget itself (`reasoning_budget_tokens`) already existed — see below for what the retest sets. | `serve/server.py` (`openai_collect`) | request or config |
| 21 | Sampling is reported as the engine was actually given it (`strata.sampling`, the engine's own spelling), so "unset" is visible instead of implied; the model configs carry the card's sampling per mode. | `serve/server.py` (`effective_settings`) | config `sampling` |
| 04/25 | The template writes the `<think>` wrapper **only when there is reasoning to preserve**. `preserve_empty_think: true` — config or a request's `chat_template_kwargs` — restores the checkpoint template's rendering exactly. | `serve/chat_template.jinja` | `preserve_empty_think` (config, request, or `POST /props`) |

`strict_params` and `preserve_empty_think` are retunable while the server runs (`POST /props`), like the API-key
policy, so a deployment can be tightened or relaxed without a restart.

### The golden cases the template change moves

3 of the pack's 10 golden cases contain a prior turn with no reasoning, so the fork's rendering differs from the
checkpoint template's there: `multi-turn`, `no generation prompt`, `tool call and response`. `serve/chat_golden.json`
records the new rendering with a `note` on each, and `preserve_empty_think=true` reproduces the pack's rendering
**byte-for-byte on all 10** — which is what makes the change a deliberate fork decision rather than a template
regression. It is the same trade the API-key scope makes: the safer default now, upstream's behaviour one setting
away.

## Tests

`serve/test_minefield.py` — one class per trap, named after it, so a regression reads as the trap reopening:
`tool_choice` (11 tests incl. that `"none"` keeps the tool out of the prompt the engine was handed, that a bad
value is a 400, and that a name the request does not offer is refused), the request surface (unit + strict/lenient
over HTTP + the retunable knobs), the effective-settings echo (non-stream, stream's first chunk, Anthropic), the
cap-hit, and the template (the pack's rendering with the knob on, this server's without it, real reasoning still
preserved, and all 10 goldens matching this tree).

    python -m unittest serve.test_minefield -v      # 39 tests
    python -m unittest discover -s serve -p 'test_*.py'

## Retest

Protocol: the **same checkpoint on the same storage** as the pre-fix run (the NVMe-staged IQ1_M Coder, so the only
variable is the code), the fork's `serve/` against the same engine binary, then the doctor + `checks/dequant_fidelity.py`
+ this fork's 5-probe battery. A trap counts as closed only when the doctor's own probe no longer reports it — the
harness is the upstream one, unmodified.

**Results: pending** (see the run log `logs/minefield-retest.*`). The trap-by-trap before/after table is appended
here when it lands.
