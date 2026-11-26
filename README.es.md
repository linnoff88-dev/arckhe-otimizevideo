# otv — otimizevideo

**🇧🇷 [Português](README.md) · 🇺🇸 [English](README.en.md) · 🇪🇸 [Español](README.es.md)**

CLI que toma un video largo (clase, podcast, conferencia — 20 a 30 min) y devuelve un corte de
**~2 minutos** con el mejor contenido, cortado con precisión a nivel de palabra.

## 📖 Guía de uso

Guía completa (landing + paso a paso): **https://inematds.github.io/otimizevideo/guia/es/**

## 1. Qué hace

`otv` procesa un video mediante un pipeline de fases (transcripción → detección de escenas →
clasificación visual → puntuación → selección → renderizado) y produce un `output.mp4` condensado.
La regla de oro del proyecto, y lo que lo diferencia de «envía el video al modelo y pídele un
corte»: **el modelo nunca elige timestamps**. Solo ve la transcripción dividida en unidades
numeradas por id (`[042] 4.2s talking_head "texto de la unidad"`) y devuelve una puntuación de 0 a 10
por id. Es el código —no el modelo— el que convierte id → tiempo (`ini`/`fim` en segundos, obtenidos
directamente de la transcripción con timestamp por palabra) y hace el corte.

Esto importa por dos motivos: (1) el corte nunca queda en medio de una palabra, porque el
límite de cada unidad ya coincide con un límite de palabra o una pausa real de la transcripción,
nunca con un timestamp inventado por el modelo; y (2) el resultado es reproducible: ejecutar la
misma selección dos veces con el mismo `notas.json` da el mismo `plan.json`, porque la lógica de
selección (mochila por puntuación, cuota por tema, gancho/cierre, cohesión) es determinista y vive
en el código, no en una alucinación del LLM.

## 2. Instalación

```bash
pip install -r requirements.txt
```

El Python del sistema en esta máquina es PEP 668 (entorno «externally managed»); si el comando
anterior falla con `error: externally-managed-environment`, usa:

```bash
pip install --user --break-system-packages -r requirements.txt
```

También deben estar en el `PATH`:

- **ffmpeg** y **ffprobe** (corte, extracción de audio, thumbnails, mux final)
- **yt-dlp** (descargar video desde una URL — solo se necesita si `otv run <url>`, no para un archivo local)

`whisper_local`, `whisperx` y `ollama` (proveedores opcionales, consulta la tabla en la sección 5) incluyen
dependencias pesadas (`openai-whisper`, `whisperx`, `torch`) que **no** están en
`requirements.txt`; instálalas solo si vas a usar esos proveedores.

### Keys

`otv` nunca escribe una clave de API en `config.yaml` ni en ningún archivo de este proyecto:
las lee en **runtime** (`otv/util/keys.py`), desde la primera fuente que tenga la variable:

1. **variable de entorno** — `GROQ_API_KEY=... python3 otv.py run ...`, systemd, Docker
2. el archivo indicado por **`OTV_ENV_FILE`**, si está definido
3. **`.env` en la raíz del proyecto** (está en `.gitignore`)
4. **`~/.config/otv/.env`**
5. los archivos `.env` de la máquina de origen — `~/projetos/openpcbotv2/.env`, `~/projetos/wifi/.env`

Las opciones 1–4 hacen que el proyecto sea portable a otra máquina o VPS sin editar código
(sección 15). Se ignoran las líneas con `#` y se acepta `export NOMBRE=valor`.

Configura `GROQ_API_KEY`, `OPENROUTER_API_KEY` y, si vas a usar ElevenLabs,
`ELEVENLABS_API_KEY`/`ELEVENLABS_VOICE_ID` (más `FAL_KEY` para el modo A+). Este proyecto no
imprime, registra ni guarda ninguna key en el disco.

## 3. Uso rápido

```bash
python3 otv.py run <url-ou-caminho-do-video> --modo A --alvo 120
```

Esto ejecuta todo el pipeline (ingest → transcribir → escenas → puntuar → seleccionar → renderizar)
y escribe el resultado en `~/projetos/output/otimizevideo/<id>/output.mp4` (`<id>` es el
ID del video de YouTube o el nombre normalizado del archivo cuando la fuente es local). El mismo
`output.mp4`, junto con `plan.json`, `notas.json`, `unidades.json` y `custos.json`, también
queda en `trabalho/<id>/` (carpeta de trabajo intermedia).

## 4. Los tres modos

