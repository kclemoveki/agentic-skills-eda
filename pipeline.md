# Pipeline de análisis con skills componibles

Receta para correr un análisis completo y reproducible sobre un dataset, usando 7 skills custom de Claude Code. Cada paso es independiente — podés saltearte los que no apliquen.

## Pre-requisitos

- Claude Code con los 7 skills instalados en `~/.claude/skills/`.
- Para `/execute-notebook`: Python con `papermill` + las deps del notebook (pandas, numpy, etc.) en el entorno activo.
- Working directory en la raíz del proyecto del análisis.

> **Setup vs Pipeline — distinción importante**: crear el entorno virtual e instalar las deps base es **setup one-time**, fuera del pipeline. El pipeline 1-7 asume que el entorno ya está listo. El paso 7 (`/freeze-deps`) **NO crea** el venv: emite un `requirements.txt` que es output del entorno actual, para que un futuro usuario pueda recrearlo con `pip install -r requirements.txt`. El orden conceptual es: (a) crear venv una sola vez, (b) correr pipeline, (c) `/freeze-deps` exporta la receta del environment para reproducibilidad futura.

## Receta

### 1. Snapshot reproducible — `/snapshot-data`

```
/snapshot-data data/<dataset>.csv
```

**Qué hace:** genera `data/<dataset>.csv.manifest.yaml` con SHA-256 + schema fingerprint + metadata. Versionable en git.

**Por qué primero:** fija la versión del dato antes de cualquier inspección. Si re-corremos el pipeline más tarde, sabemos si el dataset cambió.

**Skip si:** el dataset cambia constantemente y el manifest no aporta (raro).

### 2. Triage de calidad — `/quality-report`

```
/quality-report data/<dataset>.csv
```

**Qué hace:** ejecuta Python real, genera `quality_report_<dataset>.md` con luces 🟢🟡🔴 por completeness/consistency/usability + bloqueantes + recomendaciones.

**Skip si:** ya conocés bien el dataset y no necesitás triage.

### 3. Generar notebook — `/analyze-dataset`

```
/analyze-dataset data/<dataset>.csv
```

**Qué hace:** crea `analysis_<dataset>.ipynb` con:
- 9 secciones (Setup → Conclusiones)
- Funciones tipadas con docstrings NumPy
- Parsers defensivos (con try/except y validación)
- Lectura de los artefactos hermanos (quality_report, manifest) si existen
- **Placeholders** en celdas de observación — sin findings inventados

**Skip si:** ya tenés el notebook escrito.

### 4. Ejecutar — `/execute-notebook`

```
/execute-notebook analysis_<dataset>.ipynb
```

**Qué hace:** corre el notebook end-to-end con kernel restart vía papermill. Captura outputs, falla loud al primer error con cell index + traceback.

**Skip si:** preferís ejecutarlo a mano en JupyterLab.

### 5. Anotar findings reales — `/annotate-findings`

```
/annotate-findings analysis_<dataset>.ipynb
```

**Qué hace:** lee los outputs ejecutados y reescribe las celdas placeholder de observaciones con findings reales basados en datos verdaderos. **Nunca inventa números**.

**Skip si:** vas a revisar y escribir las observaciones a mano.

### 6. Reporte ejecutivo — `/eda-report`

```
/eda-report analysis_<dataset>.ipynb
```

**Qué hace:** sintetiza el notebook ejecutado y anotado en un reporte de 1-3 páginas con 7 secciones (Executive Summary → Glosario → Apéndice). Selecciona 3-6 findings de los más importantes, descarta código, extrae figuras como PNGs en `reports/figures/`, y emite comandos pandoc para exportar a PDF o DOCX (este último apto para subir a Google Drive y abrir como Google Docs con estilo corporativo).

**Skip si:** el notebook es para ojos técnicos solamente, sin destinatario stakeholder.

### 7. Pinear dependencias — `/freeze-deps`

```
/freeze-deps analysis_<dataset>.ipynb
```

**Qué hace:** detecta imports usados, resuelve versiones, emite `requirements.txt` (pip) o comando `poetry add` (Poetry). Agrega celda `%watermark` al notebook.

**Skip si:** estás en una etapa exploratoria sin necesidad de pinear todavía.

## Diagrama del flujo

```
              ┌──────────┐
data.csv ────►│snapshot- │ ──► manifest.yaml
              │ data     │          │
              └──────────┘          │ (input opcional)
                    │               ▼
                    │       ┌──────────────┐
                    └──────►│quality-report│ ──► quality_report.md
                            └──────────────┘          │
                                                      │ (input opcional)
                                                      ▼
                                              ┌─────────────┐
                                              │analyze-     │ ──► notebook.ipynb (con placeholders)
                                              │ dataset     │           │
                                              └─────────────┘           ▼
                                                              ┌──────────────────┐
                                                              │execute-notebook  │ ──► notebook.ipynb (ejecutado)
                                                              └──────────────────┘           │
                                                                                             ▼
                                                                                   ┌────────────────────┐
                                                                                   │annotate-findings   │ ──► notebook.ipynb (anotado)
                                                                                   └────────────────────┘    │
                                                                                            ┌────────────────┤
                                                                                            ▼                ▼
                                                                                ┌────────────┐    ┌──────────────┐
                                                                                │eda-report  │    │freeze-deps   │
                                                                                └────────────┘    └──────────────┘
                                                                                      │                 │
                                                                                      ▼                 ▼
                                                                            reports/eda_report.md   requirements.txt
                                                                            reports/figures/*.png
```

## ¿Por qué no un solo comando que automatice todo?

Pregunta común. Respuesta:

1. **Composición > monolito**. Cada skill resuelve una pieza. Podés intercambiarlas (por ejemplo, reemplazar `/quality-report` por una herramienta de tu equipo) sin tocar las demás.
2. **Visibilidad del proceso**. En la demo en vivo se ven los 6 pasos individualmente — eso *es* el punto pedagógico. Un orquestador escondería la composición.
3. **Skills llamando skills no es determinístico**. En Claude Code, un orquestador sería un SKILL.md que dice "ahora invocá X". El modelo lo interpreta y puede saltearse pasos o cambiar el orden. Para una demo en vivo, riesgoso.
4. **Skip selectivo**. Cada paso tiene un "skip si". Un monolito no podría saltarse pasos sin agregar flags.
5. **Debugging**. Si algo falla, cada paso es atómico — sabés exactamente dónde.

Si querés automatización determinística, el camino es **un script externo** (Python/shell con el Claude Agent SDK) que invoca cada skill en orden y maneja errores. Eso es código tradicional, no un skill — y está fuera del alcance de los skills agentic.
