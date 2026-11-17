# otv â€” otimizevideo

**ðŸ‡§ðŸ‡· [PortuguÃªs](README.md) Â· ðŸ‡ºðŸ‡¸ [English](README.en.md) Â· ðŸ‡ªðŸ‡¸ [EspaÃ±ol](README.es.md)**

CLI que pega um vÃ­deo longo (aula, podcast, palestra â€” 20 a 30 min) e devolve um corte de
**~2 minutos** com o melhor conteÃºdo, cortado com precisÃ£o de palavra.

## ðŸ“– Guia de uso

Guia completo (landing + passo a passo): **https://inematds.github.io/otimizevideo/guia/**

## 1. O que faz

`otv` roda um vÃ­deo por um pipeline de fases (transcriÃ§Ã£o â†’ detecÃ§Ã£o de cena â†’
classificaÃ§Ã£o visual â†’ pontuaÃ§Ã£o â†’ seleÃ§Ã£o â†’ render) e produz um `output.mp4` condensado.
A regra de ouro do projeto, e o que o diferencia de "manda o vÃ­deo pro modelo e pede um
corte": **o modelo nunca escolhe timestamps**. Ele sÃ³ vÃª a transcriÃ§Ã£o fatiada em unidades
numeradas por id (`[042] 4.2s talking_head "texto da unidade"`) e devolve uma nota de 0 a 10
por id. Ã‰ o cÃ³digo â€” nÃ£o o modelo â€” que converte id â†’ tempo (`ini`/`fim` em segundos, vindos
direto da transcriÃ§Ã£o com timestamp por palavra) e faz o corte.

Isso importa por dois motivos: (1) o corte nunca cai no meio de uma palavra, porque a
fronteira de cada unidade jÃ¡ Ã© uma fronteira de palavra/pausa real da transcriÃ§Ã£o, nunca um
timestamp inventado pelo modelo; e (2) o resultado Ã© reproduzÃ­vel â€” rodar a mesma seleÃ§Ã£o
duas vezes com o mesmo `notas.json` dÃ¡ o mesmo `plan.json`, porque a lÃ³gica de escolha
(mochila por nota, cota por tÃ³pico, gancho/fecho, coesÃ£o) Ã© determinÃ­stica e vive no cÃ³digo,
nÃ£o numa alucinaÃ§Ã£o do LLM.

## 2. InstalaÃ§Ã£o

```bash
pip install -r requirements.txt
```

O Python do sistema nesta mÃ¡quina Ã© PEP 668 (ambiente "externally managed") â€” se o comando
acima falhar com `error: externally-managed-environment`, use:

```bash
pip install --user --break-system-packages -r requirements.txt
```

TambÃ©m precisam estar no `PATH`:

- **ffmpeg** e **ffprobe** (corte, extraÃ§Ã£o de Ã¡udio, thumbnails, mux final)
- **yt-dlp** (baixar vÃ­deo de URL â€” sÃ³ necessÃ¡rio se `otv run <url>`, nÃ£o para arquivo local)

`whisper_local`, `whisperx` e `ollama` (provedores opcionais, ver tabela na seÃ§Ã£o 5) trazem
dependÃªncias pesadas (`openai-whisper`, `whisperx`, `torch`) que **nÃ£o** estÃ£o em
`requirements.txt` â€” instale-as sÃ³ se for usar esses provedores.

### Keys

`otv` nunca escreve chave de API em `config.yaml` nem em qualquer arquivo deste projeto â€”
elas sÃ£o lidas em **runtime** (`otv/util/keys.py`), na primeira fonte que tiver a variÃ¡vel:

1. **variÃ¡vel de ambiente** â€” `GROQ_API_KEY=... python3 otv.py run ...`, systemd, Docker
2. o arquivo apontado por **`OTV_ENV_FILE`**, se definido
3. **`.env` na raiz do projeto** (estÃ¡ no `.gitignore`)
4. **`~/.config/otv/.env`**
5. os `.env` da mÃ¡quina de origem â€” `~/projetos/openpcbotv2/.env`, `~/projetos/wifi/.env`

As opÃ§Ãµes 1â€“4 tornam o projeto portÃ¡vel para outra mÃ¡quina ou VPS sem editar cÃ³digo
(seÃ§Ã£o 15). Linhas com `#` sÃ£o ignoradas e `export NOME=valor` Ã© aceito.

Configure `GROQ_API_KEY`, `OPENROUTER_API_KEY` e, se for usar ElevenLabs,
`ELEVENLABS_API_KEY`/`ELEVENLABS_VOICE_ID` (mais `FAL_KEY` para o modo A+). Nenhuma key Ã©
impressa, logada ou gravada em disco por este projeto.

## 3. Uso rÃ¡pido

```bash
python3 otv.py run <url-ou-caminho-do-video> --modo A --alvo 120
```

Isso roda o pipeline inteiro (ingest â†’ transcrever â†’ cenas â†’ pontuar â†’ selecionar â†’ render)
e escreve o resultado em `~/projetos/output/otimizevideo/<id>/output.mp4` (o `<id>` Ã© o
ID do vÃ­deo do YouTube, ou o nome do arquivo normalizado, quando a fonte Ã© local). O mesmo
`output.mp4`, junto com `plan.json`, `notas.json`, `unidades.json` e `custos.json`, tambÃ©m
fica em `trabalho/<id>/` (pasta de trabalho intermediÃ¡ria).

## 4. Os trÃªs modos

