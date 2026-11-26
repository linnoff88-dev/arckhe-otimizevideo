# otv — otimizevideo

**🇧🇷 [Português](README.md) · 🇺🇸 [English](README.en.md) · 🇪🇸 [Español](README.es.md)**

CLI that takes a long video (class, podcast, lecture — 20 to 30 min) and returns a cut of
**~2 minutes** with the best content, cut precisely at word boundaries.

## 📖 User guide

Complete guide (landing page + step-by-step): **https://inematds.github.io/otimizevideo/guia/en/**

## 1. What it does

`otv` runs a video through a pipeline of stages (transcription → scene detection →
visual classification → scoring → selection → render) and produces a condensed `output.mp4`.
The project's golden rule, and what sets it apart from "send the video to the model and ask it
to make a cut": **the model never chooses timestamps**. It only sees the transcript split into
units numbered by ID (`[042] 4.2s talking_head "unit text"`) and returns a score from 0 to 10
for each ID. The code — not the model — converts ID → time (`ini`/`fim` in seconds, taken
directly from the word-timestamped transcript) and makes the cut.

This matters for two reasons: (1) the cut never falls in the middle of a word, because each
unit's boundary is already a real word/pause boundary from the transcript, never a timestamp
invented by the model; and (2) the result is reproducible — running the same selection
twice with the same `notas.json` produces the same `plan.json`, because the selection logic
(knapsack by score, topic quota, hook/ending, cohesion) is deterministic and lives in the code,
not in an LLM hallucination.

## 2. Installation

```bash
pip install -r requirements.txt
```

The system Python on this machine uses PEP 668 (an "externally managed" environment) — if the
command above fails with `error: externally-managed-environment`, use:

```bash
pip install --user --break-system-packages -r requirements.txt
```

These must also be on the `PATH`:

- **ffmpeg** and **ffprobe** (cutting, audio extraction, thumbnails, final mux)
- **yt-dlp** (downloading video from a URL — only needed for `otv run <url>`, not for a local file)

`whisper_local`, `whisperx`, and `ollama` (optional providers; see the table in section 5) come
with large dependencies (`openai-whisper`, `whisperx`, `torch`) that are **not** in
`requirements.txt` — install them only if you plan to use those providers.

### Keys

`otv` never writes an API key to `config.yaml` or any other file in this project —
keys are read at **runtime** (`otv/util/keys.py`), from the first source that has the variable:

1. **Environment variable** — `GROQ_API_KEY=... python3 otv.py run ...`, systemd, Docker
2. The file pointed to by **`OTV_ENV_FILE`**, if defined
3. **`.env` in the project root** (listed in `.gitignore`)
4. **`~/.config/otv/.env`**
5. The source machine's `.env` files — `~/projetos/openpcbotv2/.env`, `~/projetos/wifi/.env`

Options 1–4 make the project portable to another machine or VPS without editing code
(section 15). Lines with `#` are ignored, and `export NAME=value` is accepted.

Set `GROQ_API_KEY`, `OPENROUTER_API_KEY` and, if using ElevenLabs,
`ELEVENLABS_API_KEY`/`ELEVENLABS_VOICE_ID` (plus `FAL_KEY` for A+ mode). This project never
prints, logs, or writes any keys to disk.

## 3. Quick start

```bash
python3 otv.py run <url-ou-caminho-do-video> --modo A --alvo 120
```

This runs the entire pipeline (ingest → transcribe → scenes → score → select → render)
and writes the result to `~/projetos/output/otimizevideo/<id>/output.mp4` (`<id>` is the
YouTube video ID, or the normalized filename when the source is local). The same
`output.mp4`, along with `plan.json`, `notas.json`, `unidades.json`, and `custos.json`, is also
saved in `trabalho/<id>/` (the intermediate working directory).

## 4. The three modes

