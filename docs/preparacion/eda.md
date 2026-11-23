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


## Fundamentos estadísticos de los parámetros

### Distribución del objetivo

Para \(n\) observaciones de una variable continua, media y varianza muestral son

$$
\bar y=\frac{1}{n}\sum_i y_i,\qquad s^2=\frac{1}{n-1}\sum_i(y_i-\bar y)^2
$$

La mediana y el intervalo intercuartílico \(IQR=Q_{0.75}-Q_{0.25}\) son menos sensibles a valores extremos. Histogramas y KDE dependen del ancho de banda; una aparente multimodalidad debe verificarse cambiando esa resolución. Los estadísticos habituales requieren tratamiento especial para endpoints censurados.

### Correlación de descriptores

La correlación de Pearson mide dependencia lineal:

$$
r_{xy}=\frac{\sum_i(x_i-\bar x)(y_i-\bar y)}
{\sqrt{\sum_i(x_i-\bar x)^2\sum_i(y_i-\bar y)^2}}
$$

Spearman calcula correlación sobre rangos y detecta asociaciones monotónicas. Kendall se basa en pares concordantes y discordantes; sin empates, \(\tau=(C-D)/\binom{n}{2}\). Ninguna de estas medidas implica causalidad ni detecta toda posible dependencia no lineal.

### PCA: varianza y escalado

Dada la matriz centrada \(X_c\), PCA diagonaliza su covarianza y obtiene puntuaciones proyectadas:

$$
\Sigma=\frac{X_c^\top X_c}{n-1},\quad \Sigma v_k=\lambda_k v_k,\quad z_{ik}=x_{i,c}^\top v_k
$$

La varianza explicada por cada componente es

$$
EVR_k=\frac{\lambda_k}{\sum_j\lambda_j}
$$

| Parámetro de run_pca | Interpretación | Consecuencia |
| --- | --- | --- |
| n_components | Número de componentes retenidos | 2 o 3 para visualizar; mayor número para conservar variabilidad |
| scaler="standar" | Estandariza como \(z=(x-\mu)/\sigma\) | Reduce dominancia de descriptores con unidades grandes |
| scaler="minmax" | Reescala al intervalo basado en mínimo y máximo | Sensible a extremos |
| scaler="robust" | Mediana e IQR | Menos sensible a extremos marginales |
| scaler=None | Mantiene magnitudes | Variables de gran varianza pueden dominar |
| correct_nan=True | Reemplaza infinitos, filtra casi constantes e imputa mediana | En esta API se ajusta sobre los datos proporcionados |
| var_threshold | Varianza mínima antes del PCA | Se evalúa antes del escalado; depende de las unidades |

**Detalle de implementación:** el tercer resultado de run_pca es pca.singular_values_, **no autovalores** a pesar del nombre usado en la documentación previa. Los autovalores se recuperan como \(\lambda_k=s_k^2/(n-1)\) para la matriz centrada de entrada del PCA.

### t-SNE y UMAP

t-SNE construye probabilidades de vecindad en ambos espacios y minimiza su divergencia KL:

$$
KL(P\Vert Q)=\sum_{i\ne j}p_{ij}\log\frac{p_{ij}}{q_{ij}}
$$

La perplexity es \(2^{H(P_i)}\), con entropía en bits, y ajusta la escala de vecindad. Debe ser menor que el número de muestras. Las distancias entre grupos lejanos en un gráfico t-SNE no representan necesariamente distancias químicas reales.

UMAP construye un grafo de vecindad y optimiza una representación de baja dimensión. La geometría depende de la métrica, vecinos, min_dist y semilla. La API run_umap solo expone n_components y metric, aplica StandardScaler internamente y fija la semilla a 42. La función chemical_space_embedding para fingerprints puede usar Jaccard **directamente sobre bits**, que constituye una transformación diferente.

### Diversidad de scaffolds

$$
D_s=\frac{N_{\rm scaffolds}}{N_{\rm moléculas\ con\ scaffold}},\qquad
f_{\max}=\frac{\max_k n_k}{N_{\rm moléculas\ con\ scaffold}}
$$

Estos índices describen diversidad según Bemis–Murcko, no cobertura química absoluta. compute_scaffolds() descarta resultados sin scaffold, incluidos compuestos acíclicos: registrar el denominador y comprobar alineación de índices antes de combinarlo con otra tabla.

| Pregunta | Parámetro útil |
| --- | --- |
| ¿Dependencia lineal o monotónica? | method: pearson, spearman o kendall |
| ¿Cuánta dispersión lineal conservar? | n_components |
| ¿Influyen magnitudes y unidades? | scaler |
| ¿Vecindarios químicos locales? | metric y perplexity, según método |
| ¿Agrupamientos euclidianos impuestos? | n_clusters de K-Means |

PCA, t-SNE, UMAP y clustering son **exploratorios**. No prueban generalización y no deben ajustarse a un test externo si sus transformaciones se usarán para modelar.

## Visualización con Seaborn

Los siguientes gráficos usan `df`, `numeric` y `coords` de los ejemplos anteriores.

~~~python
import seaborn as sns
import matplotlib.pyplot as plt

sns.set_theme(style="whitegrid")
fig, ax = plt.subplots(figsize=(7, 4))
sns.histplot(data=df, x="Activity", kde=True, ax=ax)
ax.set(title="Distribución del endpoint", xlabel="Actividad")
plt.tight_layout()
plt.show()

if numeric.shape[1] >= 2:
    fig, ax = plt.subplots(figsize=(8, 6))
    sns.heatmap(numeric.iloc[:, :15].corr(method="spearman"),
                cmap="vlag", center=0, ax=ax)
    ax.set_title("Correlación de Spearman entre descriptores")
    plt.tight_layout()
    plt.show()

if "coords" in globals() and coords.shape[1] >= 2:
    pca_plot = coords.iloc[:, :2].copy()
    pca_plot.columns = ["PC1", "PC2"]
    fig, ax = plt.subplots(figsize=(6, 5))
    sns.scatterplot(data=pca_plot, x="PC1", y="PC2", ax=ax)
    ax.set_title("PCA exploratorio")
    plt.tight_layout()
    plt.show()
~~~

La visualización de PCA no demuestra separación predictiva ni justifica seleccionar features usando test.