| Modo | O que sai | Quando usar | O que exige |
|---|---|---|---|
| **A** | Condensado com a **fala original** do apresentador (talking head incluso) | PadrÃ£o â€” preserva o tom e a voz de quem gravou | Nada alÃ©m da transcriÃ§Ã£o; roda 100% com `visual: local` |
| **B** | **Sem** o apresentador â€” corta pra tela/grÃ¡fico/slide e **regrava a narraÃ§Ã£o** (TTS) por cima | Quando o rosto/olho no vÃ­deo original nÃ£o importa e vocÃª quer um corte mais "editorial" | ClassificaÃ§Ã£o visual por modelo (as cenas precisam estar marcadas como `demo_tela`, `slide` ou `grafico` â€” a detecÃ§Ã£o local sÃ³ distingue "tem rosto" de "nÃ£o tem") |
| **C** | SÃ³ **demonstraÃ§Ãµes e grÃ¡ficos** â€” o modo mais restrito | VÃ­deo Ã© majoritariamente demo/grÃ¡fico e vocÃª quer sÃ³ isso, sem qualquer talking head | Mesma exigÃªncia do modo B |

HÃ¡ ainda dois modos derivados, documentados no fim deste README: **A+** (mantÃ©m a fala
original mas troca o vÃ­deo do apresentador por ilustraÃ§Ã£o gerada â€” seÃ§Ã£o 13) e **N** (mantÃ©m
o vÃ­deo inteiro na tela e regrava sÃ³ o Ã¡udio como narraÃ§Ã£o â€” seÃ§Ã£o 14).

**B e C dependem de classificaÃ§Ã£o visual por modelo.** A detecÃ§Ã£o local (`visual: local`) sÃ³
sabe dizer "tem rosto grande" (`talking_head`) ou "nÃ£o" (`outro`) â€” ela nunca produz
`demo_tela`, `slide` ou `grafico`. Sem passar `--visual glm`, `--visual gemini` ou
`--visual claude_cli` (ou rodar `otv cenas <id> --classificar` manualmente), `otv run
--modo B` (ou `C`) segue em frente com um aviso e falha depois, na fase de seleÃ§Ã£o, com uma
mensagem clara dizendo que nÃ£o hÃ¡ unidade com o visual necessÃ¡rio.

## 5. Slots do `config.yaml`

O `config.yaml` na raiz define, por slot, qual provedor usar por padrÃ£o. Todo slot pode ser
sobreposto por linha de comando (`--transcricao`, `--visual`, `--pontuacao`, `--tts` em
`otv run`; `--provedor` em `otv transcrever`/`cenas`/`pontuar`/`narrar`), e o arquivo inteiro
pode ser trocado com `--config outro.yaml`.

| Slot | Provedores | Custo/exigÃªncia | Default |
|---|---|---|---|
| `transcricao` | `groq` | Cloud, precisa de `GROQ_API_KEY`. NÃ£o reporta custo em `$`; ~US$0,04/hora de Ã¡udio | **default** |
| | `whisper_local` | GPU/CPU local, grÃ¡tis, mas precisa de `openai-whisper` instalado (fora do `requirements.txt`) | |
| | `whisperx` | GPU local (CUDA se disponÃ­vel), grÃ¡tis, precisa de `whisperx`+`torch` instalados | |
| `visual` | `local` | GrÃ¡tis, sem key â€” heurÃ­stica por Ã¡rea de rosto (mediapipe). SÃ³ distingue `talking_head`/`outro`, nunca `demo_tela`/`slide`/`grafico` | **default** |
| | `glm` (`z-ai/glm-5.3-flash` via OpenRouter) | Precisa de `OPENROUTER_API_KEY`. Barato (~US$0,004 por vÃ­deo de 25 min) | |
| | `gemini` (`google/gemini-2.5-flash-lite` via OpenRouter) | Precisa de `OPENROUTER_API_KEY` | |
| | `claude_cli` (Claude Code headless, `claude -p --model sonnet`) | Sai da **assinatura** do usuÃ¡rio â€” sem API key, mas precisa do binÃ¡rio `claude` no `PATH` e logado | |
| `pontuacao` | `glm` (`z-ai/glm-5.3-flash` via OpenRouter) | Precisa de `OPENROUTER_API_KEY`. ~US$0,002 por vÃ­deo de 25 min (1 chamada, `reasoning_effort: low`) | **default** |
| | `gemini` (`google/gemini-2.5-flash-lite` via OpenRouter) | Precisa de `OPENROUTER_API_KEY` | |
| | `ollama` (`qwen3.8:27b`) | Local, grÃ¡tis, mas precisa de **daemon Ollama** rodando (`http://localhost:11434`) | |
| | `claude_cli` (Claude Code headless) | Sai da assinatura â€” sem API key, precisa do `claude` no `PATH` | |
| `tts` | `inemavox` | Local, grÃ¡tis, mas precisa do **daemon inemavox** rodando (`http://localhost:8010`). Voz e motor saem de `config.yaml â†’ voz:` / `tts_engine:` â€” o default Ã© **`edge` + `pt-BR-FranciscaNeural`** (ver seÃ§Ã£o 15) | **default** |
| | `elevenlabs` | Cloud, precisa de `ELEVENLABS_API_KEY` e `ELEVENLABS_VOICE_ID`; cobra por caractere | |

Os modelos concretos de cada slot de LLM (`glm`, `gemini`, `ollama`, `claude_cli`, e o
`turbo` do `whisper_local`) ficam em `config.yaml â†’ modelos:` â€” trocar o modelo nÃ£o muda o
slot, sÃ³ o que aquele slot chama.

## 6. ParÃ¢metros de `selecao` (`config.yaml â†’ selecao:`)

