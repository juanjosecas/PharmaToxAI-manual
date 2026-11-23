# Evaluación interna y externa

El estimador de desempeño debe corresponder al uso previsto. Un rendimiento calculado sobre las mismas observaciones utilizadas para ajustar el modelo no demuestra predictividad.

## Regresión

~~~python
from sklearn.metrics import mean_absolute_error, mean_squared_error, r2_score
import numpy as np

pred = model.predict(X_test)
metrics = {
    "MAE": mean_absolute_error(y_test, pred),
    "RMSE": np.sqrt(mean_squared_error(y_test, pred)),
    "R2": r2_score(y_test, pred),
}
print(metrics)
~~~

En el paquete, `calculate_metrics(y_true, y_pred)` incluye R², RMSE, MAE, CCC y correlaciones, además de otras métricas. **Precaución:** la clave `Q2` que devuelve esta función para un array de test no sustituye un Q² estimado por predicción fuera de fold. El indicador Q² debe documentarse con su protocolo y referencia.

`evaluate_model(model, X, y)` tiene por defecto `manage_nan="median"`: si ya existe un pipeline de imputación ajustado con train, utilizar `manage_nan=None`, para evitar una imputación independiente sobre test.

## Clasificación

~~~python
from sklearn.metrics import (
    balanced_accuracy_score, f1_score, matthews_corrcoef,
    confusion_matrix, roc_auc_score, average_precision_score
)
pred = model.predict(X_test)
print("Balanced accuracy:", balanced_accuracy_score(y_test, pred))
print("F1:", f1_score(y_test, pred, zero_division=0))
print("MCC:", matthews_corrcoef(y_test, pred))
print("Confusión:", confusion_matrix(y_test, pred))
if hasattr(model, "predict_proba"):
    p = model.predict_proba(X_test)[:, 1]  # solo clasificación binaria, orden de clases comprobado
    print("ROC-AUC:", roc_auc_score(y_test, p))
    print("Average precision:", average_precision_score(y_test, p))
~~~

`calculate_classification_metrics()` de PharmaToxAI agrega métricas; comprobar que `average` y la forma de probabilidades sean compatibles con las clases. El umbral y la regla de calibración deben fijarse en validación, no en test.

## Comparación de modelos

Registrar el baseline, la estrategia de selección, resultados de CV, errores por clase/rango de actividad y diferencias de composición química. Folds repetidos y múltiples semillas permiten evaluar sensibilidad del procedimiento, pero los resultados entre folds no son independientes.

El holdout externo se evalúa **una sola vez** al cerrar la selección de modelo. Si se reoptimiza sobre él deja de ser externo.

## Diagnóstico científico

Inspeccionar residual vs actividad, errores en extremos, falsos negativos relevantes, proporciones por clase, scaffolds poco representados, duplicados entre sets y posibles desplazamientos de distribución. Distinguir datos censurados, sesgo de ensayo y error del modelo.

**Fuentes:** [evaluation.py](https://github.com/juanjosecas/PharmaToxAI/blob/main/src/pharmatoxai/evaluation.py), [validación general](validacion.md).
