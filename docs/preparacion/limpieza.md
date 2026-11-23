# Limpieza de características y prevención de leakage

**Módulo:** `pharmatoxai.feature_cleaning`. Eliminar variables inválidas o redundantes exige separar las operaciones exploratorias de las transformaciones aprendidas.

## Reglas de trabajo

1. Fijar identidad molecular y estrategia de partición antes de ajustar imputadores, filtros o escaladores.
2. Ajustar selección de columnas, imputación, umbrales basados en distribución y escalado con **train exclusivamente**.
3. Aplicar la misma selección de columnas y parámetros a validación y test, sin volver a ajustar.
4. Dentro de validación cruzada, repetir el ajuste de transformaciones **en cada fold**.
5. Nunca incorporar `Activity`, índices de muestra ni etiquetas del test como predictores.

## Ejemplo con funciones del paquete

~~~python
import numpy as np
import pandas as pd
from pharmatoxai.feature_cleaning import (
    remove_nan_inf, impute_median_train,
    remove_low_variance, remove_correlated,
)

X_train = pd.DataFrame({
    "MW": [100., 120., 150., 180., 200.],
    "MW_copy": [100., 120., 150., 180., 200.],
    "TPSA": [10., 20., np.nan, 30., 40.],
    "Constant": [1., 1., 1., 1., 1.],
})
X_test = pd.DataFrame({
    "MW": [130., 175.],
    "MW_copy": [130., 175.],
    "TPSA": [np.nan, 28.],
    "Constant": [1., 1.],
})

train = remove_nan_inf(X_train, max_nan_frac=0.25)
test = X_test[train.columns].replace([np.inf, -np.inf], np.nan)
train, test = impute_median_train(train, test)

keep = remove_low_variance(train, threshold=0.0).columns
train, test = train[keep], test[keep]
keep = remove_correlated(train, threshold=0.95).columns
train, test = train[keep], test[keep]

assert train.columns.equals(test.columns)
assert train.notna().all().all() and test.notna().all().all()
print("Variables finales:", train.columns.tolist())
~~~

Este ejemplo deliberadamente pequeño permite comprobar la propagación de columnas y de la mediana de train. En datos reales, columnas completamente vacías en train deben excluirse antes de imputación. Los tests del repositorio cubren invariantes y problemas de leakage adicionales.

## Qué hace cada operación

| Operación | Ajuste permitido | Riesgo metodológico |
| --- | --- | --- |
| Reemplazar infinitos por NaN | Determinista | Puede ocultar fallos del generador si no se audita |
| Filtrar por proporción de NA | Solo train | Selección informada por test |
| Imputación mediana | Solo train | Desplazar la distribución utilizando el test |
| Filtrado por varianza | Solo train | Umbrales dependientes de unidades |
| Filtrado por correlación / VIF | Solo train | Cambiar columnas y perder trazabilidad |
| Selección por correlación con `y` | Train, dentro de CV | Leakage supervisado |
| PCA / escalado | Train, dentro de CV | Optimismo por ajuste global |

`clean_descriptors` sirve para limpiar una matriz durante exploración, pero **no** constituye por sí solo una transformación train/test reutilizable. `DataCleaner` ofrece una interfaz integrada; para su uso predictivo es necesario inspeccionar cómo ajusta cada etapa y verificar el comportamiento mediante tests, especialmente para operaciones avanzadas.

## Criterios que no son universales

Un umbral `var_threshold=0.01` carece de interpretación uniforme en variables con distintas escalas; filtros de bits binarios y descriptores continuos pueden requerir estrategias separadas. Un `corr_threshold=0.95` reduce redundancia, pero no garantiza mayor capacidad predictiva. Las decisiones deben evaluarse dentro de validación cruzada, no sobre el test externo.

**Salida:** train/test con columnas idénticas y en el mismo orden; parámetros del preprocesamiento conservados; registro de variables eliminadas y motivo.

