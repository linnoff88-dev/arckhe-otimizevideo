ï»¿# otv ê¢â¬â otimizevideo

**ê°Å¸â¡Â§ê°Å¸â¡Â· [PortuguêÂªs](README.md) êÂ· ê°Å¸â¡Âºê°Å¸â¡Â¸ [English](README.en.md) êÂ· ê°Å¸â¡Âªê°Å¸â¡Â¸ [EspaêÂ±ol](README.es.md)**

CLI que pega um vêÂ­deo longo (aula, podcast, palestra ê¢â¬â 20 a 30 min) e devolve um corte de
**~2 minutos** com o melhor conteêÂºdo, cortado com precisêÂ£o de palavra.

## ê°Å¸ââ Guia de uso

Guia completo (landing + passo a passo): **https://inematds.github.io/otimizevideo/guia/**

## 1. O que faz

`otv` roda um vêÂ­deo por um pipeline de fases (transcriêÂ§êÂ£o ê¢â â detecêÂ§êÂ£o de cena ê¢â â
classificaêÂ§êÂ£o visual ê¢â â pontuaêÂ§êÂ£o ê¢â â seleêÂ§êÂ£o ê¢â â render) e produz um `output.mp4` condensado.
A regra de ouro do projeto, e o que o diferencia de "manda o vêÂ­deo pro modelo e pede um
corte": **o modelo nunca escolhe timestamps**. Ele sêÂ³ vêÂª a transcriêÂ§êÂ£o fatiada em unidades
numeradas por id (`[042] 4.2s talking_head "texto da unidade"`) e devolve uma nota de 0 a 10
por id. êâ° o cêÂ³digo ê¢â¬â nêÂ£o o modelo ê¢â¬â que converte id ê¢â â tempo (`ini`/`fim` em segundos, vindos
direto da transcriêÂ§êÂ£o com timestamp por palavra) e faz o corte.

Isso importa por dois motivos: (1) o corte nunca cai no meio de uma palavra, porque a
fronteira de cada unidade jêÂ¡ êÂ© uma fronteira de palavra/pausa real da transcriêÂ§êÂ£o, nunca um
timestamp inventado pelo modelo; e (2) o resultado êÂ© reproduzêÂ­vel ê¢â¬â rodar a mesma seleêÂ§êÂ£o
duas vezes com o mesmo `notas.json` dêÂ¡ o mesmo `plan.json`, porque a lêÂ³gica de escolha
(mochila por nota, cota por têÂ³pico, gancho/fecho, coesêÂ£o) êÂ© determinêÂ­stica e vive no cêÂ³digo,
nêÂ£o numa alucinaêÂ§êÂ£o do LLM.

## 2. InstalaêÂ§êÂ£o

```bash
pip install -r requirements.txt
```

O Python do sistema nesta mêÂ¡quina êÂ© PEP 668 (ambiente "externally managed") ê¢â¬â se o comando
acima falhar com `error: externally-managed-environment`, use:

```bash
pip install --user --break-system-packages -r requirements.txt
```

TambêÂ©m precisam estar no `PATH`:

- **ffmpeg** e **ffprobe** (corte, extraêÂ§êÂ£o de êÂ¡udio, thumbnails, mux final)
- **yt-dlp** (baixar vêÂ­deo de URL ê¢â¬â sêÂ³ necessêÂ¡rio se `otv run <url>`, nêÂ£o para arquivo local)

`whisper_local`, `whisperx` e `ollama` (provedores opcionais, ver tabela na seêÂ§êÂ£o 5) trazem
dependêÂªncias pesadas (`openai-whisper`, `whisperx`, `torch`) que **nêÂ£o** estêÂ£o em
`requirements.txt` ê¢â¬â instale-as sêÂ³ se for usar esses provedores.

### Keys

`otv` nunca escreve chave de API em `config.yaml` nem em qualquer arquivo deste projeto ê¢â¬â
elas sêÂ£o lidas em **runtime** (`otv/util/keys.py`), na primeira fonte que tiver a variêÂ¡vel:

1. **variêÂ¡vel de ambiente** ê¢â¬â `GROQ_API_KEY=... python3 otv.py run ...`, systemd, Docker
2. o arquivo apontado por **`OTV_ENV_FILE`**, se definido
3. **`.env` na raiz do projeto** (estêÂ¡ no `.gitignore`)
4. **`~/.config/otv/.env`**
5. os `.env` da mêÂ¡quina de origem ê¢â¬â `~/projetos/openpcbotv2/.env`, `~/projetos/wifi/.env`

As opêÂ§êÂµes 1ê¢â¬â4 tornam o projeto portêÂ¡vel para outra mêÂ¡quina ou VPS sem editar cêÂ³digo
(seêÂ§êÂ£o 15). Linhas com `#` sêÂ£o ignoradas e `export NOME=valor` êÂ© aceito.

Configure `GROQ_API_KEY`, `OPENROUTER_API_KEY` e, se for usar ElevenLabs,
`ELEVENLABS_API_KEY`/`ELEVENLABS_VOICE_ID` (mais `FAL_KEY` para o modo A+). Nenhuma key êÂ©
impressa, logada ou gravada em disco por este projeto.

## 3. Uso rêÂ¡pido

```bash
python3 otv.py run <url-ou-caminho-do-video> --modo A --alvo 120
```

Isso roda o pipeline inteiro (ingest ê¢â â transcrever ê¢â â cenas ê¢â â pontuar ê¢â â selecionar ê¢â â render)
e escreve o resultado em `~/projetos/output/otimizevideo/<id>/output.mp4` (o `<id>` êÂ© o
ID do vêÂ­deo do YouTube, ou o nome do arquivo normalizado, quando a fonte êÂ© local). O mesmo
`output.mp4`, junto com `plan.json`, `notas.json`, `unidades.json` e `custos.json`, tambêÂ©m
fica em `trabalho/<id>/` (pasta de trabalho intermediêÂ¡ria).

## 4. Os trêÂªs modos