| Mode | Output | When to use it | Requirements |
|---|---|---|---|
| **A** | Condensed video with the presenter's **original speech** (including talking head) | Default — preserves the tone and voice of the person who recorded it | Nothing beyond transcription; runs 100% with `visual: local` |
| **B** | **Without** the presenter — cuts to the screen/chart/slide and **re-records the narration** (TTS) over it | When the face/eyes in the original video don't matter and you want a more "editorial" cut | Model-based visual classification (scenes must be labeled `demo_tela`, `slide`, or `grafico` — local detection only distinguishes "face present" from "no face") |
| **C** | Only **demos and charts** — the most restrictive mode | The video is mostly demos/charts and you want only those, with no talking head | Same requirements as mode B |

There are also two derived modes, documented at the end of this README: **A+** (keeps the
original speech but replaces the presenter's video with generated illustrations — section 13)
and **N** (keeps the entire video on screen and re-records only the audio as narration —
section 14).

**B and C depend on model-based visual classification.** Local detection (`visual: local`) only
knows how to identify "large face present" (`talking_head`) or "no" (`outro`) — it never produces
`demo_tela`, `slide`, or `grafico`. Without passing `--visual glm`, `--visual gemini`, or
`--visual claude_cli` (or manually running `otv cenas <id> --classificar`), `otv run
--modo B` (or `C`) continues with a warning and then fails during selection with a clear message
saying there are no units with the required visual type.

## 5. `config.yaml` slots

The root `config.yaml` defines which provider to use by default for each slot. Any slot can be
overridden on the command line (`--transcricao`, `--visual`, `--pontuacao`, `--tts` in
`otv run`; `--provedor` in `otv transcrever`/`cenas`/`pontuar`/`narrar`), and the entire file
can be replaced with `--config outro.yaml`.

| Slot | Providers | Cost/requirements | Default |
|---|---|---|---|
| `transcricao` | `groq` | Cloud, requires `GROQ_API_KEY`. Doesn't report cost in `$`; ~US$0.04/hour of audio | **default** |
| | `whisper_local` | Local GPU/CPU, free, but requires `openai-whisper` installed (not included in `requirements.txt`) | |
| | `whisperx` | Local GPU (CUDA if available), free, requires `whisperx`+`torch` installed | |
| `visual` | `local` | Free, no key — face-area heuristic (mediapipe). Only distinguishes `talking_head`/`outro`, never `demo_tela`/`slide`/`grafico` | **default** |
| | `glm` (`z-ai/glm-5.3-flash` via OpenRouter) | Requires `OPENROUTER_API_KEY`. Inexpensive (~US$0.004 per 25-minute video) | |
| | `gemini` (`google/gemini-2.5-flash-lite` via OpenRouter) | Requires `OPENROUTER_API_KEY` | |
| | `claude_cli` (Claude Code headless, `claude -p --model sonnet`) | Uses the user's **subscription** — no API key, but requires the `claude` binary on the `PATH` and a logged-in session | |
| `pontuacao` | `glm` (`z-ai/glm-5.3-flash` via OpenRouter) | Requires `OPENROUTER_API_KEY`. ~US$0.002 per 25-minute video (1 call, `reasoning_effort: low`) | **default** |
| | `gemini` (`google/gemini-2.5-flash-lite` via OpenRouter) | Requires `OPENROUTER_API_KEY` | |
| | `ollama` (`qwen3.8:27b`) | Local, free, but requires the **Ollama daemon** running (`http://localhost:11434`) | |
| | `claude_cli` (Claude Code headless) | Uses the subscription — no API key, requires `claude` on the `PATH` | |
| `tts` | `inemavox` | Local, free, but requires the **inemavox daemon** running (`http://localhost:8010`). Voice and engine come from `config.yaml → voz:` / `tts_engine:` — the default is **`edge` + `pt-BR-FranciscaNeural`** (see section 15) | **default** |
| | `elevenlabs` | Cloud, requires `ELEVENLABS_API_KEY` and `ELEVENLABS_VOICE_ID`; billed per character | |

The specific models for each LLM slot (`glm`, `gemini`, `ollama`, `claude_cli`, and
`turbo` for `whisper_local`) are set in `config.yaml → modelos:` — changing the model doesn't change
the slot, only what that slot calls.

## 6. `selecao` parameters (`config.yaml → selecao:`)

| Parameter | Default | Practical effect |
|---|---|---|
| `alvo_s` | 120 | Target duration, in seconds, for the final video (same value overridden by `--alvo`) |
| `tolerancia` | 0.25 | Acceptable range around the target: ceiling = `alvo_s × 1.25`, floor = `alvo_s × 0.75`. After the main knapsack has filled the pure `alvo_s` target, selection uses this ceiling as extra room for cohesion/hook/ending |
| `min_segmento_s` | 3 | Any final segment shorter than this (after cutting/snapping) is discarded — avoids fraction-of-a-second cuts |
| `min_segmento_ideal_s` | 8 | "Cohesion" target: each segment tries to pull in nearby units with reasonable scores until it reaches this length, to avoid becoming a sequence of tiny snippets |
| `cota_topico_pct` | 40 | Ceiling, as a % of `alvo_s`, for how much a single topic can occupy in the main knapsack — keeps the cut from becoming "just the most-discussed part" and ensures topic diversity |
| `nota_minima` | 5 | Minimum score (0–10) for a unit to enter the main knapsack. Hooks/endings and cohesion extensions tolerate slightly less (`nota_minima` − 2 or − 3) to complete the segment |
| `folga_ms` | 120 | Extra margin, in milliseconds, left after the end of a segment's last unit when snapping — avoids cutting right on the last syllable |
| `pausa_fronteira_ms` | 400 | Minimum pause between words (in the `unidades` stage) that counts as a natural unit boundary — along with punctuation (`.`/`!`/`?`) and the end of a transcript segment, it's one of the criteria for closing a unit |
| `max_unidade_s` | 12 | Maximum duration for a cuttable unit — even without pauses or punctuation (continuous speech), the code forces a cut at the largest available internal pause before exceeding this limit |

## 7. All subcommands

```bash
# complete pipeline
python3 otv.py run <url-ou-caminho> --modo A --alvo 120

# only download/copy the video and extract audio, without processing anything
python3 otv.py ingest <url-ou-caminho>

# individual stages, for an already ingested folder (<id> is the folder name in trabalho/)
python3 otv.py transcrever <id> --provedor groq
python3 otv.py cenas <id> --classificar --provedor glm
python3 otv.py pontuar <id> --modo A --alvo 120 --provedor glm
python3 otv.py selecionar <id> --modo A --alvo 120
python3 otv.py substituir <id> --provedor fal     # mode A+: replace the presenter with illustrations
python3 otv.py render <id> --rapido
python3 otv.py narrar <id> --provedor inemavox

# artifact inventory + cut plan summary
python3 otv.py status <id>

# cumulative spend by stage, read from trabalho/<id>/custos.json
python3 otv.py custo <id>
```

## 8. Run stages individually and reuse work

Each stage writes its own JSON to `trabalho/<id>/` (`transcript.json`, `scenes.json`,
`unidades.json`, `notas.json`, `plan.json`, `custos.json`...) and most are **idempotent**:
if the artifact already exists, the stage doesn't run again — it just returns what's already
on disk. `--forcar` forces it to run again.

**Three exceptions always run again, even without `--forcar`:** `selecionar` (recreates the plan
every time — the right behavior for running `otv selecionar <id> --modo B --alvo 90`
repeatedly until you find a good combination, without editing code), `render` (has no notion of
"already exists" — it always renders again from the current `plan.json`; that's how section 9
works), and `narrar` (likewise — it always generates the WAVs again from the current
`plan.json`). The stages that actually skip when their artifact exists are `ingest`,
`transcrever`, `cenas`, and `pontuar`.

This saves real money: if scoring (`otv pontuar`) has already run and cost one LLM call, you
can run `otv selecionar` several times with different `--modo`/`--alvo` values, or run
`otv render` again after manually editing the plan (section 9), without paying for another
model call — rerunning `selecionar`/`render`/`narrar` costs nothing because they don't call
any LLM. Only rerun `ingest`/`transcrever`/`cenas`/`pontuar` with `--forcar` when you truly
need a new result from that stage (e.g., incorrect transcription, outdated visual classification
after changing providers).

## 9. Manually edit `plan.json` and render again

`plan.json` is text — you can edit the final cut without calling any model. This feature gives
the user editorial control over the automatic selection.

A segment has this format:

```json
{
  "in": 175.1,
  "out": 182.5,
  "unidades": [58, 59, 60],
  "visual": "talking_head",
  "motivo": "explains the experiment result",
  "texto": "The judges said that the 50-year-old problem was essentially solved.",
  "estender_s": 0
}
```

You can remove an entire segment from the `segmentos` list, adjust `in`/`out` manually
(note: `unidades` will no longer match the new interval exactly, but render only uses
`in`/`out`), or edit the `manchete` text at the top of the JSON. After editing, **don't** run
`otv selecionar` again (it rewrites `plan.json` from scratch every time — see section 8) — go
straight to render:

```bash
python3 otv.py render <id>
```

**Be careful with `otv run` on the same folder.** `otv run <source>` without `--forcar` skips
`ingest`/`transcrever`/`cenas`/`pontuar` if the artifacts already exist, but it always calls
`selecionar` internally — which, as section 8 explains, rewrites `plan.json` even without
`--forcar`. Running `otv run` again in the same folder after manually editing `plan.json`
**silently deletes your edits**. After editing the plan, the correct command is to run
`otv render <id>` on its own (above), never `otv run` again.

## 10. LLM usage and actual cost

The pipeline calls an LLM at **four points**, and only one runs in the default path (mode A):

| Stage | Calls per ~25-minute video | What it sends to the model | When it runs |
|---|---|---|---|
| `pontuar` | **1** | the entire transcript split into numbered units (plain text) | always |
| `cenas --classificar` | **1 per batch of 20 thumbnails** (`otv/fases/cenas.py`) — ~378 scenes ≈ **19 calls**, multimodal, images in base64 | scene thumbnails | modes **B/C** only (and `--visual glm/gemini/claude_cli`) |
| `narrar` | **1** | the plan text, to rewrite for narration | modes **B/C/N** |
| `abertura` | **1** | title + headline + short list of scenes | when the opening is assembled |

### Measured figures (not estimates)

Real ~25-minute video (`trabalho/dQYKcjvXhIY/custos.json`), `glm` slot =
`z-ai/glm-5.3-flash` via OpenRouter:

| Stage | prompt tokens | completion tokens | Cost | Time |
|---|---|---|---|---|
| `pontuar` | 8.933 | 6.475 | **US$0,0046** | 85 s |
| `classificar` | 83.001 | 10.047 | **US$0,0099** | 195 s |
| `narrar` | 2.116 | 740 | **US$0,0007** | 303 s |
| `abertura` | 784 | 116 | **US$0,0002** | 101 s |
| **Total LLM (entire pipeline)** | | | **US$0,0154** | |

**Visual classification accounts for 90% of token usage** — the images are in base64, ~4.4k tokens
per batch of 20 thumbnails. It's the project's only multimodal step, and exactly what mode A
doesn't use.

### Cost by mode

| Mode | LLM calls | LLM cost | Total with transcription |
|---|---|---|---|
| **A** (default, `visual: local`) | only `pontuar` (+ `abertura`) | **~US$0,005** | **≈ US$0,025** |
| **N** | `pontuar` + `narrar` (+ `abertura`) | ~US$0,006 | ≈ US$0,026 |
| **B / C** | `pontuar` + `classificar` + `narrar` (+ `abertura`) | ~US$0,015 | ≈ US$0,035 |

Transcription with Groq adds ~US$0.02 to all of them (Groq doesn't report cost in `$`;
estimated at ~US$0.04 per hour of audio). Scenes, face detection, selection, render, and local
TTS cost R$0.

### What costs nothing or reduces cost

- **Re-cuts and re-renders cost R$0.** `selecionar`, `render`, and `narrar` (the TTS itself)
  don't call paid LLMs; `pontuar` and `cenas` save their artifacts and don't repeat without
  `--forcar` (section 8).
- `--pontuacao ollama` or `--visual ollama` removes the `$` cost (runs locally with
  `qwen3.8:27b`; requires the daemon).
- `--pontuacao claude_cli` / `--visual claude_cli` uses the Claude Code **subscription**, with no
  API key — the cost is recorded as `0.0` in `custos.json`.

### Warning: retries can double consumption

Each call has **2 attempts** if the response isn't valid JSON (`chat_json` in
`otv/provedores/llm.py`) and up to **3 HTTP attempts** for 5xx/429 errors. In the worst case,
a stage consumes twice what's shown in the table above.

