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


## Definiciones matemáticas e interpretación

### Regresión

Con \(e_i=y_i-\hat y_i\) y \(n\) muestras:

$$
MAE=\frac{1}{n}\sum_i|e_i|,\qquad
RMSE=\sqrt{\frac{1}{n}\sum_i e_i^2}
$$

RMSE penaliza más los grandes errores y tiene la misma unidad que el endpoint. MAE es menos sensible a errores extremos. Un valor RMSE=0.5 no significa lo mismo para pIC50 que para porcentajes de inhibición.

El coeficiente de determinación calculado sobre un conjunto evaluado es

$$
R^2=1-\frac{\sum_i(y_i-\hat y_i)^2}
{\sum_i(y_i-\bar y)^2}
$$

Puede ser negativo en test. Su interpretación depende de la variabilidad del target, por lo que comparar R² entre datasets de distinto rango puede resultar engañoso.

El coeficiente de correlación concordante (CCC) combina precisión y acuerdo absoluto:

$$
CCC=\frac{2\,\operatorname{cov}(y,\hat y)}
{\operatorname{var}(y)+\operatorname{var}(\hat y)+(\bar y-\overline{\hat y})^2}
$$

Su convención sobre el denominador de covarianza y varianza debe ser consistente. Pearson elevado no garantiza concordancia: una predicción \(\hat y=y+5\) puede tener correlación perfecta y error sistemático grande.

El Q² requiere especificar su referencia y el procedimiento que generó las predicciones; Q² de predicciones out-of-fold no debe confundirse con R² de un conjunto reutilizado para selección.

### Clasificación: matriz de confusión

| | Predicción positiva | Predicción negativa |
| --- | --- | --- |
| Positivo real | TP | FN |
| Negativo real | FP | TN |

$$
\mathrm{Sensibilidad}=\frac{TP}{TP+FN},\qquad
\mathrm{Especificidad}=\frac{TN}{TN+FP}
$$

$$
\mathrm{Precisión}=\frac{TP}{TP+FP},\quad
F_1=2\frac{\mathrm{Precisión}\cdot\mathrm{Sensibilidad}}
{\mathrm{Precisión}+\mathrm{Sensibilidad}}
$$

$$
\mathrm{BalancedAccuracy}=\frac{\mathrm{Sensibilidad}+\mathrm{Especificidad}}{2}
$$

En clasificación binaria, MCC utiliza todas las celdas de la matriz:

$$
MCC=\frac{TP\cdot TN-FP\cdot FN}
{\sqrt{(TP+FP)(TP+FN)(TN+FP)(TN+FN)}}
$$

Los casos de denominador cero requieren una convención de implementación. Accuracy puede ser alta por simple prevalencia; F1 no utiliza TN y puede ocultar mal desempeño en negativos.

### Curvas y umbrales

La curva ROC evalúa sensibilidad frente a \(FPR=FP/(FP+TN)\) al variar el umbral. La curva precisión–recall relaciona precisión con sensibilidad; el baseline de average precision depende de la prevalencia positiva. ROC-AUC resume ordenamiento de scores y no certifica probabilidades calibradas.

La tasa Brier para binaria es

$$
BS=\frac{1}{n}\sum_i(p_i-y_i)^2
$$

La calibración describe correspondencia entre probabilidades previstas y frecuencias observadas. El umbral óptimo depende del costo de FP/FN y debe decidirse en validación.

### Parámetros de evaluación

| Parámetro | Decisión |
| --- | --- |
| average="macro" | Peso igual para cada clase, aun si son raras |
| average="weighted" | Ponderación por soporte de cada clase |
| average="micro" | Agrega decisiones globalmente |
| zero_division | Convención cuando precisión/recall no están definidas |
| threshold | Determina matriz de confusión; no modifica ranking ROC |
| proba | Debe estar alineada con clases y orden del estimador |

En el ejemplo binario del capítulo, comprobar la etiqueta correspondiente a model.classes_[1] antes de interpretar predict_proba(... )[:, 1] como probabilidad de la clase positiva. ROC-AUC y average precision requieren positivos y negativos para la interpretación habitual; una partición sin ambas clases puede producir métricas indefinidas.

Para errores heterogéneos, reportar métricas por scaffold, rango de endpoint y condición de dominio de aplicabilidad, además de un resumen global.

## Gráficos diagnósticos de regresión y clasificación

Para regresión, ejecutar después de obtener `pred = model.predict(X_test)` y `y_test` numérico:

~~~python
import numpy as np
import pandas as pd
import seaborn as sns
import matplotlib.pyplot as plt

obs = np.asarray(y_test, dtype=float).ravel()
estimated = np.asarray(pred, dtype=float).ravel()
valid = np.isfinite(obs) & np.isfinite(estimated)
plot_df = pd.DataFrame({"Observado": obs[valid], "Predicho": estimated[valid]})
plot_df["Residuo"] = plot_df["Observado"] - plot_df["Predicho"]

fig, ax = plt.subplots(figsize=(6, 5))
sns.scatterplot(data=plot_df, x="Observado", y="Predicho", ax=ax)
limits = [plot_df[["Observado", "Predicho"]].min().min(),
          plot_df[["Observado", "Predicho"]].max().max()]
ax.plot(limits, limits, "k--", lw=1)
ax.set(title="Predicción externa: observado vs. predicho")
plt.tight_layout()
plt.show()

fig, ax = plt.subplots(figsize=(6, 4))
sns.scatterplot(data=plot_df, x="Predicho", y="Residuo", ax=ax)
ax.axhline(0, color="black", ls="--")
ax.set(title="Residuos en el conjunto de evaluación")
plt.tight_layout()
plt.show()
~~~

Para clasificación, **recalcular** `pred` con el clasificador antes de ejecutar:

~~~python
from sklearn.metrics import confusion_matrix
import seaborn as sns
import matplotlib.pyplot as plt

labels = sorted(set(y_test) | set(pred))
cm = confusion_matrix(y_test, pred, labels=labels)
fig, ax = plt.subplots(figsize=(5, 4))
sns.heatmap(cm, annot=True, fmt="d", cmap="Blues",
            xticklabels=labels, yticklabels=labels, ax=ax)
ax.set(xlabel="Predicción", ylabel="Observado", title="Matriz de confusión")
plt.tight_layout()
plt.show()
~~~

## Diagnóstico integrado de residuos

`plot_residuals_analysis` representa simultáneamente train y test. El bloque corresponde **solo a regresión** y necesita predicciones en ambos conjuntos.

~~~python
import numpy as np
import matplotlib.pyplot as plt
from pharmatoxai.visualization import plot_residuals_analysis

pred_train = np.asarray(model.predict(X_train), dtype=float).ravel()
pred_test = np.asarray(model.predict(X_test), dtype=float).ravel()
fig, axes = plot_residuals_analysis(
    np.asarray(y_train, dtype=float).ravel(), pred_train,
    np.asarray(y_test, dtype=float).ravel(), pred_test,
)
plt.show()
~~~

Las predicciones de train son de resustitución y solo sirven para diagnosticar sobreajuste; no sustituyen las métricas externas.