| Modo | O que sai | Quando usar | O que exige |
|---|---|---|---|
| **A** | Condensado com a **fala original** do apresentador (talking head incluso) | PadrêÂ£o ê¢â¬â preserva o tom e a voz de quem gravou | Nada alêÂ©m da transcriêÂ§êÂ£o; roda 100% com `visual: local` |
| **B** | **Sem** o apresentador ê¢â¬â corta pra tela/grêÂ¡fico/slide e **regrava a narraêÂ§êÂ£o** (TTS) por cima | Quando o rosto/olho no vêÂ­deo original nêÂ£o importa e vocêÂª quer um corte mais "editorial" | ClassificaêÂ§êÂ£o visual por modelo (as cenas precisam estar marcadas como `demo_tela`, `slide` ou `grafico` ê¢â¬â a detecêÂ§êÂ£o local sêÂ³ distingue "tem rosto" de "nêÂ£o tem") |
| **C** | SêÂ³ **demonstraêÂ§êÂµes e grêÂ¡ficos** ê¢â¬â o modo mais restrito | VêÂ­deo êÂ© majoritariamente demo/grêÂ¡fico e vocêÂª quer sêÂ³ isso, sem qualquer talking head | Mesma exigêÂªncia do modo B |

HêÂ¡ ainda dois modos derivados, documentados no fim deste README: **A+** (mantêÂ©m a fala
original mas troca o vêÂ­deo do apresentador por ilustraêÂ§êÂ£o gerada ê¢â¬â seêÂ§êÂ£o 13) e **N** (mantêÂ©m
o vêÂ­deo inteiro na tela e regrava sêÂ³ o êÂ¡udio como narraêÂ§êÂ£o ê¢â¬â seêÂ§êÂ£o 14).

**B e C dependem de classificaêÂ§êÂ£o visual por modelo.** A detecêÂ§êÂ£o local (`visual: local`) sêÂ³
sabe dizer "tem rosto grande" (`talking_head`) ou "nêÂ£o" (`outro`) ê¢â¬â ela nunca produz
`demo_tela`, `slide` ou `grafico`. Sem passar `--visual glm`, `--visual gemini` ou
`--visual claude_cli` (ou rodar `otv cenas <id> --classificar` manualmente), `otv run
--modo B` (ou `C`) segue em frente com um aviso e falha depois, na fase de seleêÂ§êÂ£o, com uma
mensagem clara dizendo que nêÂ£o hêÂ¡ unidade com o visual necessêÂ¡rio.

## 5. Slots do `config.yaml`

O `config.yaml` na raiz define, por slot, qual provedor usar por padrêÂ£o. Todo slot pode ser
sobreposto por linha de comando (`--transcricao`, `--visual`, `--pontuacao`, `--tts` em
`otv run`; `--provedor` em `otv transcrever`/`cenas`/`pontuar`/`narrar`), e o arquivo inteiro
pode ser trocado com `--config outro.yaml`.

| Slot | Provedores | Custo/exigêÂªncia | Default |
|---|---|---|---|
| `transcricao` | `groq` | Cloud, precisa de `GROQ_API_KEY`. NêÂ£o reporta custo em `$`; ~US$0,04/hora de êÂ¡udio | **default** |
| | `whisper_local` | GPU/CPU local, grêÂ¡tis, mas precisa de `openai-whisper` instalado (fora do `requirements.txt`) | |
| | `whisperx` | GPU local (CUDA se disponêÂ­vel), grêÂ¡tis, precisa de `whisperx`+`torch` instalados | |
| `visual` | `local` | GrêÂ¡tis, sem key ê¢â¬â heurêÂ­stica por êÂ¡rea de rosto (mediapipe). SêÂ³ distingue `talking_head`/`outro`, nunca `demo_tela`/`slide`/`grafico` | **default** |
| | `glm` (`z-ai/glm-5.3-flash` via OpenRouter) | Precisa de `OPENROUTER_API_KEY`. Barato (~US$0,004 por vêÂ­deo de 25 min) | |
| | `gemini` (`google/gemini-2.5-flash-lite` via OpenRouter) | Precisa de `OPENROUTER_API_KEY` | |
| | `claude_cli` (Claude Code headless, `claude -p --model sonnet`) | Sai da **assinatura** do usuêÂ¡rio ê¢â¬â sem API key, mas precisa do binêÂ¡rio `claude` no `PATH` e logado | |
| `pontuacao` | `glm` (`z-ai/glm-5.3-flash` via OpenRouter) | Precisa de `OPENROUTER_API_KEY`. ~US$0,002 por vêÂ­deo de 25 min (1 chamada, `reasoning_effort: low`) | **default** |
| | `gemini` (`google/gemini-2.5-flash-lite` via OpenRouter) | Precisa de `OPENROUTER_API_KEY` | |
| | `ollama` (`qwen3.8:27b`) | Local, grêÂ¡tis, mas precisa de **daemon Ollama** rodando (`http://localhost:11434`) | |
| | `claude_cli` (Claude Code headless) | Sai da assinatura ê¢â¬â sem API key, precisa do `claude` no `PATH` | |
| `tts` | `inemavox` | Local, grêÂ¡tis, mas precisa do **daemon inemavox** rodando (`http://localhost:8010`). Voz e motor saem de `config.yaml ê¢â â voz:` / `tts_engine:` ê¢â¬â o default êÂ© **`edge` + `pt-BR-FranciscaNeural`** (ver seêÂ§êÂ£o 15) | **default** |
| | `elevenlabs` | Cloud, precisa de `ELEVENLABS_API_KEY` e `ELEVENLABS_VOICE_ID`; cobra por caractere | |

Os modelos concretos de cada slot de LLM (`glm`, `gemini`, `ollama`, `claude_cli`, e o
`turbo` do `whisper_local`) ficam em `config.yaml ê¢â â modelos:` ê¢â¬â trocar o modelo nêÂ£o muda o
slot, sêÂ³ o que aquele slot chama.