Check the actual spend for a processed folder with:

```bash
python3 otv.py custo <id>
```

## 11. Troubleshooting

Real issues already encountered in this project (full changelog in `FALHAS.md`):

- **`inemavox` (TTS): `GET /api/jobs/{id}/download` returns 404.** The correct endpoint is
  `GET /api/jobs/{id}/audio` — already fixed in the code (`otv/provedores/tts.py`), but if you're
  pointing to a different version of the inemavox daemon, check the endpoint.
- **Headline disappears from the video without any error.** A stray `%` in the headline text
  (common in "100% de certeza" generated by an LLM) caused ffmpeg's `drawtext` to interpret
  `expansion=normal` and fail silently (warning swallowed by `-v error`, process exits with code 0
  and no headline). Fixed with `expansion=none` — if you edit `render.py`, don't remove this flag.
- **`sem_audio_original=True` wasn't muting anything.** `volume=0dB` is unity gain
  (changes nothing); actual zero gain is `volume=0`. Already fixed.
- **Continuous speech with no punctuation or pauses generated huge units** (one reached 30s, ~25%
  of a 120s cut budget). The forced cut via `max_unidade_s` fixes this — if you notice large units
  again, check that `selecao.max_unidade_s` hasn't been removed or set too high in `config.yaml`.
