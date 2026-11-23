# Fuentes técnicas del repositorio

El manual se desarrolla por capítulos pequeños y verificables. La documentación existente del proyecto, los notebooks y el código son fuentes de partida, pero **un ejemplo no se considera validado solo porque figure en README o en un notebook**. Antes de publicarlo como receta reproducible, contrastar firma/API con la revisión del módulo, ejecutar una prueba mínima y registrar dependencias.

**Repositorio fuente:** [PharmaToxAI](https://github.com/juanjosecas/PharmaToxAI) (acceso restringido actualmente).

| Tema para desarrollar | Referencia técnica en el repositorio de código | Evidencia práctica |
| --- | --- | --- |
| Carga y fuentes de datos | `docs/data_io.md`, `src/pharmatoxai/data_io.py`, `database_download.py` | Notebook de EDA, notebooks ChEMBL |
| EDA | `docs/eda.md`, `src/pharmatoxai/eda.py` | `notebooks/01_Exploratory Data Analysis.ipynb` |
| Curación | `docs/curation.md`, `src/pharmatoxai/curation.py` | `scripts/cli_curate.py` |
| Representaciones moleculares | `docs/featurization.md`, `src/pharmatoxai/featurization.py` | Notebook 02; `scripts/cli_featurize.py` |
| Limpieza y particiones | `docs/feature_cleaning.md`, `docs/splitting.md` | Notebook 02; tests de leakage |
| Entrenamiento | `docs/training.md`, `src/pharmatoxai/training.py` | Notebooks 03, regresión y clasificación |
| Deep learning | `src/pharmatoxai/nn_models.py`, `nn_training.py` | Notebooks 03 de DeepLearning |
| Evaluación y AD | `docs/evaluation.md`, `src/pharmatoxai/evaluation.py` | Notebook 04 |
| Persistencia e inferencia | `docs/model_persistence.md`, `docs/inference.md` | Notebook 04; scripts de inferencia |
| Visualización | `src/pharmatoxai/visualization.py` | Notebooks 05 y 06 |
| Clasificación multilabel | `docs/multilabel.md`, `src/pharmatoxai/multilabel.py` | Tests y ejemplos pertinentes |
| QMRF y documentación | `docs/OECD_QSAR_REPORTS.md`, `generate_qmrf.py` | `notebooks/OECD_QMRF_helper.ipynb` |

La referencia de firmas está en `docs/API_REFERENCE.md` del código. Puede desactualizarse respecto de `main`: contrastar siempre con el módulo correspondiente.

## Convención para nuevas entregas

Cada módulo del manual debería incluir:

1. Objetivo, entrada/salida y supuestos.
2. Procedimiento mínimo ejecutable usando la API comprobada.
3. Validaciones (incluido leakage y separación train/test cuando corresponda).
4. Limitaciones científicas y computacionales.
5. Referencias a la documentación, código y notebooks de origen.
6. Versión o commit utilizado para verificar el procedimiento.

Los archivos del manual permanecen independientes entre sí para poder editarlos, revisarlos y publicarlos por partes. Los capítulos pendientes no se presentarán como documentación finalizada.