## 6. ParêÂ¢metros de `selecao` (`config.yaml ê¢â â selecao:`)

| ParêÂ¢metro | Default | Efeito prêÂ¡tico |
|---|---|---|
| `alvo_s` | 120 | DuraêÂ§êÂ£o alvo, em segundos, do vêÂ­deo final (mesmo valor que `--alvo` sobrepêÂµe) |
| `tolerancia` | 0.25 | Faixa aceitêÂ¡vel ao redor do alvo: teto = `alvo_s êâ 1.25`, piso = `alvo_s êâ 0.75`. A seleêÂ§êÂ£o usa esse teto como orêÂ§amento extra para coesêÂ£o/gancho/fecho depois que a mochila principal jêÂ¡ saturou o `alvo_s` puro |
| `min_segmento_s` | 3 | Qualquer segmento final com menos que isso (depois do corte/snap) êÂ© descartado ê¢â¬â evita cortes de fraêÂ§êÂ£o de segundo |
| `min_segmento_ideal_s` | 8 | Meta de "coesêÂ£o": cada segmento tenta puxar vizinhos com nota razoêÂ¡vel atêÂ© atingir esse tamanho, pra nêÂ£o virar uma sequêÂªncia de picadinhos |
| `cota_topico_pct` | 40 | Teto, em % do `alvo_s`, que um êÂºnico têÂ³pico pode ocupar na mochila principal ê¢â¬â evita que o corte vire "sêÂ³ a parte mais falada", garantindo diversidade de assunto |
| `nota_minima` | 5 | Nota mêÂ­nima (0ê¢â¬â10) pra uma unidade entrar na mochila principal. Gancho/fecho e as extensêÂµes de coesêÂ£o toleram um pouco menos (nota_minima ê¢Ëâ 2 ou ê¢Ëâ 3) pra poder completar o segmento |
| `folga_ms` | 120 | Margem extra, em milissegundos, deixada depois do fim da êÂºltima unidade de um segmento ao fazer o snap ê¢â¬â evita cortar em cima da êÂºltima sêÂ­laba |
| `pausa_fronteira_ms` | 400 | Pausa mêÂ­nima entre palavras (na fase de `unidades`) que jêÂ¡ conta como fronteira natural de unidade ê¢â¬â junto com pontuaêÂ§êÂ£o (`.`/`!`/`?`) e fim de segmento da transcriêÂ§êÂ£o, êÂ© um dos critêÂ©rios que fecha uma unidade |
| `max_unidade_s` | 12 | Teto de duraêÂ§êÂ£o de uma unidade cortêÂ¡vel ê¢â¬â mesmo sem pausa nem pontuaêÂ§êÂ£o (fala corrida), o cêÂ³digo forêÂ§a um corte na maior pausa interna disponêÂ­vel antes de estourar esse teto |

## 7. Todos os subcomandos

```bash
# pipeline completo
python3 otv.py run <url-ou-caminho> --modo A --alvo 120

# sêÂ³ baixa/copia o vêÂ­deo e extrai êÂ¡udio, sem processar nada
python3 otv.py ingest <url-ou-caminho>

# fases individuais, sobre uma pasta jêÂ¡ ingerida (o <id> êÂ© o nome da pasta em trabalho/)
python3 otv.py transcrever <id> --provedor groq
python3 otv.py cenas <id> --classificar --provedor glm
python3 otv.py pontuar <id> --modo A --alvo 120 --provedor glm
python3 otv.py selecionar <id> --modo A --alvo 120
python3 otv.py substituir <id> --provedor fal     # modo A+: troca o apresentador por ilustraêÂ§êÂ£o
python3 otv.py render <id> --rapido
python3 otv.py narrar <id> --provedor inemavox

# inventêÂ¡rio de artefatos + resumo do plano de corte
python3 otv.py status <id>

# gasto acumulado por fase, lido de trabalho/<id>/custos.json
python3 otv.py custo <id>
```

## 8. Rodar fase por fase e reaproveitar trabalho

Cada fase escreve seu prêÂ³prio JSON em `trabalho/<id>/` (`transcript.json`, `scenes.json`,
`unidades.json`, `notas.json`, `plan.json`, `custos.json`...) e a maioria êÂ© **idempotente**:
se o artefato jêÂ¡ existe, a fase nêÂ£o roda de novo ê¢â¬â sêÂ³ devolve o que jêÂ¡ estêÂ¡ no disco.
`--forcar` obriga a refazer.

**TrêÂªs exceêÂ§êÂµes sempre refazem, mesmo sem `--forcar`:** `selecionar` (assina o plano de novo
toda vez ê¢â¬â êÂ© o comportamento certo pra poder rodar `otv selecionar <id> --modo B --alvo 90`
repetidas vezes atêÂ© achar a combinaêÂ§êÂ£o boa, sem editar cêÂ³digo), `render` (nêÂ£o tem noêÂ§êÂ£o de
"jêÂ¡ existe" ê¢â¬â sempre re-renderiza a partir do `plan.json` atual, êÂ© assim que a seêÂ§êÂ£o 9
funciona) e `narrar` (idem ê¢â¬â sempre gera os wavs de novo a partir do `plan.json` atual). As
fases `ingest`, `transcrever`, `cenas`, `pontuar` sêÂ£o as que de fato pulam quando o artefato
jêÂ¡ existe.

Isso economiza dinheiro de verdade: se a pontuaêÂ§êÂ£o (`otv pontuar`) jêÂ¡ rodou e custou
uma chamada de LLM, vocêÂª pode rodar `otv selecionar` vêÂ¡rias vezes com `--modo`/`--alvo`
diferentes, ou `otv render` de novo depois de editar o plano êÂ  mêÂ£o (seêÂ§êÂ£o 9), sem pagar
nenhuma chamada de modelo outra vez ê¢â¬â `selecionar`/`render`/`narrar` refazerem sempre nêÂ£o
custa nada, porque nêÂ£o chamam LLM nenhum. SêÂ³ refaêÂ§a `ingest`/`transcrever`/`cenas`/`pontuar`
com `--forcar` quando precisar mesmo de um resultado novo dela (ex.: transcriêÂ§êÂ£o errada,
classificaêÂ§êÂ£o visual desatualizada depois de trocar de provedor).