- **Gemini via the native API returning 503 on 3.x models, with 2.5 disabled for new accounts.**
  This is a Google account/infra limit, not a bug here — the `gemini` slot uses OpenRouter, not
  the native API, precisely to avoid depending on this.
- **`glm` via OpenRouter taking over 13 minutes to score.** The model has `reasoning` that can't
  be disabled on this endpoint; the fix was to send `reasoning: {effort: "low"}` +
  `max_tokens: 20000` (already in `config.yaml → openrouter:`) — without this, the call may never
  return in a reasonable time.
- **Groq returning `segments: null`** when `timestamp_granularities` requested only `word`. The
  fix was to request `[word, segment]` and handle `segments or []` — already in the code
  (`otv/provedores/transcricao.py`).

## 12. Links

- Design spec: [`docs/superpowers/specs/2026-08-27-otimizevideo-design.md`](docs/superpowers/specs/2026-08-27-otimizevideo-design.md)
- Implementation plan (tasks): [`docs/superpowers/plans/2026-08-27-otimizevideo.md`](docs/superpowers/plans/2026-08-27-otimizevideo.md)
- Provider research (prices, models, comparison): [`docs/pesquisa-provedores-2026-08-27.md`](docs/pesquisa-provedores-2026-08-27.md)
- Failure changelog: [`FALHAS.md`](FALHAS.md)