| Modo | Qué genera | Cuándo usarlo | Qué requiere |
|---|---|---|---|
| **A** | Condensado con la **voz original** del presentador (incluido talking head) | Predeterminado — conserva el tono y la voz de quien grabó | Nada aparte de la transcripción; funciona al 100% con `visual: local` |
| **B** | **Sin** el presentador: corta a pantalla/gráfico/diapositiva y **vuelve a grabar la narración** (TTS) encima | Cuando el rostro o la mirada del video original no importan y quieres un corte más «editorial» | Clasificación visual por modelo (las escenas deben estar marcadas como `demo_tela`, `slide` o `grafico` — la detección local solo distingue «hay un rostro» de «no hay») |
| **C** | Solo **demostraciones y gráficos** — el modo más restrictivo | El video es principalmente una demo o un gráfico y quieres solo eso, sin ningún talking head | El mismo requisito que el modo B |

También hay dos modos derivados, documentados al final de este README: **A+** (mantiene la voz
original, pero cambia el video del presentador por una ilustración generada — sección 13) y **N**
(mantiene el video completo en pantalla y vuelve a grabar solo el audio como narración — sección 14).

**B y C dependen de la clasificación visual por modelo.** La detección local (`visual: local`) solo
puede decir «hay un rostro grande» (`talking_head`) o «no» (`outro`): nunca produce
`demo_tela`, `slide` ni `grafico`. Si no pasas `--visual glm`, `--visual gemini` o
`--visual claude_cli` (o ejecutas manualmente `otv cenas <id> --classificar`), `otv run
--modo B` (o `C`) continúa con una advertencia y falla después, en la fase de selección, con un
mensaje claro que indica que no hay unidades con el tipo visual necesario.

## 5. Opciones de `config.yaml`

El `config.yaml` de la raíz define qué proveedor usar de forma predeterminada para cada slot. Todos los slots
se pueden sobrescribir desde la línea de comandos (`--transcricao`, `--visual`, `--pontuacao`, `--tts` en
`otv run`; `--provedor` en `otv transcrever`/`cenas`/`pontuar`/`narrar`), y se puede sustituir todo el archivo
con `--config outro.yaml`.

| Slot | Proveedores | Costo/requisito | Predeterminado |
|---|---|---|---|
| `transcricao` | `groq` | En la nube, requiere `GROQ_API_KEY`. No informa el costo en `$`; ~US$0,04/hora de audio | **default** |
| | `whisper_local` | GPU/CPU local, gratis, pero requiere tener instalado `openai-whisper` (fuera de `requirements.txt`) | |
| | `whisperx` | GPU local (CUDA si está disponible), gratis, requiere tener instalados `whisperx`+`torch` | |
| `visual` | `local` | Gratis, sin key — heurística por área del rostro (mediapipe). Solo distingue `talking_head`/`outro`, nunca `demo_tela`/`slide`/`grafico` | **default** |
| | `glm` (`z-ai/glm-5.3-flash` vía OpenRouter) | Requiere `OPENROUTER_API_KEY`. Barato (~US$0,004 por video de 25 min) | |
| | `gemini` (`google/gemini-2.5-flash-lite` vía OpenRouter) | Requiere `OPENROUTER_API_KEY` | |
| | `claude_cli` (Claude Code headless, `claude -p --model sonnet`) | Se factura a la **suscripción** del usuario — sin API key, pero requiere el binario `claude` en el `PATH` y haber iniciado sesión | |
| `pontuacao` | `glm` (`z-ai/glm-5.3-flash` vía OpenRouter) | Requiere `OPENROUTER_API_KEY`. ~US$0,002 por video de 25 min (1 llamada, `reasoning_effort: low`) | **default** |
| | `gemini` (`google/gemini-2.5-flash-lite` vía OpenRouter) | Requiere `OPENROUTER_API_KEY` | |
| | `ollama` (`qwen3.8:27b`) | Local, gratis, pero requiere el **daemon de Ollama** en ejecución (`http://localhost:11434`) | |
| | `claude_cli` (Claude Code headless) | Se factura a la suscripción — sin API key, requiere `claude` en el `PATH` | |
| `tts` | `inemavox` | Local, gratis, pero requiere el **daemon de inemavox** en ejecución (`http://localhost:8010`). La voz y el motor se obtienen de `config.yaml → voz:` / `tts_engine:` — el predeterminado es **`edge` + `pt-BR-FranciscaNeural`** (consulta la sección 15) | **default** |
| | `elevenlabs` | En la nube, requiere `ELEVENLABS_API_KEY` y `ELEVENLABS_VOICE_ID`; cobra por carácter | |

Los modelos concretos de cada slot de LLM (`glm`, `gemini`, `ollama`, `claude_cli` y el
`turbo` de `whisper_local`) están en `config.yaml → modelos:`. Cambiar el modelo no cambia el
slot, solo aquello a lo que llama ese slot.

## 6. Parámetros de `selecao` (`config.yaml → selecao:`)