| ParÃ¢metro | Default | Efeito prÃ¡tico |
|---|---|---|
| `alvo_s` | 120 | DuraÃ§Ã£o alvo, em segundos, do vÃ­deo final (mesmo valor que `--alvo` sobrepÃµe) |
| `tolerancia` | 0.25 | Faixa aceitÃ¡vel ao redor do alvo: teto = `alvo_s Ã— 1.25`, piso = `alvo_s Ã— 0.75`. A seleÃ§Ã£o usa esse teto como orÃ§amento extra para coesÃ£o/gancho/fecho depois que a mochila principal jÃ¡ saturou o `alvo_s` puro |
| `min_segmento_s` | 3 | Qualquer segmento final com menos que isso (depois do corte/snap) Ã© descartado â€” evita cortes de fraÃ§Ã£o de segundo |
| `min_segmento_ideal_s` | 8 | Meta de "coesÃ£o": cada segmento tenta puxar vizinhos com nota razoÃ¡vel atÃ© atingir esse tamanho, pra nÃ£o virar uma sequÃªncia de picadinhos |
| `cota_topico_pct` | 40 | Teto, em % do `alvo_s`, que um Ãºnico tÃ³pico pode ocupar na mochila principal â€” evita que o corte vire "sÃ³ a parte mais falada", garantindo diversidade de assunto |
| `nota_minima` | 5 | Nota mÃ­nima (0â€“10) pra uma unidade entrar na mochila principal. Gancho/fecho e as extensÃµes de coesÃ£o toleram um pouco menos (nota_minima âˆ’ 2 ou âˆ’ 3) pra poder completar o segmento |
| `folga_ms` | 120 | Margem extra, em milissegundos, deixada depois do fim da Ãºltima unidade de um segmento ao fazer o snap â€” evita cortar em cima da Ãºltima sÃ­laba |
| `pausa_fronteira_ms` | 400 | Pausa mÃ­nima entre palavras (na fase de `unidades`) que jÃ¡ conta como fronteira natural de unidade â€” junto com pontuaÃ§Ã£o (`.`/`!`/`?`) e fim de segmento da transcriÃ§Ã£o, Ã© um dos critÃ©rios que fecha uma unidade |
| `max_unidade_s` | 12 | Teto de duraÃ§Ã£o de uma unidade cortÃ¡vel â€” mesmo sem pausa nem pontuaÃ§Ã£o (fala corrida), o cÃ³digo forÃ§a um corte na maior pausa interna disponÃ­vel antes de estourar esse teto |

## 7. Todos os subcomandos

```bash
# pipeline completo
python3 otv.py run <url-ou-caminho> --modo A --alvo 120

# sÃ³ baixa/copia o vÃ­deo e extrai Ã¡udio, sem processar nada
python3 otv.py ingest <url-ou-caminho>

# fases individuais, sobre uma pasta jÃ¡ ingerida (o <id> Ã© o nome da pasta em trabalho/)
python3 otv.py transcrever <id> --provedor groq
python3 otv.py cenas <id> --classificar --provedor glm
python3 otv.py pontuar <id> --modo A --alvo 120 --provedor glm
python3 otv.py selecionar <id> --modo A --alvo 120
python3 otv.py substituir <id> --provedor fal     # modo A+: troca o apresentador por ilustraÃ§Ã£o
python3 otv.py render <id> --rapido
python3 otv.py narrar <id> --provedor inemavox

# inventÃ¡rio de artefatos + resumo do plano de corte
python3 otv.py status <id>

# gasto acumulado por fase, lido de trabalho/<id>/custos.json
python3 otv.py custo <id>
```

## 8. Rodar fase por fase e reaproveitar trabalho

Cada fase escreve seu prÃ³prio JSON em `trabalho/<id>/` (`transcript.json`, `scenes.json`,
`unidades.json`, `notas.json`, `plan.json`, `custos.json`...) e a maioria Ã© **idempotente**:
se o artefato jÃ¡ existe, a fase nÃ£o roda de novo â€” sÃ³ devolve o que jÃ¡ estÃ¡ no disco.
`--forcar` obriga a refazer.

**TrÃªs exceÃ§Ãµes sempre refazem, mesmo sem `--forcar`:** `selecionar` (assina o plano de novo
toda vez â€” Ã© o comportamento certo pra poder rodar `otv selecionar <id> --modo B --alvo 90`
repetidas vezes atÃ© achar a combinaÃ§Ã£o boa, sem editar cÃ³digo), `render` (nÃ£o tem noÃ§Ã£o de
"jÃ¡ existe" â€” sempre re-renderiza a partir do `plan.json` atual, Ã© assim que a seÃ§Ã£o 9
funciona) e `narrar` (idem â€” sempre gera os wavs de novo a partir do `plan.json` atual). As
fases `ingest`, `transcrever`, `cenas`, `pontuar` sÃ£o as que de fato pulam quando o artefato
jÃ¡ existe.

Isso economiza dinheiro de verdade: se a pontuaÃ§Ã£o (`otv pontuar`) jÃ¡ rodou e custou
uma chamada de LLM, vocÃª pode rodar `otv selecionar` vÃ¡rias vezes com `--modo`/`--alvo`
diferentes, ou `otv render` de novo depois de editar o plano Ã  mÃ£o (seÃ§Ã£o 9), sem pagar
nenhuma chamada de modelo outra vez â€” `selecionar`/`render`/`narrar` refazerem sempre nÃ£o
custa nada, porque nÃ£o chamam LLM nenhum. SÃ³ refaÃ§a `ingest`/`transcrever`/`cenas`/`pontuar`
com `--forcar` quando precisar mesmo de um resultado novo dela (ex.: transcriÃ§Ã£o errada,
classificaÃ§Ã£o visual desatualizada depois de trocar de provedor).