## 13. A+ mode — replace the presenter with illustrations

`--substituir gerado` (in `run`) or the `otv substituir <id>` stage generates a 16:9 illustration
for each segment labeled `talking_head` and replaces **only the video** in those clips: the audio
remains original, so the speech doesn't change at all. This only makes sense after visual
classification (`otv cenas --classificar`), because the plan's `visual` field determines which
segments show the presenter.

```bash
python3 otv.py run <url> --modo A --visual glm --substituir gerado
```

Images are saved in `trabalho/<id>/subst/seg_NN.png`, and the relative path is recorded in each
segment of `plan.json` (`"substituir": "subst/seg_00.png"`). The stage is idempotent: an
existing PNG is reused without another paid call — use `--forcar` to regenerate. Render assembles
these clips with `movie=…,zoompan` (slow Ken Burns effect) using the same video geometry, and
`otv status <id>` marks replaced segments with `→img`.

Image provider: `imagem: fal` in `config.yaml` (flux-2-klein via fal.ai, `FAL_KEY` read from
the usual `.env` files). It's the only implemented provider — to switch providers, write another
class with the same `gerar(prompt, destino)` method in `otv/provedores/imagem.py`.


## 14. Topic card (between the opening and the video)

When there is an opening, render inserts a **card** between it and the main video: a dark
background, the headline text, and a 1-second pause (0.25 s to fade in, 1 s on screen, 0.25 s
to fade out). The opening ends at a fast pace — short blocks with speech over them — and cutting
straight from it to the video makes the two feel like one thing to viewers; the card is the
breathing space that separates "the teaser" from "the video." Set `cartela_s: 0` in `config.yaml`
to disable it.

