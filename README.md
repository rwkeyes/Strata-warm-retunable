<h1 align="center">Strata_Dirigo</h1>

<p align="center"><b>Dirigo Agents' fork of <a href="https://github.com/Niko1221/Strata">Strata</a></b><br>
runtime-retunable · LAN-aware auth · minefield-hardened serving path · runs on older hardware too</p>

> ### ⚠️ The old fork is deprecated — use Strata_Dirigo
> This repository replaces **`Strata-warm-retunable`** and its **`warm-retunable`** branch. That name and branch
> are **deprecated**: they are kept only so old links keep resolving, and they will not receive the fixes or
> features listed below. **Use `Strata_Dirigo`.**
>
> * clone: `git clone https://github.com/rwkeyes/Strata_Dirigo`
> * default branch: **`main`** (based on upstream **v0.1.38**)
> * the old `warm-retunable` branch is frozen at the pre-rename state

Based on upstream **v0.1.38**. Every change is additive or opt-in: **with no flags and no config, this server
behaves exactly as upstream**, and upstream's own test files are byte-identical here — which is the check that
keeps it that way. Details, evidence and caveats for each item are in
[`FORK-FEATURES.md`](FORK-FEATURES.md).

## What this fork adds

| # | Feature | What it gives you | How to use it |
|---|---|---|---|
| 1 | **Strata's own engine re-tunes without a restart** | Change nine engine settings under a live server — no reload, no VRAM movement, no dropped session | `POST /props {"strata_tune": {...}}`, or a per-request `strata_tune` |
| 2 | **The vendored llama.cpp re-tunes too** (`llama-server`) | Re-slice slots, per-slot context and the KV pool, or rebuild the context — **the weights stay resident** | `POST /props` on a `llama-server` built from this tree (`warm-retune/apply.sh`) |
| 3 | **API-key scope + ALLOW list** | Decide who may skip the API key: nobody (upstream default), the local network, this PC only, or the check off — plus named addresses/netblocks | `--api-key-scope` / `--api-key-allow`, the config, or `POST /props` |
| 4 | **No browser tab on start** | A headless/kiosk/remote start stays quiet | `--no-open`, or `STRATA_NO_BROWSER=1` (beats `--open`) |
| 5 | **AVX1 floor** | The engine runs on an **AVX-only** CPU (Sandy Bridge-era Xeons) instead of dying with an illegal instruction | `STRATA_ISA_FLOOR=1 ./setup.sh` |
| 6 | **Calls can be gated per request** | `tool_choice: "none"` **never offers the engine a tool**, so a read-only turn cannot call one — the old behaviour failed *open* | per request |
| 7 | **The request surface is named, and every response says what ran** | Unknown fields are reported (or a 400 with `strict_params`); each response carries a `strata` block with the effective thinking/max_tokens/tools/sampling, and `cap_hit: "reasoning"` when a reply spent its whole budget thinking | `strict_params`, `POST /props` |
| 8 | **History without an empty thinking block** | Stops the empty `<think></think>` shell that nudges thinking collapse and costs the conversation cache | `preserve_empty_think: true` restores the checkpoint's exact rendering |
| 9 | **setup.py carries the llama.cpp patch** and keeps `tools/ui` on Linux | A setup run produces a tree whose `llama-server` already has the retune path | automatic |
| 10 | **A retune audit** (`warm-retune/RETUNE-CANDIDATES.md`) | Which parameters can move while running, which need a rebuild, which are baked into captured graphs — with file/line evidence | read it |
| 11 | **Minefield findings + retest** (`warm-retune/MINEFIELD-FINDINGS.md`) | The serving-path traps an independent registry found on this lane, what was fixed, and the before/after that closed them | read it |

### Runtime retune, in practice

```sh
# Strata's own engine (items 1): the nine keys are prefill, short_read, prompt_cache, prompt_cache_every,
# adapt_every, suffix_draft, mtp_max_t, pcie_frac, spec_min_p
curl -s -X POST localhost:8080/props -H 'Content-Type: application/json' \
     -d '{"strata_tune": {"prefill": 4096, "adapt_every": 100000}}'
curl -s localhost:8080/props | jq '{strata_tune, strict_params, api_key_scope}'

# A call that must not touch a tool (item 6)
curl -s localhost:8080/v1/chat/completions -H 'Content-Type: application/json' \
     -d '{"model":"…","messages":[{"role":"user","content":"just tell me the time"}],
          "tools":[…],"tool_choice":"none"}'

# A keyed server that trusts the LAN, without changing the default for anyone else (item 3)
./serve/server.py --api-key "$SECRET" --api-key-scope lan --api-key-allow 10.1.0.0/16
```