## 9. Editar `plan.json` na mÃ£o e re-renderizar

`plan.json` Ã© texto â€” dÃ¡ pra editar o corte final sem chamar nenhum modelo. Ã‰ esse recurso
que dÃ¡ controle editorial ao usuÃ¡rio sobre a escolha automÃ¡tica.

Um segmento tem este formato:

```json
{
  "in": 175.1,
  "out": 182.5,
  "unidades": [58, 59, 60],
  "visual": "talking_head",
  "motivo": "explica o resultado do experimento",
  "texto": "The judges said that the 50-year-old problem was essentially solved.",
  "estender_s": 0
}
```

DÃ¡ pra remover um segmento inteiro da lista `segmentos`, ajustar `in`/`out` manualmente
(cuidado: `unidades` deixa de bater exatamente com o novo intervalo, mas o render sÃ³ usa
`in`/`out`), ou editar o texto de `manchete` no topo do JSON. Depois de editar, **nÃ£o** rode
`otv selecionar` de novo (ele reescreve o `plan.json` do zero, sempre â€” ver seÃ§Ã£o 8) â€” vÃ¡
direto pro render:

```bash
python3 otv.py render <id>
```

**Cuidado com `otv run` na mesma pasta.** `otv run <fonte>` sem `--forcar` pula
`ingest`/`transcrever`/`cenas`/`pontuar` se os artefatos jÃ¡ existirem, mas ele sempre chama
`selecionar` internamente â€” que, como a seÃ§Ã£o 8 explica, reescreve o `plan.json` mesmo sem
`--forcar`. Rodar `otv run` de novo na mesma pasta depois de editar o `plan.json` Ã  mÃ£o
**apaga a ediÃ§Ã£o em silÃªncio**. Depois de editar o plano, o comando certo Ã© `otv render <id>`
isolado (acima), nunca `otv run` de novo.

## 10. Consumo de LLM e custo real

O pipeline chama LLM em **quatro pontos**, e sÃ³ um deles roda no caminho padrÃ£o (modo A):

| Fase | Chamadas por vÃ­deo de ~25 min | O que manda pro modelo | Quando roda |
|---|---|---|---|
| `pontuar` | **1** | a transcriÃ§Ã£o inteira fatiada em unidades numeradas (texto puro) | sempre |
| `cenas --classificar` | **1 por lote de 20 thumbnails** (`otv/fases/cenas.py`) â€” ~378 cenas â‰ˆ **19 chamadas**, multimodal, imagens em base64 | as thumbs das cenas | sÃ³ modos **B/C** (e `--visual glm/gemini/claude_cli`) |
| `narrar` | **1** | o texto do plano, pra reescrever pra locuÃ§Ã£o | modos **B/C/N** |
| `abertura` | **1** | tÃ­tulo + manchete + lista curta de cenas | quando a abertura Ã© montada |

### NÃºmeros medidos (nÃ£o estimados)

VÃ­deo real de ~25 min (`trabalho/dQYKcjvXhIY/custos.json`), slot `glm` =
`z-ai/glm-5.3-flash` via OpenRouter:

| Fase | prompt tokens | completion tokens | Custo | Tempo |
|---|---|---|---|---|
| `pontuar` | 8.933 | 6.475 | **US$0,0046** | 85 s |
| `classificar` | 83.001 | 10.047 | **US$0,0099** | 195 s |
| `narrar` | 2.116 | 740 | **US$0,0007** | 303 s |
| `abertura` | 784 | 116 | **US$0,0002** | 101 s |
| **Total de LLM (pipeline inteiro)** | | | **US$0,0154** | |

**A classificaÃ§Ã£o visual Ã© 90% do consumo de tokens** â€” sÃ£o imagens em base64, ~4,4k tokens
por lote de 20 thumbs. Ã‰ o Ãºnico ponto multimodal do projeto, e Ã© exatamente o que o modo A
nÃ£o usa.

### Custo por modo

| Modo | LLM que roda | Custo de LLM | Total com transcriÃ§Ã£o |
|---|---|---|---|
| **A** (default, `visual: local`) | sÃ³ `pontuar` (+ `abertura`) | **~US$0,005** | **â‰ˆ US$0,025** |
| **N** | `pontuar` + `narrar` (+ `abertura`) | ~US$0,006 | â‰ˆ US$0,026 |
| **B / C** | `pontuar` + `classificar` + `narrar` (+ `abertura`) | ~US$0,015 | â‰ˆ US$0,035 |

A transcriÃ§Ã£o no Groq entra com ~US$0,02 em todos eles (o Groq nÃ£o reporta custo em `$`;
estimativa de ~US$0,04 por hora de Ã¡udio). Cenas, detecÃ§Ã£o de rosto, seleÃ§Ã£o, render e TTS
local sÃ£o R$0.

### O que zera ou reduz

- **Re-cortes e re-renders sÃ£o R$0.** `selecionar`, `render` e `narrar` (o TTS em si) nÃ£o
  chamam LLM pago; `pontuar` e `cenas` gravam o artefato e nÃ£o repetem sem `--forcar`
  (seÃ§Ã£o 8).
- `--pontuacao ollama` ou `--visual ollama` tira o custo em `$` (roda local em
  `qwen3.8:27b`, precisa do daemon).
- `--pontuacao claude_cli` / `--visual claude_cli` sai da **assinatura** do Claude Code, sem
  API key â€” o custo Ã© registrado como `0.0` no `custos.json`.

### AtenÃ§Ã£o: retry pode dobrar o consumo