**Fuentes:** [feature_cleaning.py](https://github.com/juanjosecas/PharmaToxAI/blob/main/src/pharmatoxai/feature_cleaning.py), [documentación](https://github.com/juanjosecas/PharmaToxAI/blob/main/docs/feature_cleaning.md), [notebook 02](https://github.com/juanjosecas/PharmaToxAI/blob/main/notebooks/02_Featurization_Cleaning_Splitting%20of%20Molecular%20Dataset.ipynb).


## Fundamentos matemáticos de la limpieza

### Frecuencia de valores ausentes

Para una columna \(j\) de entrenamiento:

$$
f_{\mathrm{NA},j}=\frac{1}{n_{\rm train}}\sum_{i\in\rm train}\mathbf{1}(x_{ij}\text{ es NaN})
$$

remove_nan_inf(max_nan_frac=0.2) reemplaza \(\pm\infty\) por NaN y elimina columnas con \(f_{\mathrm{NA},j}>0.2\), **no con igualdad**. Una columna con 1 NA de 5 filas se conserva. Imputar faltantes causados por fallos químicos sistemáticos puede introducir sesgos.

### Imputación

La función impute_median_train utiliza medianas de train:

$$
\tilde x_j=\mathrm{mediana}\{x_{ij}:i\in\mathrm{train},\,x_{ij}\ne\mathrm{NaN}\}
$$

Los faltantes de train y test se reemplazan con el mismo \(\tilde x_j\). La imputación reduce dispersión observada y no reconstruye dependencias entre descriptores. Una columna completamente ausente en train conserva mediana NaN; excluirla antes de imputar.

### Umbral de varianza

remove_low_variance() calcula varianza poblacional con ddof=0:

$$
v_j=\frac{1}{n_{\rm train}}\sum_i(x_{ij}-\bar x_j)^2
$$

Conserva columnas con \(v_j>\tau\). Para un bit binario con frecuencia activa \(p_j\):

$$
v_j=p_j(1-p_j)
$$

Por tanto, var_threshold=0.01 en bits excluye características muy raras o casi universales; en descriptores continuos depende de sus **unidades**. Aplicar el filtro tras estandarizar elimina esa interpretación, pues las variables no constantes suelen adquirir varianza unitaria.

### Correlación de columnas

remove_correlated(threshold=0.95) elimina columnas que tienen \(|r_{jk}|>0.95\) respecto de alguna columna anterior en el orden de la tabla:

$$
|r_{jk}|>\tau\Longrightarrow\text{variable redundante candidata a exclusión}
$$

Utiliza Pearson y el triángulo superior; no garantiza conservar la variable más relacionada con el endpoint. Cambiar el orden de columnas puede cambiar el subconjunto final.

### Multicolinealidad y VIF

$$
VIF_j=\frac{1}{1-R_j^2}
$$

\(R_j^2\) es el coeficiente de determinación de una regresión auxiliar del descriptor \(j\) contra los restantes. Si \(R_j^2=0.9\), entonces VIF=10. Es relevante para estabilidad de coeficientes lineales; no establece la predictividad de árboles. El procedimiento iterativo puede ser costoso o inestable si \(p\gg n\).

### Otros filtros: cuándo tienen sentido

| Función / parámetro | Fundamento | Restricción |
| --- | --- | --- |
| remove_low_cardinality: min_unique, min_frac | Descarta diversidad de valores insuficiente | Los bits binarios requieren criterio específico |
| remove_low_signal_regression: min_abs_spearman | Exige \(|\rho_s|\) mínimo con y_train | Selección supervisada; reajustar dentro de CV |
| remove_distribution_shift: ks_threshold | \(D=\sup_x|F_{\rm train}(x)-F_{\rm test}(x)|\) | Si elimina columnas mirando el test externo, hay leakage de selección |
| remove_multicollinearity_pca: var_retained | Retiene fracción de varianza lineal | Componentes menos interpretables; ajuste solo sobre train |

**Advertencia de API:** remove_distribution_shift(train, test) compara distribuciones y elimina columnas usando el conjunto llamado test. Puede servir como **diagnóstico** del shift, pero no debe modificar la selección de variables al evaluar un holdout externo. Usar un conjunto de desarrollo explícito para decisiones de selección.

### Preprocesamiento dentro de validación cruzada

Para cada fold \(k\), las transformaciones \(\hat\theta_k\) deben ajustarse con train y aplicarse sin ajuste adicional al fold de validación:

$$
\hat\theta_k=\mathrm{FitPreprocess}(X_{{\rm train},k}),\qquad
\hat y_{{\rm val},k}=
f_k(\mathrm{Transform}_{\hat\theta_k}(X_{{\rm val},k}))
$$

Esto incluye imputación, escalado, PCA y filtros supervisados. Guardar umbrales, columnas retenidas, orden y parámetros aprendidos permite reutilizar el procesamiento sobre nuevas moléculas.

## Diagnóstico gráfico de la limpieza

Comparar porcentaje de faltantes antes y después de los filtros, usando `X_train` y `train` del ejemplo.

~~~python
import pandas as pd
import seaborn as sns
import matplotlib.pyplot as plt

missing = pd.concat([
    X_train.isna().mean().rename("Antes"),
    train.isna().mean().rename("Después")
], axis=1).fillna(0)
long = missing.reset_index(names="Descriptor").melt(
    id_vars="Descriptor", var_name="Etapa", value_name="Fracción faltante"
)
fig, ax = plt.subplots(figsize=(8, max(3, 0.35 * len(missing))))
sns.barplot(data=long, y="Descriptor", x="Fracción faltante", hue="Etapa", ax=ax)
ax.set(title="Valores ausentes por descriptor", xlim=(0, 1))
plt.tight_layout()
plt.show()
~~~

La imputación posterior a los filtros elimina NA del conjunto de entrenamiento; este gráfico describe las columnas retenidas, no valida el desempeño del modelo.

## Gráfico integrado de comparación de distribuciones

La función integrada compara una variable antes y después de la limpieza, usando los valores del ejemplo:

~~~python
import numpy as np
import matplotlib.pyplot as plt
from pharmatoxai.visualization import plot_distribution_comparison

shared = X_train.columns.intersection(train.columns)
if len(shared):
    col = shared[0]
    before = X_train[col].replace([np.inf, -np.inf], np.nan).dropna().to_numpy()
    after = train[col].replace([np.inf, -np.inf], np.nan).dropna().to_numpy()
    if len(before) and len(after):
        fig, axes = plot_distribution_comparison(
            before, after, title=f"Distribución antes/después: {col}",
            xlabel=col, labels=("Original", "Limpio")
        )
        plt.show()
~~~

No confundir el cambio de distribución con una mejora del desempeño predictivo.
