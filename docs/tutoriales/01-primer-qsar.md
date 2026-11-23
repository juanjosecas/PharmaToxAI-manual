# Tutorial 01 — Pipeline QSAR completo: mapa de ejecución

Esta serie utiliza **las mismas moléculas y artefactos** desde la preparación hasta la inferencia y la documentación. Las estructuras SMILES son reales y procesables con RDKit, pero la respuesta `Activity` se **genera artificialmente** a partir de descriptores: ninguna métrica obtenida demuestra capacidad predictiva farmacológica o toxicológica.

## Orden de ejecución

| Etapa | Tutorial | Entradas | Salidas |
|---|---|---|---|
| A. Preparación molecular | [02 — Dataset → matriz](02-preparacion-molecular.md) | SMILES de ejemplo | `outputs/qsar_pipeline/curated.csv`, `features.csv`, `train_raw.csv`, `test_raw.csv`, `split.json` |
| B. Modelado y evaluación | [03 — Train → test → AD](03-modelado-validacion.md) | Artefactos del tutorial 02 | `train_model.csv`, `test_model.csv`, `metrics.json`, `ad_flags.csv`, `preprocessing.joblib` |
| C. Persistencia y trazabilidad | [04 — Guardar → inferir → QMRF](04-persistencia-inferencia-qmrf.md) | Artefactos del tutorial 03 | Modelo serializado, predicciones verificadas, QMRF JSON |

**Directorio de trabajo:** ejecutar todas las celdas desde la raíz de `PharmaToxAI-manual`. Los tutoriales 03 y 04 dependen de los archivos generados anteriormente. Para repetir desde cero, borrar únicamente `outputs/qsar_pipeline`.

~~~bash
python -m pip install -e /ruta/a/PharmaToxAI
python -m pip install matplotlib seaborn joblib scikit-learn rdkit pandas numpy
mkdir -p outputs/qsar_pipeline
~~~

El entorno debe usar una revisión identificada de PharmaToxAI; registrar `git rev-parse HEAD` en el repositorio de la biblioteca. Para evitar confundir la instalación con el repositorio del manual, comprobar:

~~~python
import pharmatoxai, sklearn, rdkit
print("PharmaToxAI:", pharmatoxai.__file__)
print("scikit-learn:", sklearn.__version__)
print("RDKit:", rdkit.__version__)
~~~

## Decisiones metodológicas

1. **Definir endpoint y unidad** antes de importar datos experimentales. El ejemplo usa una respuesta artificial continua sin unidades.
2. **Curar identidades y duplicados** antes del split. Con datos reales conservar la procedencia de cada medición y no agregar observaciones heterogéneas sin justificación.
3. **Separar train/test antes de aprender filtros o imputación**. Las transformaciones se ajustan únicamente sobre train.
4. **Entrenar y seleccionar exclusivamente en train** (CV interna); evaluar test una vez.
5. **Calibrar AD sobre train** y aplicar a test/nuevas moléculas sin reajustarlo.
6. **Guardar modelo y preprocesamiento juntos** y verificar paridad de inferencia.
7. **Documentar incertidumbre y limitaciones**: QMRF describe el modelo; QPRF corresponde a predicciones individuales.

## Extensión a Ames, COM G11 e ICH M7

Una vez ejecutado el flujo de regresión, consultar [Genotoxicidad (COM G11 e ICH M7)](../aplicaciones/genotoxicidad-com-g11-ich-m7.md). Ames exige clasificación, evaluación de sensibilidad/especificidad, validación externa y revisión experta; un segundo clasificador estadístico no sustituye un sistema experto basado en reglas apropiado para ICH M7.

**Estado:** código preparado contra las firmas del repositorio; no se verificó mediante ejecución integral en este cambio.