Cada chamada tem **2 tentativas** se a resposta nÃ£o vier em JSON vÃ¡lido (`chat_json` em
`otv/provedores/llm.py`) e atÃ© **3 tentativas** no HTTP para erros 5xx/429. Na pior hipÃ³tese
uma fase consome o dobro do que a tabela acima mostra.

Veja o gasto real de uma pasta jÃ¡ processada com:

```bash
python3 otv.py custo <id>
```

## 11. SoluÃ§Ã£o de problemas

Casos reais jÃ¡ encontrados neste projeto (changelog completo em `FALHAS.md`):

- **`inemavox` (TTS): `GET /api/jobs/{id}/download` devolve 404.** O endpoint certo Ã©
  `GET /api/jobs/{id}/audio` â€” jÃ¡ corrigido no cÃ³digo (`otv/provedores/tts.py`), mas se vocÃª
  estiver apontando pra outra versÃ£o do daemon inemavox, confira o endpoint.
- **Manchete sumindo do vÃ­deo sem erro nenhum.** Um `%` solto no texto da manchete (comum em
  "100% de certeza" vindo de LLM) fazia o `drawtext` do ffmpeg interpretar `expansion=normal`
  e falhar silenciosamente (warning engolido por `-v error`, processo sai com cÃ³digo 0 e sem
  manchete). Corrigido com `expansion=none` â€” se vocÃª mexer em `render.py`, nÃ£o tire essa flag.
- **`sem_audio_original=True` nÃ£o estava silenciando nada.** `volume=0dB` Ã© ganho unitÃ¡rio
  (nÃ£o muda nada); o ganho real-zero Ã© `volume=0`. JÃ¡ corrigido.
- **Fala corrida sem pontuaÃ§Ã£o nem pausa gerava unidades gigantes** (uma chegou a 30s, ~25%
  do orÃ§amento de um corte de 120s). O corte forÃ§ado por `max_unidade_s` resolve isso â€” se
  vocÃª notar unidades grandes de novo, confira se `selecao.max_unidade_s` nÃ£o foi removido ou
  aumentado demais no `config.yaml`.
- **Gemini via API nativa devolvendo 503 nos modelos 3.x, e 2.5 desligado pra contas novas.**
  Ã‰ limite de conta/infra da Google, nÃ£o bug daqui â€” o slot `gemini` usa OpenRouter, nÃ£o a API
  nativa, exatamente para nÃ£o depender disso.
- **`glm` via OpenRouter travando mais de 13 minutos numa pontuaÃ§Ã£o.** O modelo tem
  `reasoning` que nÃ£o pode ser desligado nesse endpoint; a correÃ§Ã£o foi mandar
  `reasoning: {effort: "low"}` + `max_tokens: 20000` (jÃ¡ no `config.yaml â†’ openrouter:`) â€” sem
  isso a chamada pode nunca voltar em tempo hÃ¡bil.
- **Groq devolvendo `segments: null`** quando `timestamp_granularities` sÃ³ pedia `word`. A
  correÃ§Ã£o foi pedir `[word, segment]` e tratar `segments or []` â€” jÃ¡ no cÃ³digo
  (`otv/provedores/transcricao.py`).

## 12. Links

- Spec de design: [`docs/superpowers/specs/2026-08-27-otimizevideo-design.md`](docs/superpowers/specs/2026-08-27-otimizevideo-design.md)
- Plano de implementaÃ§Ã£o (tasks): [`docs/superpowers/plans/2026-08-27-otimizevideo.md`](docs/superpowers/plans/2026-08-27-otimizevideo.md)
- Pesquisa de provedores (preÃ§os, modelos, comparaÃ§Ã£o): [`docs/pesquisa-provedores-2026-08-27.md`](docs/pesquisa-provedores-2026-08-27.md)
- Changelog de falhas: [`FALHAS.md`](FALHAS.md)


## 13. Modo A+ â€” substituir o apresentador por ilustraÃ§Ã£o

`--substituir gerado` (no `run`) ou a fase `otv substituir <id>` gera uma ilustraÃ§Ã£o 16:9 por
segmento marcado como `talking_head` e troca **sÃ³ o vÃ­deo** desses trechos: o Ã¡udio continua
sendo o original, entÃ£o a fala nÃ£o muda em nada. SÃ³ faz sentido depois que o visual foi
classificado (`otv cenas --classificar`), porque Ã© o campo `visual` do plano que decide o que
Ã© apresentador.

```bash
python3 otv.py run <url> --modo A --visual glm --substituir gerado
```

As imagens ficam em `trabalho/<id>/subst/seg_NN.png` e o caminho relativo Ã© anotado em cada
segmento do `plan.json` (`"substituir": "subst/seg_00.png"`). A fase Ã© idempotente: um PNG que
jÃ¡ existe Ã© reaproveitado sem nova chamada paga â€” use `--forcar` para regerar. O render monta
esses trechos com `movie=â€¦,zoompan` (Ken Burns lento) na mesma geometria do vÃ­deo, e o
`otv status <id>` marca com `â†’img` o que jÃ¡ foi trocado.

Provedor de imagem: `imagem: fal` no `config.yaml` (flux-2-klein via fal.ai, `FAL_KEY` lida dos
`.env` de sempre). Ã‰ o Ãºnico provedor implementado â€” trocar Ã© escrever outra classe com o mesmo
mÃ©todo `gerar(prompt, destino)` em `otv/provedores/imagem.py`.


## 14. Cartela de assunto (entre a abertura e o vÃ­deo)