| Parámetro | Predeterminado | Efecto práctico |
|---|---|---|
| `alvo_s` | 120 | Duración objetivo, en segundos, del video final (el mismo valor que sobrescribe `--alvo`) |
| `tolerancia` | 0.25 | Rango aceptable alrededor del objetivo: máximo = `alvo_s × 1.25`, mínimo = `alvo_s × 0.75`. La selección usa ese máximo como presupuesto extra para cohesión/gancho/cierre después de que la mochila principal ya alcanzó el `alvo_s` puro |
| `min_segmento_s` | 3 | Se descarta cualquier segmento final con menos que esto (después del corte/snap): evita cortes de fracciones de segundo |
| `min_segmento_ideal_s` | 8 | Meta de «cohesión»: cada segmento intenta incluir vecinos con una puntuación razonable hasta alcanzar este tamaño, para evitar una secuencia de fragmentos diminutos |
| `cota_topico_pct` | 40 | Límite, en % del `alvo_s`, que un solo tema puede ocupar en la mochila principal: evita que el corte se convierta en «solo la parte de la que más se habla» y garantiza diversidad de temas |
| `nota_minima` | 5 | Puntuación mínima (0–10) para que una unidad entre en la mochila principal. El gancho/cierre y las extensiones de cohesión toleran un poco menos (nota_minima − 2 o − 3) para completar el segmento |
| `folga_ms` | 120 | Margen adicional, en milisegundos, que se deja después del final de la última unidad de un segmento al hacer el snap: evita cortar justo en la última sílaba |
| `pausa_fronteira_ms` | 400 | Pausa mínima entre palabras (en la fase de `unidades`) que ya cuenta como límite natural de unidad. Junto con la puntuación (`.`/`!`/`?`) y el final del segmento de la transcripción, es uno de los criterios que cierra una unidad |
| `max_unidade_s` | 12 | Duración máxima de una unidad que se puede cortar: incluso sin pausa ni puntuación (habla continua), el código fuerza un corte en la pausa interna más larga disponible antes de superar este límite |

## 7. Todos los subcomandos

```bash
# pipeline completo
python3 otv.py run <url-ou-caminho> --modo A --alvo 120

# solo descarga/copia el video y extrae el audio, sin procesar nada
python3 otv.py ingest <url-ou-caminho>

# fases individuales sobre una carpeta ya ingerida (<id> es el nombre de la carpeta en trabalho/)
python3 otv.py transcrever <id> --provedor groq
python3 otv.py cenas <id> --classificar --provedor glm
python3 otv.py pontuar <id> --modo A --alvo 120 --provedor glm
python3 otv.py selecionar <id> --modo A --alvo 120
python3 otv.py substituir <id> --provedor fal     # modo A+: sustituye al presentador por una ilustración
python3 otv.py render <id> --rapido
python3 otv.py narrar <id> --provedor inemavox

# inventario de artefactos + resumen del plan de corte
python3 otv.py status <id>

# gasto acumulado por fase, leído desde trabalho/<id>/custos.json
python3 otv.py custo <id>
```

## 8. Ejecutar fase por fase y reutilizar el trabajo

Cada fase escribe su propio JSON en `trabalho/<id>/` (`transcript.json`, `scenes.json`,
`unidades.json`, `notas.json`, `plan.json`, `custos.json`...) y la mayoría son **idempotentes**:
si el artefacto ya existe, la fase no vuelve a ejecutarse; solo devuelve lo que ya está en el disco.
`--forcar` obliga a rehacerla.

**Tres excepciones siempre se vuelven a ejecutar, incluso sin `--forcar`:** `selecionar` (firma el plan de nuevo
cada vez: es el comportamiento correcto para poder ejecutar `otv selecionar <id> --modo B --alvo 90`
repetidas veces hasta encontrar una buena combinación, sin editar código), `render` (no tiene noción de
«ya existe»: siempre vuelve a renderizar a partir del `plan.json` actual, así funciona la sección 9) y
`narrar` (igual: siempre vuelve a generar los wav a partir del `plan.json` actual). Las fases `ingest`,
`transcrever`, `cenas` y `pontuar` son las que realmente se omiten cuando el artefacto ya existe.

Esto ahorra dinero de verdad: si ya se ejecutó la puntuación (`otv pontuar`) y costó
una llamada al LLM, puedes ejecutar `otv selecionar` varias veces con distintos `--modo`/`--alvo`,
o volver a ejecutar `otv render` después de editar el plan a mano (sección 9), sin pagar
ninguna llamada adicional al modelo: repetir `selecionar`/`render`/`narrar` no cuesta nada,
porque no llaman a ningún LLM. Solo vuelve a ejecutar `ingest`/`transcrever`/`cenas`/`pontuar`
con `--forcar` cuando realmente necesites un resultado nuevo de esa fase (por ejemplo, una
transcripción incorrecta o una clasificación visual desactualizada tras cambiar de proveedor).

## 9. Editar `plan.json` manualmente y volver a renderizar

`plan.json` es texto: puedes editar el corte final sin llamar a ningún modelo. Este recurso
le da al usuario control editorial sobre la selección automática.

