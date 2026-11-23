# Inferencia y predicción por lotes

**Módulo:** `pharmatoxai.inference`. La entrada debe reproducir **exactamente** la featurización aplicada durante el entrenamiento: estructura curada, nombres, orden, unidades y transformaciones.

## Desde un modelo guardado

~~~python
from pharmatoxai.inference import predict, batch_predict

# X_new contiene las mismas columnas y representación que el entrenamiento.
pred = predict(
    "outputs/models/baseline_rf",
    X_new=X_new[X_train.columns],
    return_proba=False,
    verbose=False,
)
print(pred)
~~~

Para clasificación, `return_proba=True` puede devolver predicciones y probabilidades. Verificar el orden de clases y que el objeto guardado admite `predict_proba`. La función no convierte universalmente SMILES crudos a los descriptores correctos; generar las características primero.

~~~python
batches = [X_new.iloc[:100], X_new.iloc[100:]]
out = batch_predict(
    "outputs/models/baseline_rf", batches,
    aggregate="concat", verbose=False,
)
~~~

`batch_predict` permite `aggregate="separate"` para preservar particiones de entrada. Comprobar cardinalidad antes y después de featurización: las estructuras inválidas pueden descartarse por módulos de características; conservar identificadores y motivos de rechazo.

## Auditoría antes de inferir

~~~python
expected = list(X_train.columns)
assert not X_new.columns.has_duplicates
missing = set(expected) - set(X_new.columns)
if missing:
    raise ValueError(f"Faltan features: {sorted(missing)}")
X_ready = X_new.loc[:, expected]
assert X_ready.shape[1] == len(expected)
~~~

Esta prueba de esquema es necesaria pero no suficiente: dos Morgan fingerprints con radios distintos pueden compartir nombres idénticos. Verificar también los parámetros de generación y la versión del software.

La inferencia debe informar predicción, unidad, versión del modelo, estado del dominio de aplicabilidad e incertidumbre cuando esté calibrada. No estimar nuevas estadísticas de imputación con moléculas de consulta.

**Fuente:** [inference.py](https://github.com/juanjosecas/PharmaToxAI/blob/main/src/pharmatoxai/inference.py).