Quando existe abertura, o render insere entre ela e o corpo uma **cartela**: fundo escuro, o
texto da manchete e uma parada de 1 s (0,25 s pra entrar, 1 s parado, 0,25 s pra sair). A
abertura termina em ritmo alto â€” blocos curtos com fala por cima â€” e emendar o vÃ­deo direto
nela faz as duas coisas virarem uma sÃ³ pra quem assiste; a cartela Ã© a respirada que separa
"a chamada" do "o vÃ­deo". `cartela_s: 0` no `config.yaml` desliga.

**O texto nÃ£o Ã© desenhado com `drawtext`.** O `drawtext` do ffmpeg 6.1.1 trunca pelo nÃºmero
de BYTES e nÃ£o de caracteres: cada letra acentuada come uma letra do fim. Isso saiu em
produÃ§Ã£o â€” a manchete "IA e bilionÃ¡rios estÃ£o tentando vencer o envelhecimento" apareceu no
vÃ­deo como "...vencer o envelhecimen". Em PT-BR quase toda manchete tem acento, entÃ£o era a
regra e nÃ£o a exceÃ§Ã£o, e silenciosa (o ffmpeg sai com cÃ³digo 0). A manchete e a cartela agora
sÃ£o rasterizadas com PIL (`otv/util/texto.py`) e entram por `overlay` â€” o que tambÃ©m deu
quebra de linha por largura real de glifo. Se vocÃª mexer nesse overlay, **mantenha
`shortest=1`**: a imagem entra com `loop=-1` e nunca dÃ¡ EOF, entÃ£o com `shortest=0` o render
nÃ£o termina nunca (medido: 18 min de ffmpeg e 51 MB e subindo, num vÃ­deo de 5,5 s).


## 15. Modo N â€” narraÃ§Ã£o sobre o conteÃºdo inteiro

O modo A mantÃ©m a fala original, e por isso os cortes deixam saltos: a pessoa muda de assunto e
de entonaÃ§Ã£o sem transiÃ§Ã£o. O modo B resolve a fluÃªncia, mas descarta o apresentador e sÃ³ fica
com slide, demo e grÃ¡fico. O **modo N** Ã© o meio-termo: mantÃ©m o vÃ­deo inteiro na tela (como o A)
e substitui o Ã¡udio por uma narraÃ§Ã£o reescrita (como o B).

```bash
python3 otv.py run <url> --modo N
```

A narraÃ§Ã£o **nÃ£o Ã© conteÃºdo novo**: o prompt (`prompts/narrar_n.md`) manda traduzir e dar fluÃªncia
ao que jÃ¡ foi dito â€” tirar hesitaÃ§Ã£o, ligar o salto que o corte deixou, manter nÃºmero e nome
prÃ³prio exatamente como estÃ£o â€” e proÃ­be acrescentar fato, data ou conclusÃ£o que nÃ£o esteja na
fala. O `roteiro.md` Ã© o ponto de revisÃ£o antes do render.

### A voz: `edge` + `pt-BR-FranciscaNeural` (default desde 2026-09-01)

O default era `chatterbox` com a voz `rachel`. O chatterbox Ã© um modelo de clonagem de voz
nascido em inglÃªs e **escorregava para a fonÃ©tica inglesa** no meio de frases em portuguÃªs,
tipicamente ao encontrar nome prÃ³prio ("DroidUp", "Walker 3", "Moya", "startup"): a fala saÃ­a
em outra lÃ­ngua sem nenhum erro no texto â€” o roteiro estava 100% em portuguÃªs. NÃ£o havia
correÃ§Ã£o de texto que resolvesse; era o motor.

ComparaÃ§Ã£o medida no mesmo vÃ­deo, mesmo roteiro:

| | `chatterbox` / `rachel` | `edge` / `pt-BR-FranciscaNeural` |
|---|---|---|
| fala total | 187,4 s | **156,3 s** |
| imagem congelada | 2,5 s em 2 segmentos | **0 s, nenhum** |
| esticÃ£o necessÃ¡rio | atÃ© 8 s por segmento | 1â€“4,5 s |
| saÃ­da final | 210,6 s | **190,3 s** |
| geraÃ§Ã£o de uma frase | 28,2 s | **4,1 s** |

Falar mais rÃ¡pido resolve trÃªs coisas de uma vez: acaba com o congelamento, reduz o esticÃ£o
(que em vÃ­deo com legenda queimada puxa texto de outro trecho) e deixa a geraÃ§Ã£o 7Ã— mais
rÃ¡pida. Para voltar ao anterior, `voz: rachel` + `tts_engine: chatterbox` no `config.yaml`.

### Fala e imagem sempre casadas

A narraÃ§Ã£o quase nunca tem exatamente a duraÃ§Ã£o do trecho, e os dois desencontros davam
defeito visÃ­vel (medidos no vÃ­deo de 2026-09-01):

| SituaÃ§Ã£o | Antes | Agora |
|---|---|---|
| Fala **mais longa** que o trecho | congelava o quadro atÃ© o teto de 6 s e o `atrim` **decepava a frase** no meio (2,5 s perdidos num segmento) | `esticar_por_fala` avanÃ§a o `out` sobre o vÃ­deo original â€” imagem em movimento, contÃ­gua ao corte â€” e sÃ³ congela o que sobrar |
| Fala **muito mais curta** que o trecho | ficava vÃ­deo mudo (um segmento tinha 19,3 s de imagem para 4,9 s de fala: **14,4 s sem ninguÃ©m falando**) | `encolher_por_fala` encurta o segmento atÃ© a fala + 0,6 s de respiro, respeitando `min_segmento_s` |