Un segmento tiene este formato:

```json
{
  "in": 175.1,
  "out": 182.5,
  "unidades": [58, 59, 60],
  "visual": "talking_head",
  "motivo": "explica el resultado del experimento",
  "texto": "The judges said that the 50-year-old problem was essentially solved.",
  "estender_s": 0
}
```

Puedes eliminar un segmento completo de la lista `segmentos`, ajustar `in`/`out` manualmente
(cuidado: `unidades` dejará de coincidir exactamente con el nuevo intervalo, pero el render solo usa
`in`/`out`) o editar el texto de `manchete` en la parte superior del JSON. Después de editar, **no**
vuelvas a ejecutar `otv selecionar` (reescribe `plan.json` desde cero cada vez; consulta la sección 8):
ve directamente al render:

```bash
python3 otv.py render <id>
```

**Cuidado con `otv run` en la misma carpeta.** `otv run <fonte>` sin `--forcar` omite
`ingest`/`transcrever`/`cenas`/`pontuar` si los artefactos ya existen, pero siempre llama a
`selecionar` internamente, que, como explica la sección 8, vuelve a escribir `plan.json` incluso sin
`--forcar`. Si vuelves a ejecutar `otv run` en la misma carpeta después de editar `plan.json` manualmente,
**la edición se pierde en silencio**. Después de editar el plan, el comando correcto es ejecutar
`otv render <id>` por separado (arriba), nunca volver a ejecutar `otv run`.

## 10. Consumo de LLM y costo real

El pipeline llama a un LLM en **cuatro puntos**, y solo uno se ejecuta en el flujo predeterminado (modo A):

| Fase | Llamadas por video de ~25 min | Qué se envía al modelo | Cuándo se ejecuta |
|---|---|---|---|
| `pontuar` | **1** | toda la transcripción dividida en unidades numeradas (texto plano) | siempre |
| `cenas --classificar` | **1 por lote de 20 thumbnails** (`otv/fases/cenas.py`) — ~378 escenas ≈ **19 llamadas**, multimodal, imágenes en base64 | las thumbnails de las escenas | solo modos **B/C** (y `--visual glm/gemini/claude_cli`) |
| `narrar` | **1** | el texto del plan, para reescribirlo como locución | modos **B/C/N** |
| `abertura` | **1** | título + titular + lista corta de escenas | cuando se arma la apertura |

### Cifras medidas (no estimadas)

Video real de ~25 min (`trabalho/dQYKcjvXhIY/custos.json`), slot `glm` =
`z-ai/glm-5.3-flash` vía OpenRouter:

| Fase | tokens de prompt | tokens de completion | Costo | Tiempo |
|---|---|---|---|---|
| `pontuar` | 8.933 | 6.475 | **US$0,0046** | 85 s |
| `classificar` | 83.001 | 10.047 | **US$0,0099** | 195 s |
| `narrar` | 2.116 | 740 | **US$0,0007** | 303 s |
| `abertura` | 784 | 116 | **US$0,0002** | 101 s |
| **Total de LLM (pipeline completo)** | | | **US$0,0154** | |

**La clasificación visual representa el 90% del consumo de tokens**: son imágenes en base64, ~4,4k tokens
por lote de 20 thumbnails. Es el único punto multimodal del proyecto y es exactamente lo que el modo A
no usa.

### Costo por modo

| Modo | LLM que se ejecuta | Costo de LLM | Total con transcripción |
|---|---|---|---|
| **A** (predeterminado, `visual: local`) | solo `pontuar` (+ `abertura`) | **~US$0,005** | **≈ US$0,025** |
| **N** | `pontuar` + `narrar` (+ `abertura`) | ~US$0,006 | ≈ US$0,026 |
| **B / C** | `pontuar` + `classificar` + `narrar` (+ `abertura`) | ~US$0,015 | ≈ US$0,035 |

La transcripción con Groq cuesta ~US$0,02 en todos los modos (Groq no informa el costo en `$`;
estimación de ~US$0,04 por hora de audio). Escenas, detección de rostros, selección, renderizado y TTS
local cuestan R$0.

### Qué reduce o elimina el costo

- **Volver a cortar y renderizar cuesta R$0.** `selecionar`, `render` y `narrar` (el TTS en sí)
  no llaman a ningún LLM de pago; `pontuar` y `cenas` guardan el artefacto y no se repiten sin
  `--forcar` (sección 8).
- `--pontuacao ollama` o `--visual ollama` elimina el costo en `$` (se ejecuta localmente con
  `qwen3.8:27b`, requiere el daemon).
- `--pontuacao claude_cli` / `--visual claude_cli` se factura a la **suscripción** de Claude Code, sin
  API key: el costo se registra como `0.0` en `custos.json`.

### Atención: los reintentos pueden duplicar el consumo