**The text isn't drawn with `drawtext`.** ffmpeg 6.1.1 truncates by the number of BYTES, not
characters: each accented letter removes one letter from the end. This happened in production —
the headline "IA e bilionários estêo tentando vencer o envelhecimento" appeared in the video as
"...vencer o envelhecimen". In PT-BR, nearly every headline has accents, so this was the rule,
not the exception, and it failed silently (ffmpeg exits with code 0). The headline and card are
now rasterized with PIL (`otv/util/texto.py`) and added with `overlay` — which also enabled line
wrapping based on actual glyph width. If you edit this overlay, **keep `shortest=1`**: the image
uses `loop=-1` and never reaches EOF, so with `shortest=0` the render never finishes (measured:
18 minutes of ffmpeg and 51 MB and growing, for a 5.5-second video).


## 15. N mode — narration over the entire video

Mode A keeps the original speech, so the cuts create jumps: the person changes topics and
intonation without a transition. Mode B solves the flow issue, but discards the presenter and
keeps only slides, demos, and charts. **Mode N** is the middle ground: it keeps the entire video
on screen (like A) and replaces the audio with rewritten narration (like B).

```bash
python3 otv.py run <url> --modo N
```

The narration is **not new content**: the prompt (`prompts/narrar_n.md`) says to translate and
make what was already said flow naturally — remove hesitation, bridge the jump left by the cut,
and keep numbers and proper names exactly as they are — and prohibits adding any fact, date, or
conclusion that wasn't in the speech. `roteiro.md` is the review point before rendering.

### The voice: `edge` + `pt-BR-FranciscaNeural` (default since 2026-09-01)

The default used to be `chatterbox` with the `rachel` voice. Chatterbox is a voice-cloning model
developed for English and **slipped into English phonetics** mid-sentence in Portuguese,
typically when it encountered a proper name ("DroidUp", "Walker 3", "Moya", "startup"): the
speech came out in another language even though the text was correct — the script was 100% in
Portuguese. No text changes could fix it; the engine was the problem.

Measured comparison using the same video and script:

| | `chatterbox` / `rachel` | `edge` / `pt-BR-FranciscaNeural` |
|---|---|---|
| total speech | 187,4 s | **156,3 s** |
| frozen image | 2,5 s in 2 segments | **0 s, none** |
| required stretch | up to 8 s per segment | 1–4,5 s |
| final output | 210,6 s | **190,3 s** |
| generating one sentence | 28,2 s | **4,1 s** |

Speaking faster solves three things at once: it eliminates freezing, reduces stretching
(which, in videos with burned-in captions, pulls text from another segment), and makes generation
7× faster. To switch back, set `voz: rachel` + `tts_engine: chatterbox` in `config.yaml`.

### Speech and image always stay in sync

Narration almost never has exactly the same duration as the clip, and both mismatches caused
visible defects (measured on the 2026-09-01 video):

| Situation | Before | Now |
|---|---|---|
| Speech is **longer** than the clip | froze the frame for up to 6 s, and `atrim` **cut off the sentence** mid-way (2,5 s lost in one segment) | `esticar_por_fala` extends `out` over the original video — moving image contiguous with the cut — and only freezes any remainder |
| Speech is **much shorter** than the clip | left silent video (one segment had 19,3 s of image for 4,9 s of speech: **14,4 s with no one speaking**) | `encolher_por_fala` shortens the segment to the speech + 0,6 s of breathing room, respecting `min_segmento_s` |