O esticÃ£o para na entrada do prÃ³ximo segmento (nÃ£o repete material que jÃ¡ vai aparecer) e no
fim do vÃ­deo; sem `metadata.json` (duraÃ§Ã£o desconhecida) ele nÃ£o acontece e o comportamento
cai no freeze antigo, que sempre cabe. Efeito no vÃ­deo de referÃªncia: a imagem parada caiu de
**44 s em 10 segmentos** (5 deles com 5 s ou mais) para **2,5 s em 2 segmentos**, e nenhuma
frase Ã© mais cortada.

Diferente dos outros modos, o N **nÃ£o precisa de classificaÃ§Ã£o visual** (nÃ£o filtra por tipo de
imagem) e usa um teto de congelamento maior: **6 s** em vez de 3 s. PortuguÃªs Ã© mais prolixo que
inglÃªs, entÃ£o a narraÃ§Ã£o costuma passar da duraÃ§Ã£o do trecho â€” em vez de truncar a frase, o
Ãºltimo quadro congela e espera a fala terminar, e o prÃ³ximo trecho entra depois.

## 16. Rodar sem Claude Code e sem Codex (VPS headless)

**Sim â€” o pipeline inteiro roda numa VPS sem nenhum agente de cÃ³digo instalado.** O `otv` Ã©
um CLI Python normal; `claude_cli` Ã© sÃ³ *um dos provedores opcionais* dos slots `visual` e
`pontuacao`, nunca uma dependÃªncia. Codex nÃ£o Ã© usado em lugar nenhum do projeto.

O que a VPS precisa de verdade:

| Item | ObrigatÃ³rio? | ObservaÃ§Ã£o |
|---|---|---|
| Python 3 + `requirements.txt` | sim | `requests`, `pyyaml`, `yt-dlp`, `scenedetect`, `opencv-python`, `mediapipe` |
| `ffmpeg` / `ffprobe` no `PATH` | sim | corte, thumbs, mux |
| `yt-dlp` | sÃ³ se a fonte for URL | arquivo local dispensa |
| `GROQ_API_KEY` | sim (default de transcriÃ§Ã£o) | ou trocar por `whisper_local`/`whisperx` |
| `OPENROUTER_API_KEY` | sim (default de pontuaÃ§Ã£o) | ou trocar por `ollama` |
| GPU | **nÃ£o** | `visual: local` (mediapipe) e a detecÃ§Ã£o de cena rodam em CPU |
| daemon `inemavox` (:8010) | sÃ³ modos B/C/N | TTS â€” sem ele, use `--tts elevenlabs` |
| daemon Ollama (:11434) | sÃ³ se escolher `ollama` | |
| binÃ¡rio `claude` | **nÃ£o** | sÃ³ se escolher `--pontuacao claude_cli` / `--visual claude_cli` |

Config mÃ­nima 100% cloud, sem daemon nenhum e sem GPU:

```yaml
transcricao: groq
visual: local        # modo A; para B/C use glm (tambÃ©m cloud)
pontuacao: glm
tts: elevenlabs      # evita depender do inemavox local
```

Duas ressalvas de VPS pequena:

- **`opencv-python` + `mediapipe` puxam bibliotecas de sistema.** Em imagem slim/headless,
  instale `libgl1` e `libglib2.0-0`, ou troque por `opencv-python-headless`.
- **A detecÃ§Ã£o de cena Ã© CPU-bound**: 130 s para um vÃ­deo de 25 min na mÃ¡quina de referÃªncia.
  Numa VPS de 1 vCPU conte com bem mais, mas ela roda uma vez sÃ³ e Ã© idempotente.

**As keys nÃ£o exigem os caminhos da mÃ¡quina de origem.** Na VPS, use qualquer uma das
fontes portÃ¡veis (seÃ§Ã£o 2 â†’ *Keys*): variÃ¡veis de ambiente no systemd/Docker, um `.env` na
raiz do projeto, `~/.config/otv/.env`, ou `OTV_ENV_FILE` apontando pra onde vocÃª quiser.

```bash
# opÃ§Ã£o 1: ambiente (nada em disco)
GROQ_API_KEY=... OPENROUTER_API_KEY=... python3 otv.py run <url> --modo A

# opÃ§Ã£o 2: .env na raiz do projeto (ignorado pelo git)
printf 'GROQ_API_KEY=...\nOPENROUTER_API_KEY=...\n' > .env
```

## 17. Painel web â€” mandar e ver o que jÃ¡ rodou

```bash
./start.sh                 # sobe o painel; ./start.sh 9000 usa outra porta
./stop.sh                  # para (recusa se houver job rodando)
```

`start.sh` faz a prÃ©-checagem antes de subir â€” `ffmpeg`/`ffprobe` no `PATH`, os mÃ³dulos
Python do `requirements.txt`, `yt-dlp`, e se as keys estÃ£o achando (sÃ³ avisa, nÃ£o bloqueia) â€”
porque falhar aqui Ã© mais barato que falhar no meio de um job jÃ¡ pago. Depois roda com
`setsid`, entÃ£o **o painel sobrevive ao fechar o terminal**, e grava `trabalho/.painel/painel.pid`.

Recusa subir se jÃ¡ houver painel rodando **em qualquer porta**: duas instÃ¢ncias
compartilhariam `trabalho/.painel/` e a segunda sobrescreveria o histÃ³rico de jobs da
primeira. `./stop.sh` **se recusa a parar com job em andamento** (`--forcar` mata o job
junto). OpÃ§Ãµes: `./start.sh --local` prende em `127.0.0.1`; `./start.sh --token` gera um
token e exige `?t=â€¦`.

Sem os scripts, o comando cru Ã© `python3 otv.py painel [--porta 8022] [--host 0.0.0.0]`.