## 9. Editar `plan.json` na mêÂ£o e re-renderizar

`plan.json` êÂ© texto ê¢â¬â dêÂ¡ pra editar o corte final sem chamar nenhum modelo. êâ° esse recurso
que dêÂ¡ controle editorial ao usuêÂ¡rio sobre a escolha automêÂ¡tica.

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

DêÂ¡ pra remover um segmento inteiro da lista `segmentos`, ajustar `in`/`out` manualmente
(cuidado: `unidades` deixa de bater exatamente com o novo intervalo, mas o render sêÂ³ usa
`in`/`out`), ou editar o texto de `manchete` no topo do JSON. Depois de editar, **nêÂ£o** rode
`otv selecionar` de novo (ele reescreve o `plan.json` do zero, sempre ê¢â¬â ver seêÂ§êÂ£o 8) ê¢â¬â vêÂ¡
direto pro render:

```bash
python3 otv.py render <id>
```

**Cuidado com `otv run` na mesma pasta.** `otv run <fonte>` sem `--forcar` pula
`ingest`/`transcrever`/`cenas`/`pontuar` se os artefatos jêÂ¡ existirem, mas ele sempre chama
`selecionar` internamente ê¢â¬â que, como a seêÂ§êÂ£o 8 explica, reescreve o `plan.json` mesmo sem
`--forcar`. Rodar `otv run` de novo na mesma pasta depois de editar o `plan.json` êÂ  mêÂ£o
**apaga a ediêÂ§êÂ£o em silêÂªncio**. Depois de editar o plano, o comando certo êÂ© `otv render <id>`
isolado (acima), nunca `otv run` de novo.

## 10. Consumo de LLM e custo real

O pipeline chama LLM em **quatro pontos**, e sêÂ³ um deles roda no caminho padrêÂ£o (modo A):

| Fase | Chamadas por vêÂ­deo de ~25 min | O que manda pro modelo | Quando roda |
|---|---|---|---|
| `pontuar` | **1** | a transcriêÂ§êÂ£o inteira fatiada em unidades numeradas (texto puro) | sempre |
| `cenas --classificar` | **1 por lote de 20 thumbnails** (`otv/fases/cenas.py`) ê¢â¬â ~378 cenas ê¢â°Ë **19 chamadas**, multimodal, imagens em base64 | as thumbs das cenas | sêÂ³ modos **B/C** (e `--visual glm/gemini/claude_cli`) |
| `narrar` | **1** | o texto do plano, pra reescrever pra locuêÂ§êÂ£o | modos **B/C/N** |
| `abertura` | **1** | têÂ­tulo + manchete + lista curta de cenas | quando a abertura êÂ© montada |

### NêÂºmeros medidos (nêÂ£o estimados)

VêÂ­deo real de ~25 min (`trabalho/dQYKcjvXhIY/custos.json`), slot `glm` =
`z-ai/glm-5.3-flash` via OpenRouter:

| Fase | prompt tokens | completion tokens | Custo | Tempo |
|---|---|---|---|---|
| `pontuar` | 8.933 | 6.475 | **US$0,0046** | 85 s |
| `classificar` | 83.001 | 10.047 | **US$0,0099** | 195 s |
| `narrar` | 2.116 | 740 | **US$0,0007** | 303 s |
| `abertura` | 784 | 116 | **US$0,0002** | 101 s |
| **Total de LLM (pipeline inteiro)** | | | **US$0,0154** | |

**A classificaêÂ§êÂ£o visual êÂ© 90% do consumo de tokens** ê¢â¬â sêÂ£o imagens em base64, ~4,4k tokens
por lote de 20 thumbs. êâ° o êÂºnico ponto multimodal do projeto, e êÂ© exatamente o que o modo A
nêÂ£o usa.

### Custo por modo

| Modo | LLM que roda | Custo de LLM | Total com transcriêÂ§êÂ£o |
|---|---|---|---|
| **A** (default, `visual: local`) | sêÂ³ `pontuar` (+ `abertura`) | **~US$0,005** | **ê¢â°Ë US$0,025** |
| **N** | `pontuar` + `narrar` (+ `abertura`) | ~US$0,006 | ê¢â°Ë US$0,026 |
| **B / C** | `pontuar` + `classificar` + `narrar` (+ `abertura`) | ~US$0,015 | ê¢â°Ë US$0,035 |

A transcriêÂ§êÂ£o no Groq entra com ~US$0,02 em todos eles (o Groq nêÂ£o reporta custo em `$`;
estimativa de ~US$0,04 por hora de êÂ¡udio). Cenas, detecêÂ§êÂ£o de rosto, seleêÂ§êÂ£o, render e TTS
local sêÂ£o R$0.

### O que zera ou reduz

- **Re-cortes e re-renders sêÂ£o R$0.** `selecionar`, `render` e `narrar` (o TTS em si) nêÂ£o
  chamam LLM pago; `pontuar` e `cenas` gravam o artefato e nêÂ£o repetem sem `--forcar`
  (seêÂ§êÂ£o 8).
- `--pontuacao ollama` ou `--visual ollama` tira o custo em `$` (roda local em
  `qwen3.8:27b`, precisa do daemon).
- `--pontuacao claude_cli` / `--visual claude_cli` sai da **assinatura** do Claude Code, sem
  API key ê¢â¬â o custo êÂ© registrado como `0.0` no `custos.json`.

### AtenêÂ§êÂ£o: retry pode dobrar o consumo

Cada chamada tem **2 tentativas** se a resposta nêÂ£o vier em JSON vêÂ¡lido (`chat_json` em
`otv/provedores/llm.py`) e atêÂ© **3 tentativas** no HTTP para erros 5xx/429. Na pior hipêÂ³tese
uma fase consome o dobro do que a tabela acima mostra.