Cada llamada tiene **2 intentos** si la respuesta no llega en JSON válido (`chat_json` en
`otv/provedores/llm.py`) y hasta **3 intentos** HTTP para errores 5xx/429. En el peor de los casos,
una fase consume el doble de lo que muestra la tabla anterior.

Consulta el gasto real de una carpeta ya procesada con:

```bash
python3 otv.py custo <id>
```

## 11. Solución de problemas

Casos reales detectados en este proyecto (el changelog completo está en `FALHAS.md`):

- **`inemavox` (TTS): `GET /api/jobs/{id}/download` devuelve 404.** El endpoint correcto es
  `GET /api/jobs/{id}/audio` — ya está corregido en el código (`otv/provedores/tts.py`), pero si
  estás apuntando a otra versión del daemon de inemavox, verifica el endpoint.
- **El titular desaparecía del video sin mostrar ningún error.** Un `%` suelto en el texto del titular (común en
  «100% de certeza» generado por un LLM) hacía que `drawtext` de ffmpeg interpretara `expansion=normal`
  y fallara en silencio (warning oculto por `-v error`, el proceso termina con código 0 y sin
  titular). Se corrigió con `expansion=none`: si modificas `render.py`, no quites esta flag.
- **`sem_audio_original=True` no silenciaba nada.** `volume=0dB` es una ganancia unitaria
  (no cambia nada); la ganancia realmente nula es `volume=0`. Ya está corregido.
- **El habla continua sin puntuación ni pausas generaba unidades gigantes** (una llegó a 30 s, ~25%
  del presupuesto de un corte de 120 s). El corte forzado por `max_unidade_s` lo resuelve: si
  vuelves a ver unidades grandes, verifica que no se haya eliminado o aumentado demasiado
  `selecao.max_unidade_s` en `config.yaml`.
- **Gemini mediante la API nativa devolvía 503 con los modelos 3.x y 2.5 estaba desactivado para cuentas nuevas.**
  Es un límite de cuenta/infraestructura de Google, no un bug de este proyecto: el slot `gemini` usa OpenRouter, no la API
  nativa, precisamente para no depender de eso.
- **`glm` vía OpenRouter se quedaba bloqueado más de 13 minutos durante una puntuación.** El modelo tiene
  `reasoning` que no se puede desactivar en este endpoint; la solución fue enviar
  `reasoning: {effort: "low"}` + `max_tokens: 20000` (ya está en `config.yaml → openrouter:`); sin
  eso, la llamada puede no responder a tiempo.
- **Groq devolvía `segments: null`** cuando `timestamp_granularities` solo pedía `word`. La
  solución fue pedir `[word, segment]` y manejar `segments or []`: ya está en el código
  (`otv/provedores/transcricao.py`).

## 12. Enlaces

- Especificación de diseño: [`docs/superpowers/specs/2026-08-27-otimizevideo-design.md`](docs/superpowers/specs/2026-08-27-otimizevideo-design.md)
- Plan de implementación (tareas): [`docs/superpowers/plans/2026-08-27-otimizevideo.md`](docs/superpowers/plans/2026-08-27-otimizevideo.md)
- Investigación de proveedores (precios, modelos, comparación): [`docs/pesquisa-provedores-2026-08-27.md`](docs/pesquisa-provedores-2026-08-27.md)
- Changelog de fallas: [`FALHAS.md`](FALHAS.md)


## 13. Modo A+ — sustituir al presentador por una ilustración

`--substituir gerado` (en `run`) o la fase `otv substituir <id>` genera una ilustración 16:9 por
cada segmento marcado como `talking_head` y sustituye **solo el video** de esos fragmentos: el
audio sigue siendo el original, así que la voz no cambia. Solo tiene sentido después de clasificar
el contenido visual (`otv cenas --classificar`), porque el campo `visual` del plan determina qué
se considera presentador.

```bash
python3 otv.py run <url> --modo A --visual glm --substituir gerado
```

Las imágenes quedan en `trabalho/<id>/subst/seg_NN.png` y la ruta relativa se anota en cada
segmento de `plan.json` (`"substituir": "subst/seg_00.png"`). La fase es idempotente: si el PNG
ya existe, se reutiliza sin una nueva llamada de pago; usa `--forcar` para generarlo de nuevo. El render
compone estos fragmentos con `movie=…,zoompan` (efecto Ken Burns lento) con la misma geometría del video, y
`otv status <id>` marca con `→img` lo que ya se sustituyó.

Proveedor de imágenes: `imagem: fal` en `config.yaml` (flux-2-klein vía fal.ai; `FAL_KEY` se lee de los
archivos `.env` habituales). Es el único proveedor implementado: para cambiarlo, hay que escribir otra clase con el mismo
método `gerar(prompt, destino)` en `otv/provedores/imagem.py`.


## 14. Placa del tema (entre la apertura y el video)

