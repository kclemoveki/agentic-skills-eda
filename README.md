# agentic-skills-eda

Siete [Claude Code skills](https://code.claude.com/docs/en/skills) que componen un pipeline de EDA reproducible y auditable. Material de la charla **"Agentic Skills: Ponele Piloto Automático a tu EDA"** — DataAr Córdoba 2026.

## Qué resuelven

Un análisis exploratorio típico tiene rituales que cada Data Scientist re-escribe a mano: imports, chequeos de calidad, parsers defensivos, hipótesis estadísticas, formato de notebook. Estos siete skills capturan esos rituales como instrucciones reutilizables que Claude Code sigue al invocarlos.

```
/snapshot-data     → manifest YAML reproducible (SHA-256 + schema fingerprint)
/quality-report    → triage 🟢🟡🔴 por completeness, consistency, usability
/analyze-dataset   → notebook EDA con 9 secciones, parsers defensivos, placeholders honestos
/execute-notebook  → corre el notebook con papermill, falla loud al primer error
/annotate-findings → reescribe los placeholders con findings reales basados en outputs ejecutados
/eda-report        → sintetiza el notebook en un reporte ejecutivo de 1-3 páginas (markdown)
/freeze-deps       → pinea versiones (requirements.txt o poetry add) + watermark cell
```

Cada uno es atómico, reemplazable y composable. Ver [`pipeline.md`](pipeline.md) para la receta.

## Instalación

### 1. Clonar el repo

```bash
git clone git@github.com:kclemoveki/agentic-skills-eda.git
cd agentic-skills-eda
```

### 2. Copiar los skills al directorio de Claude Code

```bash
cp -r skills/* ~/.claude/skills/
```

Claude Code los descubre automáticamente. Verificá tipeando `/help` en una sesión nueva: deberías ver los siete listados.

### 3. Crear un entorno Python con las deps

**Requisito**: Python **3.11 o superior** (`pandas 3.x`, `numpy 2.x` y `matplotlib 3.10+` no soportan Python ≤ 3.10).

Usá la herramienta que prefieras (pip, uv, conda, poetry, etc.). Ejemplo con venv estándar:

```bash
python3.11 -m venv .venv
source .venv/bin/activate
pip install --upgrade pip
pip install -r requirements.txt
pip install papermill==2.7.0
```

**Por qué `papermill` separado:** `requirements.txt` lo genera `/freeze-deps`, que detecta los imports **dentro del notebook**. Papermill es la herramienta que **ejecuta** el notebook (la usa `/execute-notebook`), no algo que el notebook importe. Por eso queda invisible al introspeccionar imports y hay que instalarlo aparte.

Si tu sistema no tiene Python 3.11 disponible:
- macOS: `brew install python@3.11`
- Ubuntu/Debian: `sudo apt install python3.11 python3.11-venv`
- Otras distros: ver [python.org/downloads](https://www.python.org/downloads/)

Las versiones del `requirements.txt` son las exactas que produjo `/freeze-deps` en la corrida de demo. Si querés menos estrictez, podés relajar los pins (`==` → `>=`) y probar tu suerte con versiones más viejas, pero no está testeado.

### 4. Bajar el dataset (opcional, si querés reproducir la demo)

Ver [`data/README.md`](data/README.md) — el dataset está en Kaggle bajo CC BY-SA y no se redistribuye en este repo.

## Cómo correr el pipeline

Una vez con el venv activo y el dataset en `data/messi_all_goals.csv`:

```
/snapshot-data data/messi_all_goals.csv
/quality-report data/messi_all_goals.csv
/analyze-dataset data/messi_all_goals.csv
/execute-notebook analysis_messi_all_goals.ipynb
/annotate-findings analysis_messi_all_goals.ipynb
/eda-report analysis_messi_all_goals.ipynb
/freeze-deps analysis_messi_all_goals.ipynb
```

Detalle paso a paso, con qué hace cada skill y cuándo saltearlo, en [`pipeline.md`](pipeline.md).

## Estructura del repo

```
agentic-skills-eda/
├── skills/                  ← los 7 SKILL.md (el "producto")
├── data/                    ← manifest del dataset + pointer al Kaggle
├── notebooks/               ← demo del pipeline ejecutado y anotado
├── reports/                 ← demo del eda-report (markdown + figuras)
├── diagrams/                ← Mermaid + ASCII para acompañar la charla
├── pipeline.md              ← receta del pipeline + Q&A "¿por qué no automatizar?"
├── requirements.txt         ← deps del runtime (output de /freeze-deps)
├── quality_report_*.md      ← demo del quality report
└── README.md                ← este archivo
```

## Próximos pasos

Lo que está agendado pero todavía no implementado:

- **Hardening de seguridad explícita en SKILL.md**: probamos prompt injection contra `/analyze-dataset`; el modelo ya se defiende vía higiene de datos. Hay margen para codificar reglas defensivas más explícitas.
- **`/eda-report` con templates corporativos opcionales**: el skill ya soporta `templates/reference.docx` para output styled vía pandoc. Falta documentar cómo el usuario provee su propio template.
- **Composición con orquestador externo**: `pipeline.md` es manual deliberadamente. Un script con el Claude Agent SDK que recorra los siete skills sería el siguiente paso natural si querés determinismo de pipeline.

## Licencia

[MIT](LICENSE).

Las opiniones expresadas en la charla y este repo son personales del autor, no de su empleador.