Veja o gasto real de uma pasta jêÂ¡ processada com:

```bash
python3 otv.py custo <id>
```

## 11. SoluêÂ§êÂ£o de problemas

Casos reais jêÂ¡ encontrados neste projeto (changelog completo em `FALHAS.md`):

- **`inemavox` (TTS): `GET /api/jobs/{id}/download` devolve 404.** O endpoint certo êÂ©
  `GET /api/jobs/{id}/audio` ê¢â¬â jêÂ¡ corrigido no cêÂ³digo (`otv/provedores/tts.py`), mas se vocêÂª
  estiver apontando pra outra versêÂ£o do daemon inemavox, confira o endpoint.
- **Manchete sumindo do vêÂ­deo sem erro nenhum.** Um `%` solto no texto da manchete (comum em
  "100% de certeza" vindo de LLM) fazia o `drawtext` do ffmpeg interpretar `expansion=normal`
  e falhar silenciosamente (warning engolido por `-v error`, processo sai com cêÂ³digo 0 e sem
  manchete). Corrigido com `expansion=none` ê¢â¬â se vocêÂª mexer em `render.py`, nêÂ£o tire essa flag.
- **`sem_audio_original=True` nêÂ£o estava silenciando nada.** `volume=0dB` êÂ© ganho unitêÂ¡rio
  (nêÂ£o muda nada); o ganho real-zero êÂ© `volume=0`. JêÂ¡ corrigido.
- **Fala corrida sem pontuaêÂ§êÂ£o nem pausa gerava unidades gigantes** (uma chegou a 30s, ~25%
  do orêÂ§amento de um corte de 120s). O corte forêÂ§ado por `max_unidade_s` resolve isso ê¢â¬â se
  vocêÂª notar unidades grandes de novo, confira se `selecao.max_unidade_s` nêÂ£o foi removido ou
  aumentado demais no `config.yaml`.
- **Gemini via API nativa devolvendo 503 nos modelos 3.x, e 2.5 desligado pra contas novas.**
  êâ° limite de conta/infra da Google, nêÂ£o bug daqui ê¢â¬â o slot `gemini` usa OpenRouter, nêÂ£o a API
  nativa, exatamente para nêÂ£o depender disso.
- **`glm` via OpenRouter travando mais de 13 minutos numa pontuaêÂ§êÂ£o.** O modelo tem
  `reasoning` que nêÂ£o pode ser desligado nesse endpoint; a correêÂ§êÂ£o foi mandar
  `reasoning: {effort: "low"}` + `max_tokens: 20000` (jêÂ¡ no `config.yaml ê¢â â openrouter:`) ê¢â¬â sem
  isso a chamada pode nunca voltar em tempo hêÂ¡bil.
- **Groq devolvendo `segments: null`** quando `timestamp_granularities` sêÂ³ pedia `word`. A
  correêÂ§êÂ£o foi pedir `[word, segment]` e tratar `segments or []` ê¢â¬â jêÂ¡ no cêÂ³digo
  (`otv/provedores/transcricao.py`).

## 12. Links

- Spec de design: [`docs/superpowers/specs/2026-08-27-otimizevideo-design.md`](docs/superpowers/specs/2026-08-27-otimizevideo-design.md)
- Plano de implementaêÂ§êÂ£o (tasks): [`docs/superpowers/plans/2026-08-27-otimizevideo.md`](docs/superpowers/plans/2026-08-27-otimizevideo.md)
- Pesquisa de provedores (preêÂ§os, modelos, comparaêÂ§êÂ£o): [`docs/pesquisa-provedores-2026-08-27.md`](docs/pesquisa-provedores-2026-08-27.md)
- Changelog de falhas: [`FALHAS.md`](FALHAS.md)


## 13. Modo A+ ê¢â¬â substituir o apresentador por ilustraêÂ§êÂ£o

`--substituir gerado` (no `run`) ou a fase `otv substituir <id>` gera uma ilustraêÂ§êÂ£o 16:9 por
segmento marcado como `talking_head` e troca **sêÂ³ o vêÂ­deo** desses trechos: o êÂ¡udio continua
sendo o original, entêÂ£o a fala nêÂ£o muda em nada. SêÂ³ faz sentido depois que o visual foi
classificado (`otv cenas --classificar`), porque êÂ© o campo `visual` do plano que decide o que
êÂ© apresentador.

```bash
python3 otv.py run <url> --modo A --visual glm --substituir gerado
```

As imagens ficam em `trabalho/<id>/subst/seg_NN.png` e o caminho relativo êÂ© anotado em cada
segmento do `plan.json` (`"substituir": "subst/seg_00.png"`). A fase êÂ© idempotente: um PNG que
jêÂ¡ existe êÂ© reaproveitado sem nova chamada paga ê¢â¬â use `--forcar` para regerar. O render monta
esses trechos com `movie=ê¢â¬Â¦,zoompan` (Ken Burns lento) na mesma geometria do vêÂ­deo, e o
`otv status <id>` marca com `ê¢â âimg` o que jêÂ¡ foi trocado.

Provedor de imagem: `imagem: fal` no `config.yaml` (flux-2-klein via fal.ai, `FAL_KEY` lida dos
`.env` de sempre). êâ° o êÂºnico provedor implementado ê¢â¬â trocar êÂ© escrever outra classe com o mesmo
mêÂ©todo `gerar(prompt, destino)` em `otv/provedores/imagem.py`.


## 14. Cartela de assunto (entre a abertura e o vêÂ­deo)

Quando existe abertura, o render insere entre ela e o corpo uma **cartela**: fundo escuro, o
texto da manchete e uma parada de 1 s (0,25 s pra entrar, 1 s parado, 0,25 s pra sair). A
abertura termina em ritmo alto ê¢â¬â blocos curtos com fala por cima ê¢â¬â e emendar o vêÂ­deo direto
nela faz as duas coisas virarem uma sêÂ³ pra quem assiste; a cartela êÂ© a respirada que separa
"a chamada" do "o vêÂ­deo". `cartela_s: 0` no `config.yaml` desliga.