### Controlling the slots and the KV pool while it runs (item 2)

With `--kv-unified` every slot shares **one** KV budget, and each slot's context is a *reservation* against it:
`-np 4 -c 131072` is four 32,768-token shares — so three idle slots are holding 96k tokens of KV that a single
long-context request could be using. The usual fix is to restart the server with different flags, which means
reading the whole model back in (a minute or more) and dropping every open conversation. This patch makes the
shape a knob instead:

```sh
llama-server -m model.gguf -ngl 99 -fa on --kv-unified --props -np 4 -c 131072 -a my-model
curl -s localhost:8080/props | jq '{total_slots, slots_max, kv_pool_n_ctx, kv_unified}'  # the shape now

curl -s -X POST localhost:8080/props -d '{"parallel": 4, "ctx_per_slot": 32768}'   # four short sessions
curl -s -X POST localhost:8080/props -d '{"parallel": 1, "ctx_per_slot": 131072}'  # one long-context request
curl -s -X POST localhost:8080/props -d '{"ctx": 262144}'                          # grow the pool itself
curl -s -X POST localhost:8080/props -d '{"cache_type_k": "q8_0"}'                 # or the KV's own shape
```

**Why you would want to:** the demand oscillates — a coding agent fanning out four subagents wants four slots, and
the 100k-token repository you paste a minute later wants one big one — and neither shape is "the right config".
Moving between them costs **0–0.4 ms** for a re-slice and **~0.8 s** to rebuild the pool (a 32k → 262k pool,
measured), because the weight buffers are never touched; a restart costs a full model reload. A request that would
pull a **busy** slot out from under a live conversation is refused with
`HTTP 400 slot N is busy, cannot take it out of service` — never a hang, never a silent drop. Measured costs, the
VRAM-per-pool table, and the `parallel_max` / oversubscribe paths:
[`warm-retune/SLOTS-AND-KV-POOL.md`](warm-retune/SLOTS-AND-KV-POOL.md).

### Hardware reach

* **AMD**: HIP backend, measured on an **RX 7900 XTX 24 GB** (the reference box for the numbers in
  `warm-retune/`).
* **NVIDIA**: CUDA 12/13 builds, as upstream.
* **CPU-only**: the AVX1 floor (item 5) — measured on a Xeon E5-2687W.
* **Older AMD**: the `--amdgpu_targets` build path keeps gfx1010-era cards working.

## Coming very soon: Intel Arc

**Intel Arc support is coming very soon.** It is the next backend target on the roadmap here: the engine's
backend selection and the setup path are being extended so an Arc card is a supported option rather than a manual
build. Watch this repository's commits — it will be documented in `docs/DETAILS.md` and announced in the README
when it lands.

## What is verified