Sobe um servidor da **stdlib** (sem Flask, sem dependÃªncia nova) que escuta em `0.0.0.0`,
entÃ£o qualquer aparelho da LAN abre pelo IP da mÃ¡quina. Ele roda **dentro do projeto**: usa o
mesmo `config.yaml` e a mesma pasta `trabalho/`, e dispara o prÃ³prio `otv.py` como
subprocesso â€” nÃ£o existe caminho paralelo, o painel nÃ£o sabe fazer nada que o CLI nÃ£o faÃ§a.

Na tela:

- **Mandar** â€” cola a URL do YouTube *ou o caminho de um vÃ­deo local*, escolhe o modo (cada
  opÃ§Ã£o traz a descriÃ§Ã£o do que sai: `A â€” fala original`, `N â€” vÃ­deo inteiro, narraÃ§Ã£o
  regravada`, `B â€” sem apresentador`, `C â€” sÃ³ demo e grÃ¡fico`), alvo, slot visual e o
  checkbox **A+** (trocar o apresentador por ilustraÃ§Ã£o gerada), e clica em Rodar. Uma linha
  de dica embaixo do formulÃ¡rio explica o modo escolhido e **avisa quando a combinaÃ§Ã£o vai
  falhar** â€” B, C e A+ precisam de `Visual = glm`, porque a detecÃ§Ã£o `local` nÃ£o sabe o que Ã©
  slide/demo/grÃ¡fico. A fila Ã© **serial, um job por vez** (detecÃ§Ã£o de cena Ã©
  CPU-bound; dois jobs paralelos sÃ³ se atrapalham).
- **Acompanhar** â€” o log ao vivo Ã© o stdout das fases (`[ingest] ok`, `[pontuar] ok`â€¦), gravado
  em `trabalho/.painel/painel-<job>.log`. O job Ã© subprocesso, entÃ£o **continua rodando se vocÃª
  fechar o browser** â€” reabrir reengata no log.
- **Ver o que jÃ¡ rodou** â€” cada rodada de `trabalho/` com thumb, tÃ­tulo, duraÃ§Ã£o original,
  modo, duraÃ§Ã£o do corte, manchete, custo em US$ e os artefatos que existem. Quem jÃ¡ tem
  `output.mp4` ganha player inline (com `Range`, entÃ£o dÃ¡ pra dar seek).
- **Re-cortar de graÃ§a** â€” botÃµes `re-selecionar` (pede outro alvo/modo) e `re-render` em cada
  rodada. Nenhum dos dois chama LLM: US$0, quantas vezes quiser (seÃ§Ã£o 8).
- **Editar o corte** â€” `editar corte` abre a lista de segmentos: dÃ¡ pra desmarcar um segmento
  (sai do corte), ajustar `in`/`out` e trocar a manchete, com o total recalculado na hora.
  `salvar plano` grava o `plan.json` (o anterior vira `plan.bak.json`); `salvar e renderizar`
  jÃ¡ enfileira o render. Ã‰ a ediÃ§Ã£o da seÃ§Ã£o 9 pela tela â€” o servidor sÃ³ aceita `in`, `out`,
  a inclusÃ£o e a manchete: `unidades`, `visual` e `substituir` sÃ£o preservados do plano
  original, entÃ£o um POST malformado nÃ£o inventa campo nem apaga o que o pipeline calculou.
  Remover segmento com narraÃ§Ã£o jÃ¡ gerada invalida o casamento wavâ†”segmento â€” o painel avisa
  e limpa a narraÃ§Ã£o, pedindo `narrar` de novo antes do render nos modos B e N.

### Onde ele fica acessÃ­vel

Ao subir, o painel imprime todos os endereÃ§os IPv4 por onde responde:

```
painel: http://localhost:8022
        http://192.168.1.172:8022
        http://100.70.253.64:8022  (tailscale â€” alcanÃ§Ã¡vel fora da LAN)
```

**Repare no endereÃ§o Tailscale.** Numa mÃ¡quina com tailnet, "LAN" nÃ£o Ã© o limite: o painel
responde para qualquer aparelho do tailnet, inclusive fora de casa. Se o firewall bloquear a
porta (nÃ£o Ã© o caso na mÃ¡quina de referÃªncia), libere com `sudo ufw allow 8022/tcp`; para
prender sÃ³ na mÃ¡quina local, `--host 127.0.0.1`.

### SeguranÃ§a

Sem autenticaÃ§Ã£o por padrÃ£o â€” a premissa Ã© LAN domÃ©stica. Defina `OTV_PAINEL_TOKEN` para
exigir token (`http://â€¦:8022/?t=<token>`):

```bash
OTV_PAINEL_TOKEN=$(openssl rand -hex 16) python3 otv.py painel
```

Duas travas existem porque isto fica exposto na rede, e nÃ£o devem ser removidas:

- **todo comando vai pro `Popen` como lista de argumentos, nunca por shell** â€” sem isso o
  campo de fonte viraria execuÃ§Ã£o de comando arbitrÃ¡rio para qualquer um da LAN;
- **todo id Ã© resolvido e conferido contra a raiz de `trabalho/`** antes de virar caminho de
  arquivo, e as fases disparÃ¡veis sÃ£o uma lista branca (`selecionar`, `render`, `narrar`,
  `abertura`, `substituir`) â€” sem isso `../../etc/passwd` viraria download.

Lembre que **qualquer aparelho que alcance a mÃ¡quina pode disparar um job pago** (~US$0,03
por vÃ­deo) â€” LAN e tailnet incluÃ­dos. O painel avisa no start quando sobe sem token. Se
isso incomodar, defina `OTV_PAINEL_TOKEN` ou use `--host 127.0.0.1`.

---

Projeto adaptado da comunidade INEMA TDS - Marca Arckhe