**O texto nêÂ£o êÂ© desenhado com `drawtext`.** O `drawtext` do ffmpeg 6.1.1 trunca pelo nêÂºmero
de BYTES e nêÂ£o de caracteres: cada letra acentuada come uma letra do fim. Isso saiu em
produêÂ§êÂ£o ê¢â¬â a manchete "IA e bilionêÂ¡rios estêÂ£o tentando vencer o envelhecimento" apareceu no
vêÂ­deo como "...vencer o envelhecimen". Em PT-BR quase toda manchete tem acento, entêÂ£o era a
regra e nêÂ£o a exceêÂ§êÂ£o, e silenciosa (o ffmpeg sai com cêÂ³digo 0). A manchete e a cartela agora
sêÂ£o rasterizadas com PIL (`otv/util/texto.py`) e entram por `overlay` ê¢â¬â o que tambêÂ©m deu
quebra de linha por largura real de glifo. Se vocêÂª mexer nesse overlay, **mantenha
`shortest=1`**: a imagem entra com `loop=-1` e nunca dêÂ¡ EOF, entêÂ£o com `shortest=0` o render
nêÂ£o termina nunca (medido: 18 min de ffmpeg e 51 MB e subindo, num vêÂ­deo de 5,5 s).


## 15. Modo N ê¢â¬â narraêÂ§êÂ£o sobre o conteêÂºdo inteiro

O modo A mantêÂ©m a fala original, e por isso os cortes deixam saltos: a pessoa muda de assunto e
de entonaêÂ§êÂ£o sem transiêÂ§êÂ£o. O modo B resolve a fluêÂªncia, mas descarta o apresentador e sêÂ³ fica
com slide, demo e grêÂ¡fico. O **modo N** êÂ© o meio-termo: mantêÂ©m o vêÂ­deo inteiro na tela (como o A)
e substitui o êÂ¡udio por uma narraêÂ§êÂ£o reescrita (como o B).

```bash
python3 otv.py run <url> --modo N
```

A narraêÂ§êÂ£o **nêÂ£o êÂ© conteêÂºdo novo**: o prompt (`prompts/narrar_n.md`) manda traduzir e dar fluêÂªncia
ao que jêÂ¡ foi dito ê¢â¬â tirar hesitaêÂ§êÂ£o, ligar o salto que o corte deixou, manter nêÂºmero e nome
prêÂ³prio exatamente como estêÂ£o ê¢â¬â e proêÂ­be acrescentar fato, data ou conclusêÂ£o que nêÂ£o esteja na
fala. O `roteiro.md` êÂ© o ponto de revisêÂ£o antes do render.

### A voz: `edge` + `pt-BR-FranciscaNeural` (default desde 2026-09-01)

O default era `chatterbox` com a voz `rachel`. O chatterbox êÂ© um modelo de clonagem de voz
nascido em inglêÂªs e **escorregava para a fonêÂ©tica inglesa** no meio de frases em portuguêÂªs,
tipicamente ao encontrar nome prêÂ³prio ("DroidUp", "Walker 3", "Moya", "startup"): a fala saêÂ­a
em outra lêÂ­ngua sem nenhum erro no texto ê¢â¬â o roteiro estava 100% em portuguêÂªs. NêÂ£o havia
correêÂ§êÂ£o de texto que resolvesse; era o motor.

ComparaêÂ§êÂ£o medida no mesmo vêÂ­deo, mesmo roteiro:

| | `chatterbox` / `rachel` | `edge` / `pt-BR-FranciscaNeural` |
|---|---|---|
| fala total | 187,4 s | **156,3 s** |
| imagem congelada | 2,5 s em 2 segmentos | **0 s, nenhum** |
| esticêÂ£o necessêÂ¡rio | atêÂ© 8 s por segmento | 1ê¢â¬â4,5 s |
| saêÂ­da final | 210,6 s | **190,3 s** |
| geraêÂ§êÂ£o de uma frase | 28,2 s | **4,1 s** |

Falar mais rêÂ¡pido resolve trêÂªs coisas de uma vez: acaba com o congelamento, reduz o esticêÂ£o
(que em vêÂ­deo com legenda queimada puxa texto de outro trecho) e deixa a geraêÂ§êÂ£o 7êâ mais
rêÂ¡pida. Para voltar ao anterior, `voz: rachel` + `tts_engine: chatterbox` no `config.yaml`.

### Fala e imagem sempre casadas

A narraêÂ§êÂ£o quase nunca tem exatamente a duraêÂ§êÂ£o do trecho, e os dois desencontros davam
defeito visêÂ­vel (medidos no vêÂ­deo de 2026-09-01):

| SituaêÂ§êÂ£o | Antes | Agora |
|---|---|---|
| Fala **mais longa** que o trecho | congelava o quadro atêÂ© o teto de 6 s e o `atrim` **decepava a frase** no meio (2,5 s perdidos num segmento) | `esticar_por_fala` avanêÂ§a o `out` sobre o vêÂ­deo original ê¢â¬â imagem em movimento, contêÂ­gua ao corte ê¢â¬â e sêÂ³ congela o que sobrar |
| Fala **muito mais curta** que o trecho | ficava vêÂ­deo mudo (um segmento tinha 19,3 s de imagem para 4,9 s de fala: **14,4 s sem ninguêÂ©m falando**) | `encolher_por_fala` encurta o segmento atêÂ© a fala + 0,6 s de respiro, respeitando `min_segmento_s` |

O esticêÂ£o para na entrada do prêÂ³ximo segmento (nêÂ£o repete material que jêÂ¡ vai aparecer) e no
fim do vêÂ­deo; sem `metadata.json` (duraêÂ§êÂ£o desconhecida) ele nêÂ£o acontece e o comportamento
cai no freeze antigo, que sempre cabe. Efeito no vêÂ­deo de referêÂªncia: a imagem parada caiu de
**44 s em 10 segmentos** (5 deles com 5 s ou mais) para **2,5 s em 2 segmentos**, e nenhuma
frase êÂ© mais cortada.