| | |
|---|---|
| Test suite | **271 tests** green (`python -m unittest discover -s serve`); `test_detok`'s 3 errors are pre-existing on upstream v0.1.38 in a minimal environment (a missing `regex` module) |
| Upstream's own tests | unmodified in `test_lifecycle.py`, `test_mcp.py`, `test_monitor.py`; `test_server.py` differs only by an unused import and the fork's own test class |
| Independent serving-path registry | [`model-serving-minefield`](https://github.com/Blackwellboy/model-serving-minefield) doctor run against this server: the traps it found (unvalidated request surface, `tool_choice` ignored, empty think shells) now come back **clean**; the two that remain are properties of the lane and the checkpoint, named in `MINEFIELD-FINDINGS.md` |
| Storage A/B | HDD vs NVMe on the same checkpoint and config: cold start 311 s → 64 s, decode 2.8× (`warm-retune/NVME-VS-HDD.md`) |

## Docs in this repository

| file | what |
|---|---|
| [`FORK-FEATURES.md`](FORK-FEATURES.md) | every feature above, one page each: what it does, its limits, how far it is verified |
| [`warm-retune/SLOTS-AND-KV-POOL.md`](warm-retune/SLOTS-AND-KV-POOL.md) | controlling slots and the KV pool on a running `llama-server`: the example, the why, the measured costs |
| [`warm-retune/RETUNE-CANDIDATES.md`](warm-retune/RETUNE-CANDIDATES.md) | the audit: what can be retuned while running, and what cannot |
| [`warm-retune/MINEFIELD-FINDINGS.md`](warm-retune/MINEFIELD-FINDINGS.md) | the serving-path traps found on this lane, the fixes, the retest |
| [`warm-retune/NVME-VS-HDD.md`](warm-retune/NVME-VS-HDD.md) | the storage A/B measurements |
| [`WARM-RETUNABLE.md`](WARM-RETUNABLE.md) | the fork's original notes and per-feature measurements |

## Licence

Upstream Strata is MIT (see [`LICENSE`](LICENSE)); the vendored llama.cpp is MIT, and the `llama-server` retune
patch is a derivative work under the same terms.

---

<sub>Everything below is upstream Strata's own README, kept as it ships.</sub>

<h1 align="center">Strata</h1>

<p align="center"><b>Run a 125-billion-parameter AI model on your own gaming PC</b><br>
NVIDIA or AMD graphics card (12 GB or more) · Windows or Linux · free and open source</p>

<p align="center"><a href="https://github.com/Niko1221/Strata/releases/download/v0.1.10/Pagoda.mp4"><img src="docs/media/pagoda-preview.webp" width="720" alt="A voxel pagoda garden that Strata's model wrote, running in the browser"></a><br>
<sub>A voxel pagoda garden, 1 shot prompt running on an RTX 5070 with Strata (IQ3_S, 128K context) ·
<a href="https://github.com/Niko1221/Strata/releases/download/v0.1.10/Pagoda.mp4">full video (49 s)</a></sub></p>

Strata runs **[Qwen3.8-Flash-Next](https://huggingface.co/Qwen/Qwen3.8-Flash-Next)** - a large, smart AI model that
normally needs a server - on a normal PC. It chats, writes code, reads pictures and works with your apps and coding
agents, and nothing leaves your PC.

## How fast is it?

Measured on two ordinary gaming PCs. "Writes answers" is how fast the reply appears in a short chat; "reads your
prompt" is how fast it takes in what you send (a 32K-token document, code or chat history). A token is about ¾ of a
word, so 60 tokens per second is faster than you can read.

<table>
<tr><th>NVIDIA: RTX 5070 (12 GB), Ryzen 5 7600, 64 GB RAM</th><th>AMD: RX 9070 XT (16 GB), Ryzen 9 3900X, 47 GB RAM</th></tr>
<tr><td>

| Size | Writes answers | Reads your prompt |
| --- | ---: | ---: |
| **Q2_0** | 94 tokens/s | 2,650 tokens/s |
| **IQ2_XS** | 79 tokens/s | 2,090 tokens/s |
| **IQ3_XXS** | 62 tokens/s | 1,750 tokens/s |
| **IQ3_S** | 53 tokens/s | 1,620 tokens/s |
| **Coder** | 55 tokens/s | 2,180 tokens/s |

</td><td>

| Size | Writes answers | Reads your prompt |
| --- | ---: | ---: |
| **Q2_0** | 60 tokens/s | 1,160 tokens/s |
| **IQ2_XS** | 52 tokens/s | 1,110 tokens/s |
| **Coder** | 44 tokens/s | 1,420 tokens/s |

</td></tr>
</table>

A card with more VRAM is faster: an RTX 3090 (24 GB) should write roughly 100-140 tokens per second. Long chats,
other cards: [speed of each model](docs/MODELS.md#how-fast-is-each-size), [community results](docs/COMMUNITY_BENCHMARKS.md).

<p align="center"><a href="https://buymeacoffee.com/strataengine"><img src="https://cdn.buymeacoffee.com/buttons/v2/default-yellow.png" alt="Buy Me A Coffee" height="50"></a><br>
<sub>Strata is free. If it runs well on your PC, a coffee keeps the work on it going.</sub></p>

## What you need

| | |
| --- | --- |
| **Graphics card** | **NVIDIA** GeForce RTX 20, 30, 40 or 50 series, or **AMD** Radeon RX 7900 XT / XTX, RX 7800 XT / 7700 XT, RX 9060 XT, RX 9070 / 9070 XT, Radeon AI PRO R9700 or RX 6800 / 6900 series - with **12 GB of VRAM or more** |
| **RAM** | 32 GB or more - how much decides [which model](#which-model-should-i-pick) fits; 64 GB runs every size |
| **Disk** | about 80 GB free, on an SSD if you can (the first start is much faster) |
| **System** | Windows 10 / 11 or Linux, and a current graphics driver from NVIDIA or AMD |

Everything else is installed for you. Two or three cards can share the model ([multi-GPU](docs/MULTI_GPU.md)).
The full list: [docs/INSTALL.md](docs/INSTALL.md#what-you-need).

## Install

### Let your AI set it up

Use an AI coding assistant (Claude Code, Cursor, Codex, GitHub Copilot, ...)? Paste this into it:

```text
Set up Strata on this PC for me: https://github.com/Niko1221/Strata - follow docs/AI_SETUP.md in that repository.
```

It checks your graphics card, RAM and disk, picks the model that fits, installs it, starts it and tells you how to
connect your apps. AI tools can also install, start and stop Strata themselves through its
[MCP server](docs/MCP_SERVER.md).

### Or do it yourself

[Download Strata](https://github.com/Niko1221/Strata/archive/refs/heads/main.zip) and unzip it (or `git clone` it).
**Windows:** double-click **`START-HERE.bat`**. **Linux:** run **`./setup.sh`** in the Strata folder.

The same steps for NVIDIA and AMD: the installer finds your card and sets up the right engine for it. It asks which
model, which size, how much context (how much text it keeps in mind) and whether it should read pictures - press
Enter each time for the recommended answer. Then it downloads the model (~70 GB; you can stop and it continues where
it left off) and starts it. Your browser opens the Strata app at `http://127.0.0.1:8080` (a headless or kiosk start
can pass `--no-open` to the server, or set `STRATA_NO_BROWSER=1`).

> **While the model starts, your PC can be slow or stop responding for 1-3 minutes** (longest the first time): Strata
> loads 35-55 GB into your RAM and locks part of it for the graphics card. That's normal - wait, and don't close the
> window. The window tells you what it is doing.

**Next time**, run `START-HERE.bat` (or `./setup.sh`) again: it starts right away, nothing is downloaded twice. Close
its window to stop the model. `UPDATE.bat` (`./update.sh`) updates Strata without starting it. Updating, Docker,
several cards, where the files go and every option:
[docs/INSTALL.md](docs/INSTALL.md).

## Which model should I pick?

The installer recommends one for your RAM. The same model comes in sizes that are compressed more or less: smaller
is faster, larger is a bit smarter.

| Your RAM | Take | Why |
| --- | --- | --- |
| **32 GB** | **Coder** | it fits 32 GB, and it is made for code (with a 24 GB card, Q2_0 and IQ2_XS run too) |
| **48 GB** | **IQ2_XS** (or Q2_0, the fastest) | the larger sizes do not fit |
| **64 GB** | **IQ2_XS** (recommended), or IQ3_XXS / IQ3_S | every size fits; IQ3_S is the best, and the slowest |
| **96 GB or more** | **IQ3_S**, or Unsloth's 4-bit (experimental) | room for the largest sizes with everything else open |

- **[Coder](docs/MODELS.md#coder)** - a coding version with half of the experts removed: 91% of the full model's
  SWE-bench Verified score (by its authors), fits 32 GB of RAM. Weaker outside code, including Chinese and other
  CJK text (#438): for those, take Q2_0, IQ2_XS or IQ3_S, which keep every expert.
- **[Swift 1.5](docs/MODELS.md#swift-15)** - a fine-tune that thinks much shorter before it answers, so you get the
  answer sooner, at about the same quality.
- **[Unsloth UD-Q4_K_XL](docs/MODELS.md#unsloth-ud-q4_k_xl-experimental)** (experimental) - the closest to the full
  model, but most of it is read from the SSD while it answers: 7-8.5 tokens/s on a 64 GB PC.
- **[OrcaRouter's Uncensored IQ3_XXS](docs/MODELS.md#orcarouter-uncensored-iq3_xxs)** - a manual setup, not in the
  installer's menu.

Sizes, downloads and what fits where: [docs/MODELS.md](docs/MODELS.md). You can add another model later with
`SETUP.bat` (Linux: `./setup.sh --setup`).

## Using it

<p align="center"><img src="docs/media/runpagoda.png" width="900" alt="The Strata app's Monitor tab next to a coding agent"><br>
<sub>The Strata app's <b>Monitor</b> (left) while a coding agent writes the pagoda garden from the video (right)</sub></p>

- **In the browser:** `http://127.0.0.1:8080` - **Chat**, a live **Monitor** of the model and your GPU/CPU/RAM, and
  **About** with the settings and addresses.
- **Your apps and coding agents:** add an "OpenAI-compatible" provider with base URL **`http://127.0.0.1:8080/v1`**,
  any API key and any model name. Apps that use Anthropic's API: `http://127.0.0.1:8080/v1/messages` (Claude Code:
  `ANTHROPIC_BASE_URL=http://127.0.0.1:8080`).
- **Thinking:** choose **off, low, medium or high** in the chat menu or your app's "reasoning effort". Off is
  fastest; high is best for hard questions.
- **Pictures:** say yes to "Images?" in setup, then click **Picture** in the chat, or attach them in your app
  (AMD cards: on Linux through the processor, not on Windows yet).
- **From your phone or another PC:** `START-HERE.bat --setup --host 0.0.0.0 --api-key <secret>` - always with a key.
- **Good to know:** it answers one request at a time. The first message of a chat is read in full (about 1 minute
  per 30,000 tokens); follow-ups start in seconds.

More: [where your chats are stored](docs/INSTALL.md#where-things-are-stored), [the API](docs/DETAILS.md#using-it).

## Something went wrong?

- **My PC froze the first time Strata started.** Normal while it loads the model: wait, don't close the window.
  Still frozen after 10 minutes? Restart the PC, close other programs and try again, or pick a smaller size.
- **It stopped while downloading or installing.** Run `START-HERE.bat` (or `./setup.sh`) again: it continues where
  it stopped.
- **It's very slow and the disk light keeps blinking, or "the engine stopped unexpectedly".** Not enough free RAM:
  close other programs (browsers use a lot), or pick a smaller size (Q2_0 or IQ2_XS).
- **It says port 8080 is already in use.** Strata is already running - look for its window.

More problems and their fixes: [docs/TROUBLESHOOTING.md](docs/TROUBLESHOOTING.md). Still stuck? Open an
[issue](https://github.com/Niko1221/Strata/issues) and attach `strata-<model>.log` from the Strata folder.

## How does it work?

Models like this one normally run on servers with hundreds of gigabytes of graphics memory. Your graphics card has
12-24 GB. Strata makes it fit by **sharing the work across your whole PC** - like a kitchen, where the things you use
all the time stay on the counter and the rest waits in the pantry.

<p align="center"><img src="docs/media/how-it-works.svg" width="860" alt="The model's 24,576 experts: the busiest on the graphics card, all of them in RAM, a lookup table on the SSD"></p>

- **The model is a team of 24,576 small specialists ("experts"),** and each word needs only 10 of them.
- **Your graphics card** keeps the few thousand experts that are asked most often; **your RAM** holds all of them,
  and **your processor** works on the rest at the same time. **Your SSD** holds a big lookup table.

<p align="center"><img src="docs/media/guess-and-check.svg" width="860" alt="A small helper guesses the next words; the big model checks them all at once and keeps the right ones"></p>

- **Guess, then check:** a small helper guesses the next few words and the big model checks them all at once, so
  you get the same answer, 1.6-1.8x sooner.
- **Long texts are read in big pieces** (up to 8,192 tokens at a time): over 1,000 tokens per second.

The longer explanation: [docs/HOW_IT_WORKS.md](docs/HOW_IT_WORKS.md). Every part and its numbers: [the
details](docs/DETAILS.md#how-it-works) and the [paper](docs/paper/Strata-Paper.pdf).

## Credits and license

The model is [Qwen3.8-Flash-Next](https://huggingface.co/Qwen/Qwen3.8-Flash-Next) by the Qwen team, compressed by
[ISTA-DASLab](https://huggingface.co/ISTA-DASLab/Qwen3.8-Flash-Next-GSQ-RCO-GGUF), UkisAI (Swift 1.5) and Unsloth;
Strata is built with parts of [llama.cpp / ggml](https://github.com/ggml-org/llama.cpp). All credits:
[docs/HOW_IT_WORKS.md](docs/HOW_IT_WORKS.md#credits). Strata is open source under the [MIT License](LICENSE); a few
parts and every model carry their own licenses ([which ones](docs/HOW_IT_WORKS.md#license)).

## Support Strata

Strata is free and open source. If it is useful to you, you can support its development:

<p align="center"><a href="https://buymeacoffee.com/strataengine"><img src="https://cdn.buymeacoffee.com/buttons/v2/default-yellow.png" alt="Buy Me A Coffee" height="50"></a></p>