The stretch stops at the start of the next segment (it doesn't repeat material that will already
appear) and at the end of the video; without `metadata.json` (unknown duration), it doesn't
happen and behavior falls back to the old freeze, which always fits. Effect on the reference
video: the frozen image dropped from **44 s in 10 segments** (5 of them 5 s or more) to
**2,5 s in 2 segments**, and no sentence is cut off anymore.

Unlike the other modes, N **doesn't need visual classification** (it doesn't filter by image
type) and uses a longer freeze limit: **6 s** instead of 3 s. Portuguese is more verbose than
English, so narration tends to run longer than the clip — instead of truncating the sentence,
the last frame freezes and waits for the speech to finish, then the next clip starts.

## 16. Run without Claude Code or Codex (headless VPS)

**Yes — the entire pipeline runs on a VPS without any coding agent installed.** `otv` is a
regular Python CLI; `claude_cli` is just *one optional provider* for the `visual` and
`pontuacao` slots, never a dependency. Codex isn't used anywhere in the project.

What the VPS actually needs:

| Item | Required? | Notes |
|---|---|---|
| Python 3 + `requirements.txt` | yes | `requests`, `pyyaml`, `yt-dlp`, `scenedetect`, `opencv-python`, `mediapipe` |
| `ffmpeg` / `ffprobe` on the `PATH` | yes | cutting, thumbnails, mux |
| `yt-dlp` | only if the source is a URL | local files don't need it |
| `GROQ_API_KEY` | yes (default transcription) | or switch to `whisper_local`/`whisperx` |
| `OPENROUTER_API_KEY` | yes (default scoring) | or switch to `ollama` |
| GPU | **no** | `visual: local` (mediapipe) and scene detection run on CPU |
| `inemavox` daemon (:8010) | modes B/C/N only | TTS — without it, use `--tts elevenlabs` |
| Ollama daemon (:11434) | only if choosing `ollama` | |
| `claude` binary | **no** | only if choosing `--pontuacao claude_cli` / `--visual claude_cli` |

Minimal 100% cloud config, with no daemon and no GPU:

```yaml
transcricao: groq
visual: local        # mode A; for B/C use glm (also cloud)
pontuacao: glm
tts: elevenlabs      # avoids relying on local inemavox
```

Two caveats for a small VPS:

- **`opencv-python` + `mediapipe` require system libraries.** In a slim/headless image,
  install `libgl1` and `libglib2.0-0`, or switch to `opencv-python-headless`.
- **Scene detection is CPU-bound**: 130 s for a 25-minute video on the reference machine.
  Expect much longer on a 1 vCPU VPS, but it runs only once and is idempotent.

**Keys don't require the source machine's paths.** On a VPS, use any of the portable sources
(section 2 → *Keys*): environment variables in systemd/Docker, a `.env` in the project root,
`~/.config/otv/.env`, or `OTV_ENV_FILE` pointing wherever you want.

```bash
# option 1: environment (nothing on disk)
GROQ_API_KEY=... OPENROUTER_API_KEY=... python3 otv.py run <url> --modo A

# option 2: .env in the project root (ignored by git)
printf 'GROQ_API_KEY=...\nOPENROUTER_API_KEY=...\n' > .env
```

## 17. Web panel — submit jobs and see what has already run

```bash
./start.sh                 # starts the panel; ./start.sh 9000 uses another port
./stop.sh                  # stops it (refuses if a job is running)
```

`start.sh` runs preflight checks before starting — `ffmpeg`/`ffprobe` on the `PATH`, Python
modules from `requirements.txt`, `yt-dlp`, and whether keys can be found (warns only, doesn't
block) — because failing here is cheaper than failing halfway through a paid job. It then runs
with `setsid`, so **the panel survives closing the terminal**, and writes
`trabalho/.painel/painel.pid`.

It refuses to start if a panel is already running **on any port**: two instances would share
`trabalho/.painel/`, and the second would overwrite the first one's job history. `./stop.sh`
**refuses to stop while a job is running** (`--forcar` kills the job too). Options:
`./start.sh --local` binds to `127.0.0.1`; `./start.sh --token` generates a token and requires
`?t=…`.