Diferente dos outros modos, o N **nêÂ£o precisa de classificaêÂ§êÂ£o visual** (nêÂ£o filtra por tipo de
imagem) e usa um teto de congelamento maior: **6 s** em vez de 3 s. PortuguêÂªs êÂ© mais prolixo que
inglêÂªs, entêÂ£o a narraêÂ§êÂ£o costuma passar da duraêÂ§êÂ£o do trecho ê¢â¬â em vez de truncar a frase, o
êÂºltimo quadro congela e espera a fala terminar, e o prêÂ³ximo trecho entra depois.

## 16. Rodar sem Claude Code e sem Codex (VPS headless)

**Sim ê¢â¬â o pipeline inteiro roda numa VPS sem nenhum agente de cêÂ³digo instalado.** O `otv` êÂ©
um CLI Python normal; `claude_cli` êÂ© sêÂ³ *um dos provedores opcionais* dos slots `visual` e
`pontuacao`, nunca uma dependêÂªncia. Codex nêÂ£o êÂ© usado em lugar nenhum do projeto.

O que a VPS precisa de verdade:

| Item | ObrigatêÂ³rio? | ObservaêÂ§êÂ£o |
|---|---|---|
| Python 3 + `requirements.txt` | sim | `requests`, `pyyaml`, `yt-dlp`, `scenedetect`, `opencv-python`, `mediapipe` |
| `ffmpeg` / `ffprobe` no `PATH` | sim | corte, thumbs, mux |
| `yt-dlp` | sêÂ³ se a fonte for URL | arquivo local dispensa |
| `GROQ_API_KEY` | sim (default de transcriêÂ§êÂ£o) | ou trocar por `whisper_local`/`whisperx` |
| `OPENROUTER_API_KEY` | sim (default de pontuaêÂ§êÂ£o) | ou trocar por `ollama` |
| GPU | **nêÂ£o** | `visual: local` (mediapipe) e a detecêÂ§êÂ£o de cena rodam em CPU |
| daemon `inemavox` (:8010) | sêÂ³ modos B/C/N | TTS ê¢â¬â sem ele, use `--tts elevenlabs` |
| daemon Ollama (:11434) | sêÂ³ se escolher `ollama` | |
| binêÂ¡rio `claude` | **nêÂ£o** | sêÂ³ se escolher `--pontuacao claude_cli` / `--visual claude_cli` |

Config mêÂ­nima 100% cloud, sem daemon nenhum e sem GPU:

```yaml
transcricao: groq
visual: local        # modo A; para B/C use glm (tambêÂ©m cloud)
pontuacao: glm
tts: elevenlabs      # evita depender do inemavox local
```

Duas ressalvas de VPS pequena:

- **`opencv-python` + `mediapipe` puxam bibliotecas de sistema.** Em imagem slim/headless,
  instale `libgl1` e `libglib2.0-0`, ou troque por `opencv-python-headless`.
- **A detecêÂ§êÂ£o de cena êÂ© CPU-bound**: 130 s para um vêÂ­deo de 25 min na mêÂ¡quina de referêÂªncia.
  Numa VPS de 1 vCPU conte com bem mais, mas ela roda uma vez sêÂ³ e êÂ© idempotente.

**As keys nêÂ£o exigem os caminhos da mêÂ¡quina de origem.** Na VPS, use qualquer uma das
fontes portêÂ¡veis (seêÂ§êÂ£o 2 ê¢â â *Keys*): variêÂ¡veis de ambiente no systemd/Docker, um `.env` na
raiz do projeto, `~/.config/otv/.env`, ou `OTV_ENV_FILE` apontando pra onde vocêÂª quiser.

```bash
# opêÂ§êÂ£o 1: ambiente (nada em disco)
GROQ_API_KEY=... OPENROUTER_API_KEY=... python3 otv.py run <url> --modo A

# opêÂ§êÂ£o 2: .env na raiz do projeto (ignorado pelo git)
printf 'GROQ_API_KEY=...\nOPENROUTER_API_KEY=...\n' > .env
```

## 17. Painel web ê¢â¬â mandar e ver o que jêÂ¡ rodou

```bash
./start.sh                 # sobe o painel; ./start.sh 9000 usa outra porta
./stop.sh                  # para (recusa se houver job rodando)
```

`start.sh` faz a prêÂ©-checagem antes de subir ê¢â¬â `ffmpeg`/`ffprobe` no `PATH`, os mêÂ³dulos
Python do `requirements.txt`, `yt-dlp`, e se as keys estêÂ£o achando (sêÂ³ avisa, nêÂ£o bloqueia) ê¢â¬â
porque falhar aqui êÂ© mais barato que falhar no meio de um job jêÂ¡ pago. Depois roda com
`setsid`, entêÂ£o **o painel sobrevive ao fechar o terminal**, e grava `trabalho/.painel/painel.pid`.

Recusa subir se jêÂ¡ houver painel rodando **em qualquer porta**: duas instêÂ¢ncias
compartilhariam `trabalho/.painel/` e a segunda sobrescreveria o histêÂ³rico de jobs da
primeira. `./stop.sh` **se recusa a parar com job em andamento** (`--forcar` mata o job
junto). OpêÂ§êÂµes: `./start.sh --local` prende em `127.0.0.1`; `./start.sh --token` gera um
token e exige `?t=ê¢â¬Â¦`.

Sem os scripts, o comando cru êÂ© `python3 otv.py painel [--porta 8022] [--host 0.0.0.0]`.

