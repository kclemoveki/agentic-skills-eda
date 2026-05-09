# Quality Report: messi_all_goals.csv

**Generated:** 2026-05-08
**Rows × Cols:** 790 × 16
**Memory:** 0.7 MB

## 🚦 Overall: 🟢 Dataset listo para EDA, con limpieza menor sugerida.

## Dimensiones

### Completeness: 🟢
- Global missing: 0.0%
- Rows fully populated: 100.0%
- Worst columns: ninguna (todas las columnas están completas)

### Consistency: 🟢
- Exact duplicates: 0.0%
- Candidate keys: ninguna columna alcanza unicidad ≥99% (la más alta es `date` con 518/790 ≈ 65.6%)
- Mixed-type columns: ninguna

### Usability: 🟢
- Usable columns: 13 / 16 (81.3%)
- Constant columns: ninguna
- Near-unique non-keys: ninguna
- High-cardinality categoricals: `date` (518), `opponent` (125), `assist_player` (87)

## Bloqueantes (fix antes de EDA)

Ninguno.

## Recomendaciones de limpieza

1. **Parsear `date` a `datetime`** — actualmente es `str` con 518 valores únicos; sin conversión no se pueden hacer agregados temporales ni ordenar cronológicamente.
2. **Validar normalización de `match_stage` (45 únicos)** — chequear si hay variantes equivalentes ("Round of 16" vs "R16", "Group Stage" vs "Group A/B/...") antes de agrupar.
3. **Agrupar `assist_player` raros** — 87 asistentes únicos en 790 goles; conviene un cubo "Otros" para los que aparecen <N veces antes de visualizar.
4. **Descomponer `match_score` y `score_at_goal`** — son strings ("2-1"); extraer goles propios/rivales como enteros si se quiere modelar diferencia de marcador.
5. **Revisar redundancia entre `venue` / `is_home_goal` y entre `season` / `goal_decade` / `date`** — confirmar si ambas se necesitan o si una es derivable.

## Stats rápidos

| Columna | dtype | Nulos | Únicos |
|---------|-------|-------|--------|
| competition | str | 0% | 13 |
| match_stage | str | 0% | 45 |
| date | str | 0% | 518 |
| venue | str | 0% | 2 |
| club | str | 0% | 4 |
| opponent | str | 0% | 125 |
| match_score | str | 0% | 49 |
| player_position | str | 0% | 5 |
| goal_minute | int64 | 0% | 96 |
| score_at_goal | str | 0% | 39 |
| goal_type | str | 0% | 13 |
| assist_player | str | 0% | 87 |
| season | str | 0% | 22 |
| goal_decade | str | 0% | 3 |
| is_home_goal | bool | 0% | 2 |
| goal_minute_bucket | str | 0% | 7 |
