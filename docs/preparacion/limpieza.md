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