Cuando hay una apertura, el render inserta entre ella y el cuerpo una **placa**: fondo oscuro, el
texto del titular y una pausa de 1 s (0,25 s para entrar, 1 s en pantalla, 0,25 s para salir). La
apertura termina a un ritmo rápido — bloques breves con voz encima — y enlazarla directamente con el video
hace que, para quien mira, ambas cosas se conviertan en una sola; la placa es la respiración que separa
«la llamada» del «video». `cartela_s: 0` en `config.yaml` la desactiva.

**El texto no se dibuja con `drawtext`.** `drawtext` de ffmpeg 6.1.1 trunca según el número
de BYTES, no de caracteres: cada letra acentuada se come una letra del final. Esto ocurrió en
producción: el titular «IA e bilionários estêo tentando vencer o envelhecimento» apareció en el
video como «...vencer o envelhecimen». En PT-BR casi todos los titulares llevan tilde, así que era
la regla y no la excepción, y el fallo era silencioso (ffmpeg termina con código 0). El titular y la placa
ahora se rasterizan con PIL (`otv/util/texto.py`) y se insertan mediante `overlay`. Esto también permitió
ajustar los saltos de línea al ancho real de los glifos. Si modificas este overlay, **mantén
`shortest=1`**: la imagen se incluye con `loop=-1` y nunca llega a EOF, así que con `shortest=0` el render
no termina nunca (medido: 18 min de ffmpeg y 51 MB y sigue creciendo, en un video de 5,5 s).


## 15. Modo N — narración sobre todo el contenido

El modo A conserva la voz original, por eso los cortes producen saltos: la persona cambia de tema y
de entonación sin transición. El modo B resuelve la fluidez, pero descarta al presentador y solo deja
diapositivas, demos y gráficos. El **modo N** es el punto medio: mantiene el video completo en pantalla (como el A)
y sustituye el audio por una narración reescrita (como el B).

```bash
python3 otv.py run <url> --modo N
```

La narración **no es contenido nuevo**: el prompt (`prompts/narrar_n.md`) indica traducir y dar fluidez
a lo que ya se dijo — quitar titubeos, conectar el salto que dejó el corte, conservar los números y nombres
propios exactamente como están — y prohíbe agregar datos, fechas o conclusiones que no estén en lo dicho.
`roteiro.md` es el punto de revisión antes del render.

### La voz: `edge` + `pt-BR-FranciscaNeural` (predeterminada desde 2026-09-01)

El valor predeterminado era `chatterbox` con la voz `rachel`. Chatterbox es un modelo de clonación de voz
creado en inglés y **pasaba a la fonética inglesa** a mitad de frases en portugués,
por lo general al encontrar un nombre propio («DroidUp», «Walker 3», «Moya», «startup»): la voz salía
en otro idioma aunque el texto no tuviera ningún error; el guion estaba 100% en portugués. No había
ninguna corrección de texto que lo solucionara: era el motor.

Comparación medida en el mismo video y con el mismo guion:

| | `chatterbox` / `rachel` | `edge` / `pt-BR-FranciscaNeural` |
|---|---|---|
| voz total | 187,4 s | **156,3 s** |
| imagen congelada | 2,5 s en 2 segmentos | **0 s, ninguno** |
| alargamiento necesario | hasta 8 s por segmento | 1–4,5 s |
| salida final | 210,6 s | **190,3 s** |
| generación de una frase | 28,2 s | **4,1 s** |

Hablar más rápido resuelve tres cosas a la vez: elimina el congelamiento, reduce el alargamiento
(que en un video con subtítulos incrustados arrastra texto de otro fragmento) y hace que la generación sea
7× más rápida. Para volver a lo anterior, configura `voz: rachel` + `tts_engine: chatterbox` en `config.yaml`.

### Voz e imagen siempre sincronizadas

La narración casi nunca dura exactamente lo mismo que el fragmento, y ambos desfases provocaban
defectos visibles (medidos en el video del 2026-09-01):

| Situación | Antes | Ahora |
|---|---|---|
| Voz **más larga** que el fragmento | congelaba el cuadro hasta el límite de 6 s y `atrim` **cortaba la frase** a la mitad (se perdían 2,5 s en un segmento) | `esticar_por_fala` adelanta `out` sobre el video original — imagen en movimiento, contigua al corte — y solo congela lo que queda |
| Voz **mucho más corta** que el fragmento | quedaba video en silencio (un segmento tenía 19,3 s de imagen para 4,9 s de voz: **14,4 s sin que nadie hablara**) | `encolher_por_fala` acorta el segmento a la duración de la voz + 0,6 s de respiro, respetando `min_segmento_s` |