Sobe um servidor da **stdlib** (sem Flask, sem dependêÂªncia nova) que escuta em `0.0.0.0`,
entêÂ£o qualquer aparelho da LAN abre pelo IP da mêÂ¡quina. Ele roda **dentro do projeto**: usa o
mesmo `config.yaml` e a mesma pasta `trabalho/`, e dispara o prêÂ³prio `otv.py` como
subprocesso ê¢â¬â nêÂ£o existe caminho paralelo, o painel nêÂ£o sabe fazer nada que o CLI nêÂ£o faêÂ§a.

Na tela:

- **Mandar** ê¢â¬â cola a URL do YouTube *ou o caminho de um vêÂ­deo local*, escolhe o modo (cada
  opêÂ§êÂ£o traz a descriêÂ§êÂ£o do que sai: `A ê¢â¬â fala original`, `N ê¢â¬â vêÂ­deo inteiro, narraêÂ§êÂ£o
  regravada`, `B ê¢â¬â sem apresentador`, `C ê¢â¬â sêÂ³ demo e grêÂ¡fico`), alvo, slot visual e o
  checkbox **A+** (trocar o apresentador por ilustraêÂ§êÂ£o gerada), e clica em Rodar. Uma linha
  de dica embaixo do formulêÂ¡rio explica o modo escolhido e **avisa quando a combinaêÂ§êÂ£o vai
  falhar** ê¢â¬â B, C e A+ precisam de `Visual = glm`, porque a detecêÂ§êÂ£o `local` nêÂ£o sabe o que êÂ©
  slide/demo/grêÂ¡fico. A fila êÂ© **serial, um job por vez** (detecêÂ§êÂ£o de cena êÂ©
  CPU-bound; dois jobs paralelos sêÂ³ se atrapalham).
- **Acompanhar** ê¢â¬â o log ao vivo êÂ© o stdout das fases (`[ingest] ok`, `[pontuar] ok`ê¢â¬Â¦), gravado
  em `trabalho/.painel/painel-<job>.log`. O job êÂ© subprocesso, entêÂ£o **continua rodando se vocêÂª
  fechar o browser** ê¢â¬â reabrir reengata no log.
- **Ver o que jêÂ¡ rodou** ê¢â¬â cada rodada de `trabalho/` com thumb, têÂ­tulo, duraêÂ§êÂ£o original,
  modo, duraêÂ§êÂ£o do corte, manchete, custo em US$ e os artefatos que existem. Quem jêÂ¡ tem
  `output.mp4` ganha player inline (com `Range`, entêÂ£o dêÂ¡ pra dar seek).
- **Re-cortar de graêÂ§a** ê¢â¬â botêÂµes `re-selecionar` (pede outro alvo/modo) e `re-render` em cada
  rodada. Nenhum dos dois chama LLM: US$0, quantas vezes quiser (seêÂ§êÂ£o 8).
- **Editar o corte** ê¢â¬â `editar corte` abre a lista de segmentos: dêÂ¡ pra desmarcar um segmento
  (sai do corte), ajustar `in`/`out` e trocar a manchete, com o total recalculado na hora.
  `salvar plano` grava o `plan.json` (o anterior vira `plan.bak.json`); `salvar e renderizar`
  jêÂ¡ enfileira o render. êâ° a ediêÂ§êÂ£o da seêÂ§êÂ£o 9 pela tela ê¢â¬â o servidor sêÂ³ aceita `in`, `out`,
  a inclusêÂ£o e a manchete: `unidades`, `visual` e `substituir` sêÂ£o preservados do plano
  original, entêÂ£o um POST malformado nêÂ£o inventa campo nem apaga o que o pipeline calculou.
  Remover segmento com narraêÂ§êÂ£o jêÂ¡ gerada invalida o casamento wavê¢â âsegmento ê¢â¬â o painel avisa
  e limpa a narraêÂ§êÂ£o, pedindo `narrar` de novo antes do render nos modos B e N.

### Onde ele fica acessêÂ­vel

Ao subir, o painel imprime todos os endereêÂ§os IPv4 por onde responde:

```
painel: http://localhost:8022
        http://192.168.1.172:8022
        http://100.70.253.64:8022  (tailscale ê¢â¬â alcanêÂ§êÂ¡vel fora da LAN)
```

**Repare no endereêÂ§o Tailscale.** Numa mêÂ¡quina com tailnet, "LAN" nêÂ£o êÂ© o limite: o painel
responde para qualquer aparelho do tailnet, inclusive fora de casa. Se o firewall bloquear a
porta (nêÂ£o êÂ© o caso na mêÂ¡quina de referêÂªncia), libere com `sudo ufw allow 8022/tcp`; para
prender sêÂ³ na mêÂ¡quina local, `--host 127.0.0.1`.

### SeguranêÂ§a

Sem autenticaêÂ§êÂ£o por padrêÂ£o ê¢â¬â a premissa êÂ© LAN domêÂ©stica. Defina `OTV_PAINEL_TOKEN` para
exigir token (`http://ê¢â¬Â¦:8022/?t=<token>`):

```bash
OTV_PAINEL_TOKEN=$(openssl rand -hex 16) python3 otv.py painel
```

Duas travas existem porque isto fica exposto na rede, e nêÂ£o devem ser removidas:

- **todo comando vai pro `Popen` como lista de argumentos, nunca por shell** ê¢â¬â sem isso o
  campo de fonte viraria execuêÂ§êÂ£o de comando arbitrêÂ¡rio para qualquer um da LAN;
- **todo id êÂ© resolvido e conferido contra a raiz de `trabalho/`** antes de virar caminho de
  arquivo, e as fases disparêÂ¡veis sêÂ£o uma lista branca (`selecionar`, `render`, `narrar`,
  `abertura`, `substituir`) ê¢â¬â sem isso `../../etc/passwd` viraria download.

Lembre que **qualquer aparelho que alcance a mêÂ¡quina pode disparar um job pago** (~US$0,03
por vêÂ­deo) ê¢â¬â LAN e tailnet incluêÂ­dos. O painel avisa no start quando sobe sem token. Se
isso incomodar, defina `OTV_PAINEL_TOKEN` ou use `--host 127.0.0.1`.

---

Projeto  - ARCKHÉe