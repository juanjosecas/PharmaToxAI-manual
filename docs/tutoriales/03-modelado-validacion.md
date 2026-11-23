# Tutorial 03 — Pipeline B: limpieza → entrenamiento → test → AD

**Requiere ejecutar [Tutorial 02](02-preparacion-molecular.md).** Usa exactamente las particiones allí guardadas. El endpoint sigue siendo artificial y las métricas no representan desempeño QSAR real.

## 1. Cargar particiones congeladas

~~~python
from pathlib import Path
import json
import joblib
import numpy as np
import pandas as pd
import seaborn as sns
import matplotlib.pyplot as plt

OUT = Path("outputs/qsar_pipeline")
train_df = pd.read_csv(OUT / "train_raw.csv")
test_df = pd.read_csv(OUT / "test_raw.csv")
split_info = json.loads((OUT / "split.json").read_text(encoding="utf-8"))
assert len(train_df) == len(split_info["train_index"])
assert len(test_df) == len(split_info["test_index"])
excluded = {"ID", "SMILES", "SMILES_clean", "Activity"}
feature_cols = [c for c in train_df.select_dtypes(include="number")
                if c not in excluded]
assert feature_cols
X_train_raw = train_df[feature_cols].replace([np.inf, -np.inf], np.nan)
X_test_raw = test_df[feature_cols].replace([np.inf, -np.inf], np.nan)
y_train = train_df["Activity"]
y_test = test_df["Activity"]
~~~

## 2. Ajustar limpieza solo con train

~~~python
from pharmatoxai.feature_cleaning import (
    remove_nan_inf, impute_median_train, remove_low_variance
)

train_filtered = remove_nan_inf(X_train_raw, max_nan_frac=0.20)
selected_before_variance = train_filtered.columns.tolist()
test_filtered = X_test_raw[selected_before_variance]
X_train, X_test = impute_median_train(train_filtered, test_filtered)
medians = train_filtered.median().fillna(0.0)
X_train = X_train.fillna(0.0)
X_test = X_test.fillna(0.0)
keep = remove_low_variance(X_train, threshold=0.0).columns.tolist()
X_train = X_train[keep]
X_test = X_test[keep]
assert keep and X_train.columns.equals(X_test.columns)
assert np.isfinite(X_train.to_numpy()).all()
assert np.isfinite(X_test.to_numpy()).all()

# Conservar reglas de transformación para inferencia futura.
preprocessing = {
    "raw_feature_names": feature_cols,
    "filtered_columns": selected_before_variance,
    "medians": medians,
    "selected_columns": keep,
}
joblib.dump(preprocessing, OUT / "preprocessing.joblib")
X_train.to_csv(OUT / "train_model.csv", index=False)
X_test.to_csv(OUT / "test_model.csv", index=False)
print(X_train.shape, X_test.shape)
~~~

Las columnas eliminadas, imputaciones y umbrales se determinan sin inspeccionar el test. Si una columna está completamente ausente en train, el ejemplo la rellena con cero después de imputación; en datos reales conviene excluirla o justificar su tratamiento.

## 3. Entrenamiento y validación interna

~~~python
from pharmatoxai.training import train_qsar_regression

result = train_qsar_regression(
    X_train, y_train, model_type="random_forest",
    tune=False, run_rfe=False, cv=3,
    random_state=42, n_jobs=1,
)
model = result["model"]
print("Métricas de entrenamiento:", result.get("performance"))
print("Ámbito:", result.get("performance_scope"))
~~~

El modelo se entrena **solo con train**. Si se comparan algoritmos o hiperparámetros, hacerlo mediante CV dentro de train. Las métricas de resustitución no son validación externa.

## 4. Evaluación sobre test retenido

~~~python
from sklearn.metrics import mean_absolute_error, mean_squared_error, r2_score

pred_train = np.asarray(model.predict(X_train)).ravel()
pred_test = np.asarray(model.predict(X_test)).ravel()
metrics = {
    "MAE": float(mean_absolute_error(y_test, pred_test)),
    "RMSE": float(np.sqrt(mean_squared_error(y_test, pred_test))),
    "R2": float(r2_score(y_test, pred_test)),
}
print(metrics)
(OUT / "metrics.json").write_text(
    json.dumps({"test": metrics, "dataset": "artificial",
                "model": "random_forest", "seed": 42}, indent=2),
    encoding="utf-8",
)
pd.DataFrame({"observado": y_test.to_numpy(),
              "predicho": pred_test}).to_csv(
    OUT / "test_predictions.csv", index=False
)
~~~

~~~python
from pharmatoxai.visualization import plot_residuals_analysis

fig, axes = plot_residuals_analysis(
    y_train.to_numpy(), pred_train,
    y_test.to_numpy(), pred_test,
)
plt.show()

plot_df = pd.DataFrame({"Observado": y_test, "Predicho": pred_test})
sns.scatterplot(data=plot_df, x="Observado", y="Predicho")
limits = [min(plot_df.min()), max(plot_df.max())]
plt.plot(limits, limits, "k--")
plt.title("Test: observado frente a predicho")
plt.tight_layout()
plt.show()
~~~

## 5. Dominio de aplicabilidad

~~~python
from pharmatoxai.evaluation import applicability_domain

flags, thresholds, artifacts = applicability_domain(
    X_train, X_test, alpha=0.975, k=5,
    n_components=0.95, random_state=42, verbose=False
)
assert len(flags) == len(X_test)
flags.to_csv(OUT / "ad_flags.csv", index=False)
joblib.dump({"thresholds": thresholds, "artifacts": artifacts},
            OUT / "ad.joblib")
print(thresholds)
print(flags.head())
~~~

Los flags de PharmaToxAI identifican observaciones que **superan** umbrales de distancia; `True` significa potencialmente fuera del AD, no dentro. El AD basado en PCA es complementario a la cobertura estructural y mecanística.

~~~python
flag_cols = [c for c in flags if c.endswith("_flag")]
if flag_cols:
    rates = flags[flag_cols].astype(bool).mean().rename_axis("Criterio").reset_index(name="Fracción fuera")
    sns.barplot(data=rates, x="Fracción fuera", y="Criterio")
    plt.xlim(0, 1)
    plt.title("Fracción fuera del dominio de aplicabilidad")
    plt.tight_layout()
    plt.show()
~~~

## 6. Contrato de salida

~~~python
for filename in ["train_model.csv", "test_model.csv", "preprocessing.joblib",
                 "metrics.json", "test_predictions.csv", "ad_flags.csv", "ad.joblib"]:
    assert (OUT / filename).exists(), filename
print("Modelado completo:", OUT.resolve())
~~~

**Siguiente:** [Tutorial 04 — persistencia, inferencia y QMRF](04-persistencia-inferencia-qmrf.md). No reutilizar el test para elegir parámetros después de inspeccionar estas métricas.
