# Clasificación multilabel

**Módulo:** `pharmatoxai.multilabel`. Cada molécula puede pertenecer simultáneamente a varias clases binarias; se utilizan clasificadores independientes por endpoint sobre la misma matriz de features.

## Esquema y validaciones

`X` debe ser un DataFrame no vacío con nombres únicos; `y` debe ser un DataFrame binario con índices y orden coincidentes. La implementación actual **no admite valores faltantes en las etiquetas**: no presentar etiquetas desconocidas como 0. Se exige variación por endpoint y suficientes positivos/negativos para CV.

~~~python
from pharmatoxai.multilabel import (
    train_qsar_multilabel, evaluate_multilabel_model,
    optimize_multilabel_thresholds,
)

assert X_train.index.equals(y_train.index)
assert set(y_train.to_numpy().ravel()).issubset({0, 1})

result = train_qsar_multilabel(
    X_train, y_train,
    model_type="random_forest",
    tune=False, cv=3, scoring="f1_macro",
    impute_nan="median", random_state=42, n_jobs=1,
)
model = result["model"]
print(result["performance_scope"])  # training_resubstitution
~~~

El entrenamiento integra procesamiento por fold; si se suministra `groups`, utiliza GroupKFold para mantener grupos completos. La clasificación multilabel no equivale a clasificación multiclase de etiquetas mutuamente excluyentes.

## Umbrales, validación y test

~~~python
thresholds = optimize_multilabel_thresholds(
    model, X_validation, y_validation,
    metric="f1",
)
report = evaluate_multilabel_model(
    model, X_test, y_test,
    threshold=thresholds.to_numpy(),
)
print(report["metrics"])
~~~

La selección de umbrales **debe hacerse en validación, nunca en test**. La evaluación calcula métricas por endpoint y macro/micro según disponibilidad. Si una etiqueta del test carece de positivos o negativos, ciertas métricas son indefinidas y se reportan como `NaN`, no como cero.

### Limitación importante

La selección del modelo, sus hiperparámetros y el umbral son etapas distintas. Con datasets reales con etiquetas parciales, es necesario definir previamente un procedimiento explícito para manejar missing labels; la función actual no lo implementa.

**Fuente:** [multilabel.py](https://github.com/juanjosecas/PharmaToxAI/blob/main/src/pharmatoxai/multilabel.py) y [guía técnica](https://github.com/juanjosecas/PharmaToxAI/blob/main/docs/multilabel.md).

## Gráfico de prevalencia por endpoint

La prevalencia debe calcularse en `y_train`, antes de optimizar umbrales. Este gráfico ayuda a detectar tareas raras o muy desbalanceadas.

~~~python
import pandas as pd
import seaborn as sns
import matplotlib.pyplot as plt

prevalence = y_train.mean(axis=0).sort_values()
fig, ax = plt.subplots(figsize=(8, max(3, 0.35 * len(prevalence))))
sns.barplot(x=prevalence.values, y=prevalence.index, ax=ax)
ax.set(xlim=(0, 1), xlabel="Fracción de positivos",
       ylabel="Endpoint", title="Prevalencia por endpoint (train)")
plt.tight_layout()
plt.show()
~~~

Este bloque supone etiquetas binarias 0/1 completas, como exige actualmente `train_qsar_multilabel`.
