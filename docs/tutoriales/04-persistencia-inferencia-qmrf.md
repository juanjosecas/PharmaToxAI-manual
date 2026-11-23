# Tutorial 04 — Persistencia, inferencia e informe

Este tutorial continúa el [tutorial 03](03-modelado-validacion.md). Reutiliza sus variables `model`, `X_train`, `X_test`, `y_train`, `y_test`, `train_idx` y `test_idx`. El endpoint del ejercicio anterior es **artificial** y no demuestra utilidad QSAR.

## 1. Guardar el modelo y restaurarlo

~~~python
from pharmatoxai.model_persistence import save_model, load_model

save_model(
    model, X_train, X_test, y_train, y_test,
    output_path="outputs/models",
    model_name="tutorial04_rf",
    feature_names=list(X_train.columns),
    metadata={"dataset": "synthetic", "random_state": 42},
)
loaded = load_model(
    "outputs/models/tutorial04_rf",
    load_data=False, verbose=False,
)
~~~

Verificar paridad entre el modelo original y el recuperado, sin reentrenamiento:

~~~python
import numpy as np

np.testing.assert_allclose(
    model.predict(X_test),
    loaded["model"].predict(X_test),
    rtol=1e-8, atol=1e-10,
)
~~~

## 2. Inferencia con el paquete

~~~python
from pharmatoxai.inference import predict

y_pred = predict(
    "outputs/models/tutorial04_rf",
    X_new=X_test.loc[:, X_train.columns],
    return_proba=False,
    verbose=False,
)
assert len(y_pred) == len(X_test)
~~~

Esta prueba usa **features ya preparadas**. Para usar nuevas moléculas hay que ejecutar la misma curación, featurización y preprocesamiento de entrenamiento y comprobar los parámetros de cada backend. Los modelos serializados se cargan solo si su origen es confiable.

## 3. Generar un registro QMRF

~~~python
from pharmatoxai.generate_qmrf import generate_qmrf_report, export_qmrf

report = generate_qmrf_report(
    model=loaded["model"],
    X_train=X_train, y_train=y_train,
    X_test=X_test, y_test=y_test,
    y_pred=y_pred,
    train_idx=train_idx, test_idx=test_idx,
    model_name="Ejercicio sintético",
    endpoint="Activity (sin significado experimental)",
    task_type="regression",
    extra={
        "dataset": "synthetic_not_chemical",
        "validation": "random holdout",
        "source_commit": "REEMPLAZAR_POR_COMMIT_REAL",
    },
)
export_qmrf(report, "outputs/tutorial04_qmrf.json")
~~~

El JSON resultante es una prueba de ensamblado de información, **no** un QMRF completo ni un certificado de cumplimiento OECD. El software marca algunos principios por presencia de campos; la validación metodológica sigue siendo responsabilidad del protocolo científico.

## 4. Auditoría de salida

Comprobar la igualdad de predicciones antes/después de guardar, el número de observaciones, columnas de entrada, identificación del estimador y registros del split. Registrar `pip freeze` y el commit del código. Los valores de desempeño son ilustrativos y no deben publicarse como resultados toxicológicos.

Este tutorial está contrastado con firmas del código, **sin prueba de integración ejecutada** en un entorno con PharmaToxAI.
