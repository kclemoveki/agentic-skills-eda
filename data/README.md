# `data/` — Demo dataset

Este directorio guarda el **manifest** del dataset usado en la demo, pero **no el CSV mismo**.

## Por qué

El dataset (`messi_all_goals.csv`) está bajo licencia **CC BY-SA 4.0**. Redistribuirlo requeriría que este repo también sea CC BY-SA, lo cual contradice la licencia MIT del código. Por compatibilidad de licencias, linkeamos al dataset original.

## Cómo bajarlo

1. Crear cuenta en [Kaggle](https://www.kaggle.com).
2. Descargar de [`sigmaborov/messi-all-goals-goat`](https://www.kaggle.com/datasets/sigmaborov/messi-all-goals-goat) (790 goles de Messi entre 2004 y 2026, incluido Inter Miami).
3. Guardar el CSV como `data/messi_all_goals.csv` en este repo.

Crédito al autor: [`sigmaborov`](https://www.kaggle.com/sigmaborov) en Kaggle, dataset actualizado al 2026-03-22.

## Verificar integridad

Tras descargar, podés correr:

```bash
sha256sum data/messi_all_goals.csv
```

Comparar contra el `sha256` del `messi_all_goals.csv.manifest.yaml` que está en este directorio. Si no coincide, el dataset cambió desde que se generó este análisis.

## Archivos en este directorio

- `messi_all_goals.csv.manifest.yaml` — manifest con SHA-256, schema y metadata, generado por `/snapshot-data`. Versionable en git.
- `messi_all_goals.csv` — **NO incluido en el repo**, descargar de Kaggle (ver arriba).
