# Informes QSAR y QMRF

**Módulo:** `pharmatoxai.generate_qmrf`. El objetivo es reunir metadatos, definición del endpoint, algoritmo, datos, validación, dominio de aplicabilidad e interpretación en un registro auditable.

## Metadatos mínimos

~~~python
from pharmatoxai.generate_qmrf import build_qmrf_metadata, export_qmrf

meta = build_qmrf_metadata(
    model_name="QSAR experimental",
    authors=["Equipo de investigación"],
    endpoint="Actividad (unidad definida)",
)
export_qmrf({"metadata": meta}, "outputs/report_metadata.json")
~~~

El ejemplo crea solamente **metadatos**; no debe presentarse como un informe QMRF completo.

## Compilar un informe a partir del experimento

~~~python
from pharmatoxai.generate_qmrf import generate_qmrf_report, export_qmrf

report = generate_qmrf_report(
    model=model,
    X_train=X_train, y_train=y_train,
    X_test=X_test, y_test=y_test,
    y_pred=model.predict(X_test),
    train_idx=train_idx, test_idx=test_idx,
    model_name="QSAR modelo 01",
    endpoint="Activity (unidad documentada)",
    task_type="regression",
    extra={"source_commit": "REEMPLAZAR_CON_COMMIT_REAL"},
)
export_qmrf(report, "outputs/qmrf.json")
~~~

`generate_qmrf_report` también admite resumen de datos, SHAP y dominio de aplicabilidad. Algunos indicadores `oecd_principles` se completan mediante **presencia de argumentos** y no constituyen una auditoría de cumplimiento científico o regulatorio. En particular, `unambiguous_algorithm=True` no significa que el método haya sido validado por una autoridad.

## Completar manualmente

Registrar protocolo y calidad de las mediciones, unidad, tratamiento de replicados, criterios de curación, train/test split, representaciones moleculares, algoritmo y parámetros, evaluación interna y externa, incertidumbre, AD y explicación mecanística cuando sea defendible.

Para documentación en Word existe `generate_word_report`; necesita `python-docx` y una estructura de datos compatible. No confundir el generador de JSON con un validador automático OECD.

**Fuentes:** [generate_qmrf.py](https://github.com/juanjosecas/PharmaToxAI/blob/main/src/pharmatoxai/generate_qmrf.py) y [documentación QMRF](https://github.com/juanjosecas/PharmaToxAI/blob/main/docs/OECD_QSAR_REPORTS.md).