Without the scripts, the raw command is `python3 otv.py painel [--porta 8022] [--host 0.0.0.0]`.

It starts a server from the **stdlib** (no Flask, no new dependency) that listens on
`0.0.0.0`, so any device on the LAN can open it using the machine's IP. It runs **inside the
project**: uses the same `config.yaml` and `trabalho/` folder, and launches `otv.py` itself as
a subprocess — there is no parallel path; the panel can't do anything the CLI can't do.

On screen:

- **Submit** — paste a YouTube URL *or the path to a local video*, choose the mode (each option
  describes the output: `A — original speech`, `N — entire video, re-recorded narration`,
  `B — without presenter`, `C — demos and charts only`), target, visual slot, and the **A+**
  checkbox (replace the presenter with generated illustrations), then click Run. A hint below
  the form explains the selected mode and **warns when the combination will fail** — B, C, and
  A+ require `Visual = glm`, because `local` detection can't identify slides/demos/charts. The
  queue is **serial, one job at a time** (scene detection is CPU-bound; two parallel jobs only
  get in each other's way).
- **Monitor** — the live log is the stage stdout (`[ingest] ok`, `[pontuar] ok`…), saved to
  `trabalho/.painel/painel-<job>.log`. The job is a subprocess, so **it keeps running if you
  close the browser** — reopening reconnects to the log.
- **See what has already run** — each run in `trabalho/` with thumbnail, title, original
  duration, mode, cut duration, headline, cost in US$, and existing artifacts. Runs with an
  `output.mp4` get an inline player (with `Range`, so you can seek).
- **Re-cut for free** — `re-selecionar` (request another target/mode) and `re-render` buttons
  for each run. Neither calls an LLM: US$0, as many times as you want (section 8).
- **Edit the cut** — `editar corte` opens the segment list: you can uncheck a segment (remove it
  from the cut), adjust `in`/`out`, and change the headline, with the total recalculated
  immediately. `salvar plano` saves `plan.json` (the previous one becomes `plan.bak.json`);
  `salvar e renderizar` also queues the render. This is section 9's editing through the UI — the
  server only accepts `in`, `out`, segment inclusion, and the headline: `unidades`, `visual`,
  and `substituir` are preserved from the original plan, so a malformed POST can't invent a
  field or erase what the pipeline calculated. Removing a segment with generated narration
  invalidates the WAV-to-segment mapping — the panel warns and clears the narration, requiring
  `narrar` again before rendering in modes B and N.

### Where it is accessible

When it starts, the panel prints all IPv4 addresses where it responds:

```
panel: http://localhost:8022
        http://192.168.1.172:8022
        http://100.70.253.64:8022  (tailscale — reachable outside the LAN)
```

**Note the Tailscale address.** On a machine with a tailnet, the LAN isn't the limit: the panel
responds to any device on the tailnet, including outside the home. If the firewall blocks the
port (not the case on the reference machine), allow it with `sudo ufw allow 8022/tcp`; to bind
only to the local machine, use `--host 127.0.0.1`.

### Security

No authentication by default — the assumption is a home LAN. Set `OTV_PAINEL_TOKEN` to require
a token (`http://…:8022/?t=<token>`):

```bash
OTV_PAINEL_TOKEN=$(openssl rand -hex 16) python3 otv.py painel
```

There are two safeguards because this is exposed to the network; do not remove them:

- **every command is passed to `Popen` as an argument list, never through a shell** — without
  this, the source field would become arbitrary command execution for anyone on the LAN;
- **every ID is resolved and checked against the root of `trabalho/`** before becoming a file
  path, and the runnable stages are an allowlist (`selecionar`, `render`, `narrar`,
  `abertura`, `substituir`) — without this, `../../etc/passwd` would become a download.

Remember that **any device that can reach the machine can start a paid job** (~US$0.03
per video) — LAN and tailnet included. The panel warns at startup when it runs without a token.
If that's a problem, set `OTV_PAINEL_TOKEN` or use `--host 127.0.0.1`.