El alargamiento se detiene al entrar en el siguiente segmento (no repite material que ya aparecerá) y al
final del video; si no hay `metadata.json` (duración desconocida), no se aplica y se vuelve al comportamiento
anterior de congelar el cuadro, que siempre cabe. Efecto en el video de referencia: la imagen congelada bajó de
**44 s en 10 segmentos** (5 de ellos de 5 s o más) a **2,5 s en 2 segmentos**, y ya no se corta ninguna frase.

A diferencia de los otros modos, el N **no necesita clasificación visual** (no filtra por tipo de
imagen) y usa un límite de congelamiento más alto: **6 s** en lugar de 3 s. El portugués suele ser más prolijo que
el inglés, así que la narración a menudo supera la duración del fragmento; en vez de truncar la frase, se congela
el último cuadro y se espera a que termine la voz; después entra el siguiente fragmento.

## 16. Ejecutar sin Claude Code ni Codex (VPS headless)

**Sí: todo el pipeline funciona en una VPS sin ningún agente de código instalado.** `otv` es
un CLI normal de Python; `claude_cli` es solo *uno de los proveedores opcionales* de los slots
`visual` y `pontuacao`, nunca una dependencia. Codex no se usa en ninguna parte del proyecto.

Lo que la VPS realmente necesita:

| Elemento | ¿Obligatorio? | Observación |
|---|---|---|
| Python 3 + `requirements.txt` | sí | `requests`, `pyyaml`, `yt-dlp`, `scenedetect`, `opencv-python`, `mediapipe` |
| `ffmpeg` / `ffprobe` en el `PATH` | sí | corte, thumbnails, mux |
| `yt-dlp` | solo si la fuente es una URL | no hace falta para archivos locales |
| `GROQ_API_KEY` | sí (predeterminada para transcripción) | o cambiar a `whisper_local`/`whisperx` |
| `OPENROUTER_API_KEY` | sí (predeterminada para puntuación) | o cambiar a `ollama` |
| GPU | **no** | `visual: local` (mediapipe) y la detección de escenas funcionan en CPU |
| daemon de `inemavox` (:8010) | solo modos B/C/N | TTS; sin él, usa `--tts elevenlabs` |
| daemon de Ollama (:11434) | solo si eliges `ollama` | |
| binario `claude` | **no** | solo si eliges `--pontuacao claude_cli` / `--visual claude_cli` |

Configuración mínima 100% en la nube, sin ningún daemon y sin GPU:

```yaml
transcricao: groq
visual: local        # modo A; para B/C usa glm (también en la nube)
pontuacao: glm
tts: elevenlabs      # evita depender de inemavox local
```

Dos advertencias para VPS pequeñas:

- **`opencv-python` + `mediapipe` requieren bibliotecas del sistema.** En una imagen slim/headless,
  instala `libgl1` y `libglib2.0-0`, o cambia a `opencv-python-headless`.
- **La detección de escenas depende de la CPU**: 130 s para un video de 25 min en la máquina de referencia.
  En una VPS de 1 vCPU, calcula bastante más tiempo; aun así, se ejecuta una sola vez y es idempotente.

**Las keys no requieren las rutas de la máquina de origen.** En la VPS, usa cualquiera de las
fuentes portables (sección 2 → *Keys*): variables de entorno en systemd/Docker, un archivo `.env` en la
raíz del proyecto, `~/.config/otv/.env` o `OTV_ENV_FILE` apuntando a donde quieras.

```bash
# opción 1: entorno (nada en el disco)
GROQ_API_KEY=... OPENROUTER_API_KEY=... python3 otv.py run <url> --modo A

# opción 2: .env en la raíz del proyecto (ignorado por git)
printf 'GROQ_API_KEY=...\nOPENROUTER_API_KEY=...\n' > .env
```

## 17. Panel web — enviar y ver lo que ya se ejecutó

```bash
./start.sh                 # inicia el panel; ./start.sh 9000 usa otro puerto
./stop.sh                  # lo detiene (se niega si hay un job en ejecución)
```

`start.sh` realiza una comprobación previa antes de iniciar: `ffmpeg`/`ffprobe` en el `PATH`, los módulos
de Python de `requirements.txt`, `yt-dlp` y que encuentre las keys (solo avisa, no bloquea),
porque es más barato fallar aquí que a mitad de un job ya pagado. Después se ejecuta con
`setsid`, así que **el panel sigue funcionando al cerrar la terminal**, y guarda `trabalho/.painel/painel.pid`.

Se niega a iniciar si ya hay un panel en ejecución **en cualquier puerto**: dos instancias
compartirían `trabalho/.painel/` y la segunda sobrescribiría el historial de jobs de la
primera. `./stop.sh` **se niega a detenerlo si hay un job en curso** (`--forcar` también mata el job).
Opciones: `./start.sh --local` lo fija a `127.0.0.1`; `./start.sh --token` genera un
token y exige `?t=…`.

Sin los scripts, el comando directo es `python3 otv.py painel [--porta 8022] [--host 0.0.0.0]`.

