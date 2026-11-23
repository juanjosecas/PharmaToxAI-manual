# Exploración inicial y EDA

**Módulo:** `pharmatoxai.eda`. El EDA debe caracterizar datos y decisiones de diseño sin utilizar el conjunto de test para ajustar un pipeline predictivo.

## Auditoría descriptiva

~~~python
from pharmatoxai.data_io import load_data
from pharmatoxai.eda import basic_counts, target_summary

df = load_data("data/compounds.csv")
counts = basic_counts(df, smiles_col="SMILES", target_col="Activity")
print(counts)

# Para un endpoint cuantitativo correctamente parseado:
print(target_summary(df, target_col="Activity"))
~~~

`basic_counts` cuenta SMILES textuales: dos SMILES diferentes pueden describir la misma estructura. `target_summary` calcula métricas numéricas cuando la columna tiene tipo numérico; si no, devuelve recuentos por categoría.

## Distribución del endpoint

~~~python
import matplotlib.pyplot as plt
import pandas as pd

y = pd.to_numeric(df["Activity"], errors="coerce").dropna()
fig, ax = plt.subplots(figsize=(7, 4))
ax.hist(y, bins="auto", edgecolor="white")
ax.set(title="Distribución del endpoint", xlabel="Actividad (unidad definida)", ylabel="Frecuencia")
fig.tight_layout()
plt.show()
~~~

Interpretar asimetría, rango y posibles límites de detección de acuerdo con la definición fisicoquímica o biológica del endpoint. Una transformación logarítmica exige unidades consistentes y valores válidos; no aplicar `log10` indiscriminadamente a valores negativos o cero. En clasificación, sustituir el histograma por frecuencias de clases y revisar el desbalance.

## Matriz de descriptores y correlaciones

~~~python
from pharmatoxai.eda import correlation_matrix
numeric = df.select_dtypes(include="number").drop(columns=["Activity"], errors="ignore")
if numeric.shape[1] >= 2:
    corr = correlation_matrix(numeric, method="spearman")
    print(corr.iloc[:5, :5])
~~~

Las correlaciones entre características son exploratorias. La selección de variables, los umbrales aprendidos y la imputación que intervengan en un modelo deben ajustarse **solo con train**, y volver a aplicarse a validation/test.

## Proyección del espacio de descriptores

~~~python
from pharmatoxai.eda import run_pca
X = numeric.replace([float("inf"), -float("inf")], float("nan"))
if X.shape[0] >= 3 and X.shape[1] >= 2:
    coords, explained, eigenvalues = run_pca(
        X, n_components=2, scaler="standar", correct_nan=True
    )
    print("Varianza explicada:", explained)
    print(coords.head())
~~~

El parámetro `scaler="standar"` conserva la grafía utilizada en la API actual. `run_pca` ajusta internamente escalado e imputación: adecuado para exploración de un dataset; **no reutilizar la proyección ajustada al conjunto completo en un pipeline predictivo**. Para modelado, emplear transformadores con `fit` sobre train.

UMAP y t-SNE son proyecciones exploratorias sensibles a hiperparámetros. Distancias entre grupos en una figura no prueban separación química, causalidad ni capacidad predictiva.

## EDA molecular

Tras curación, pueden examinarse scaffolds, diversidad y regiones poco representadas. Las moléculas acíclicas pueden carecer de scaffold de Bemis–Murcko; una ausencia no equivale a una estructura inválida.

**Entregables:** tabla de calidad, definición del endpoint, distribución de actividad, descripción del espacio químico y decisiones registradas antes del modelado.

**Fuente:** [eda.py](https://github.com/juanjosecas/PharmaToxAI/blob/main/src/pharmatoxai/eda.py), [docs/eda.md](https://github.com/juanjosecas/PharmaToxAI/blob/main/docs/eda.md) y [notebook de EDA](https://github.com/juanjosecas/PharmaToxAI/blob/main/notebooks/01_Exploratory%20Data%20Analysis.ipynb).
