# Tutorial 03 — Entrenamiento y evaluación externa

Ejemplo **sintético**, reproducible y autocontenido para comprobar la API de regresión. No representa moléculas, ensayos ni desempeño QSAR experimental. Para un estudio real, reemplazar la matriz por features moleculares del [Bloque A](../preparacion/index.md) y seleccionar la estrategia de partición en función del uso previsto.

## 1. Dataset artificial

~~~python
import numpy as np
import pandas as pd
from pharmatoxai.splitting import random_split

rng = np.random.default_rng(42)
n = 120
df = pd.DataFrame({
    "descriptor_a": rng.normal(size=n),
    "descriptor_b": rng.uniform(-1, 1, size=n),
    "descriptor_c": rng.normal(size=n),
    "descriptor_d": rng.uniform(0, 1, size=n),
})
df["Activity"] = (
    0.9 * df["descriptor_a"]
    - 0.5 * df["descriptor_b"]
    + 0.2 * df["descriptor_d"]
    + rng.normal(0, 0.25, size=n)
)
train_idx, test_idx, train_df, test_df = random_split(
    df, test_size=0.20, random_state=42
)
cols = ["descriptor_a", "descriptor_b", "descriptor_c", "descriptor_d"]
X_train, y_train = train_df[cols], train_df["Activity"]
X_test, y_test = test_df[cols], test_df["Activity"]
assert set(train_idx).isdisjoint(set(test_idx))
~~~

Aquí el split aleatorio es admisible porque los datos no representan estructuras químicas. Para QSAR, contrastar también estrategias scaffold, agrupamientos de análogos y separación por fecha o por ensayo.

## 2. Entrenar con PharmaToxAI

~~~python
from pharmatoxai.training import train_qsar_regression

result = train_qsar_regression(
    X_train, y_train,
    model_type="random_forest",
    tune=False, run_rfe=False,
    cv=3, random_state=42, n_jobs=1
)
model = result["model"]
print("Ámbito de las métricas internas:", result["performance_scope"])
~~~

Las métricas incluidas en `result["performance"]` describen el ajuste sobre entrenamiento; no utilizarlas como estimación del rendimiento externo.

## 3. Evaluación sobre test

~~~python
from sklearn.metrics import mean_absolute_error, mean_squared_error, r2_score

pred = model.predict(X_test)
metrics = {
    "MAE": mean_absolute_error(y_test, pred),
    "RMSE": float(np.sqrt(mean_squared_error(y_test, pred))),
    "R2": r2_score(y_test, pred),
}
print(metrics)
~~~

No seleccionar nuevos hiperparámetros a partir de estos resultados. Si fuese necesario retocar el modelo, reservar otro holdout verdaderamente independiente.

## 4. Dominio de aplicabilidad

~~~python
from pharmatoxai.evaluation import applicability_domain

flags, thresholds, artifacts = applicability_domain(
    X_train, X_test,
    alpha=0.975,
    k=5,
    n_components=0.95,
    random_state=42,
    verbose=False,
)
print(flags.head())
print(thresholds)
~~~

El dominio aquí solo sirve como demostración de cálculo en descriptores artificiales. No puede interpretarse como cobertura química. Los artefactos contienen escalado, proyección y parámetros para aplicar el procedimiento sin reajustar sobre datos nuevos.

## 5. Registrar identificadores y parámetros

~~~python
from pathlib import Path
import json
import sklearn
import pharmatoxai

Path("outputs").mkdir(exist_ok=True)
summary = {
    "seed": 42,
    "dataset": "synthetic_not_chemical",
    "split": "random",
    "n_train": len(train_idx),
    "n_test": len(test_idx),
    "features": cols,
    "sklearn": sklearn.__version__,
    "package_path": str(pharmatoxai.__file__),
    "metrics": {k: float(v) for k, v in metrics.items()},
}
Path("outputs/tutorial_03_summary.json").write_text(
    json.dumps(summary, indent=2), encoding="utf-8"
)
~~~

En datos experimentales también se deben conservar el commit de PharmaToxAI, unidades, identidad molecular, criterio de curación, IDs y distribución de endpoints por split. Los resultados de este ejemplo **no están verificados por ejecución de integración** en la presente edición del manual.

Continuar con [particiones](../modelado/particiones.md), [entrenamiento](../modelado/entrenamiento-qsar.md), [métricas](../evaluacion/metricas.md) y [dominio de aplicabilidad](../evaluacion/dominio-aplicabilidad.md).