Inicia un servidor de la **stdlib** (sin Flask ni dependencias nuevas) que escucha en `0.0.0.0`,
así que cualquier dispositivo de la LAN puede abrirlo con la IP de la máquina. Funciona **dentro del proyecto**: usa el
mismo `config.yaml` y la misma carpeta `trabalho/`, y ejecuta el propio `otv.py` como
subproceso; no hay una ruta paralela: el panel no puede hacer nada que el CLI no haga.

En la pantalla:

- **Enviar** — pega la URL de YouTube *o la ruta de un video local*, elige el modo (cada
  opción incluye la descripción de lo que genera: `A — voz original`, `N — video completo, narración
  regrabada`, `B — sin presentador`, `C — solo demo y gráfico`), objetivo, slot visual y la
  casilla **A+** (sustituir al presentador por una ilustración generada), y haz clic en Ejecutar. Una línea
  de ayuda debajo del formulario explica el modo elegido y **avisa cuando la combinación va a fallar**:
  B, C y A+ requieren `Visual = glm`, porque la detección `local` no distingue diapositivas/demos/gráficos. La
  cola es **serial, un job a la vez** (la detección de escenas depende de la CPU; dos jobs en paralelo solo se estorban).
- **Seguir** — el registro en vivo es la salida estándar de las fases (`[ingest] ok`, `[pontuar] ok`…), guardada
  en `trabalho/.painel/painel-<job>.log`. El job es un subproceso, así que **sigue ejecutándose aunque cierres el navegador**;
  al volver a abrirlo, se retoma el registro.
- **Ver lo que ya se ejecutó** — cada ejecución de `trabalho/` con thumbnail, título, duración original,
  modo, duración del corte, titular, costo en US$ y los artefactos existentes. Las que ya tienen
  `output.mp4` incluyen un reproductor integrado (con `Range`, así que se puede buscar una posición).
- **Volver a cortar gratis** — botones `re-selecionar` (elige otro objetivo/modo) y `re-render` en cada
  ejecución. Ninguno llama a un LLM: US$0, las veces que quieras (sección 8).
- **Editar el corte** — `editar corte` abre la lista de segmentos: puedes desmarcar un segmento
  (se elimina del corte), ajustar `in`/`out` y cambiar el titular, con el total recalculado al instante.
  `salvar plano` guarda el `plan.json` (el anterior pasa a llamarse `plan.bak.json`); `salvar e renderizar`
  también pone el render en la cola. Es la edición de la sección 9 desde la pantalla: el servidor solo acepta `in`, `out`,
  la inclusión y el titular; `unidades`, `visual` y `substituir` se conservan del plan
  original, así que un POST malformado no inventa campos ni borra lo que calculó el pipeline.
  Eliminar un segmento con una narración ya generada invalida la correspondencia wav↔segmento; el panel avisa
  y borra la narración, y solicita ejecutar `narrar` de nuevo antes del render en los modos B y N.

### Dónde se puede acceder

Al iniciar, el panel imprime todas las direcciones IPv4 en las que responde:

```
panel: http://localhost:8022
       http://192.168.1.172:8022
       http://100.70.253.64:8022  (tailscale — accesible fuera de la LAN)
```

**Fíjate en la dirección de Tailscale.** En una máquina con tailnet, el límite no es la «LAN»:
el panel responde a cualquier dispositivo del tailnet, incluso fuera de casa. Si el firewall bloquea el
puerto (no es el caso en la máquina de referencia), habilítalo con `sudo ufw allow 8022/tcp`; para
limitarlo a la máquina local, usa `--host 127.0.0.1`.

### Seguridad

De forma predeterminada no hay autenticación: se asume una LAN doméstica. Define `OTV_PAINEL_TOKEN` para
exigir un token (`http://…:8022/?t=<token>`):

```bash
OTV_PAINEL_TOKEN=$(openssl rand -hex 16) python3 otv.py painel
```

Hay dos protecciones porque esto queda expuesto en la red y no deben eliminarse:

- **todos los comandos se envían a `Popen` como una lista de argumentos, nunca mediante shell**: sin esto,
  el campo de fuente permitiría ejecutar comandos arbitrarios a cualquier persona de la LAN;
- **todos los ids se resuelven y verifican respecto de la raíz de `trabalho/`** antes de convertirse en rutas de
  archivos, y las fases que se pueden ejecutar forman una lista permitida (`selecionar`, `render`, `narrar`,
  `abertura`, `substituir`): sin esto, `../../etc/passwd` se convertiría en una descarga.

Recuerda que **cualquier dispositivo que pueda acceder a la máquina puede iniciar un job de pago** (~US$0,03
por video), incluidos los de la LAN y el tailnet. El panel avisa al iniciarse si no tiene token. Si eso
te preocupa, define `OTV_PAINEL_TOKEN` o usa `--host 127.0.0.1`.
