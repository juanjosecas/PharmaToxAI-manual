# Entrenamiento clásico de regresión y clasificación

**Módulo:** `pharmatoxai.training`. Partimos de `X_train` numérico y `y_train` definidos tras la partición; nunca pasar `Activity` ni identificadores dentro de `X_train`.

## Regresión

~~~python
from pharmatoxai.training import train_qsar_regression

artifact = train_qsar_regression(
    X_train, y_train,
    model_type="random_forest",
    tune=False,
    run_rfe=False,
    impute_nan="median",
    cv=5,
    random_state=42,
    n_jobs=1,
)
model = artifact["model"]
print(artifact["performance_scope"])  # training_resubstitution
~~~

`tune=True` activa búsqueda de hiperparámetros; revisar `search`, `n_iter`, `scoring`, `cv` y carga computacional. Los hiperparámetros deben elegirse sobre train/CV; el test se reserva para la evaluación final. No usar selección por RFE externo sin comprobar su interacción con CV y el holdout.

El artefacto incluye el modelo entrenado, descripción de features, configuración de entrenamiento, datos de búsqueda y métricas. Leer `performance_scope` antes de interpretarlas.

## Clasificación

~~~python
from pharmatoxai.training import train_qsar_classification

result = train_qsar_classification(
    X_train, y_train,
    model_type="random_forest",
    tune=False,
    cv=5,
    run_rfe=False,
    use_smote=False,
    impute_nan="median",
    balance_classes=True,
    random_state=42,
    n_jobs=1,
    verbose=False,
)
model = result["model"]
print(result["performance_scope"])
~~~

Para binaria, reportar por clase sensibilidad, especificidad, precisión, F1, balanced accuracy, MCC y métricas basadas en probabilidades. Para multicategoría, aclarar el promedio (macro, weighted, micro) y los parámetros de ROC-AUC. Elegir umbrales de decisión sin inspeccionar el test. Si se usa SMOTE, debe ajustarse exclusivamente en cada entrenamiento/fold, nunca antes del split.

## Validación interna

El modelo debe incluir dentro de la CV las operaciones ajustables de preprocesamiento. Como baseline independiente de la API, una tubería de scikit-learn es útil para auditar leakage:

~~~python
from sklearn.pipeline import make_pipeline
from sklearn.impute import SimpleImputer
from sklearn.preprocessing import StandardScaler
from sklearn.linear_model import Ridge
from sklearn.model_selection import KFold, cross_validate

pipeline = make_pipeline(SimpleImputer(strategy="median"), StandardScaler(), Ridge(alpha=1.0))
folds = KFold(n_splits=5, shuffle=True, random_state=42)
scores = cross_validate(pipeline, X_train, y_train, cv=folds, scoring="neg_root_mean_squared_error")
print(-scores["test_score"])
~~~

Si las moléculas están agrupadas por scaffold, sustituir KFold por un procedimiento de grupos y suministrar la identidad de cada grupo. La selección de representación también integra el proceso de optimización y no debe hacerse sobre test.

**Fuentes:** [training.py](https://github.com/juanjosecas/PharmaToxAI/blob/main/src/pharmatoxai/training.py), [documentación](https://github.com/juanjosecas/PharmaToxAI/blob/main/docs/training.md), notebooks 03 de regresión y clasificación.


## Fundamentos del ajuste y elección de hiperparámetros

El modelo busca una función \(f_\theta(x)\) que explique o prediga un endpoint a partir de descriptores. En regresión, una función objetivo general es

$$
\hat\theta=\arg\min_\theta
\left[
\frac{1}{n}\sum_{i=1}^n L(y_i,f_\theta(x_i))
+\lambda\,\Omega(\theta)
\right]
$$

La función de pérdida \(L\) mide discrepancia, mientras que \(\Omega\) penaliza complejidad; \(\lambda\) controla la fuerza de regularización. Su significado **varía por estimador**. El parámetro regularization_strength de PharmaToxAI es una interfaz de configuración y no debe interpretarse como el mismo \(\lambda\) numérico para RandomForest, SVM o regresión lineal.

### Regularización lineal y modelos de árboles

Para Ridge, la penalización típica es \(\Omega(\beta)=\lVert\beta\rVert_2^2\). Para Lasso, \(\Omega(\beta)=\lVert\beta\rVert_1\), que favorece coeficientes exactamente nulos. Elastic Net combina ambas. La penalización requiere especial atención al escalado de las variables.

En Random Forest, la capacidad está determinada por profundidad, número mínimo de muestras por hoja, número de árboles y subconjunto de características, no por una única penalización cuadrática. En boosting, learning rate, número de iteraciones, profundidad, regularización y early stopping afectan simultáneamente el sesgo y la varianza.

### Validación cruzada y búsqueda

En K-fold, los datos de desarrollo se dividen en \(K\) subconjuntos. Cada fold se predice con un estimador ajustado sin sus observaciones. La media de una métrica es

$$
\bar M_{\mathrm{CV}}=\frac{1}{K}\sum_{k=1}^{K}M_k
$$

Los folds no son muestras independientes: esta expresión resume el protocolo de evaluación, **no** constituye por sí sola un intervalo de confianza. RepeatedKFold repite asignaciones y permite estudiar sensibilidad, sin producir nuevas muestras biológicas.

La búsqueda selecciona hiperparámetros \(\lambda^*\) optimizando una métrica interna:

$$
\lambda^*=\arg\min_{\lambda\in\Lambda}
\widehat R_{\mathrm{CV}}(\lambda)
$$

Buscar muchos modelos y escoger el mínimo introduce sesgo de selección si se informa esa misma CV como rendimiento final. La CV anidada separa selección interna de evaluación externa. El test permanece intacto hasta la selección definitiva.

| Parámetro de PharmaToxAI | Interpretación |
| --- | --- |
| cv | Número de folds de validación; limitado por observaciones y grupos |
| cv_repeats | Número de repeticiones de particiones, cuando el entrenador correspondiente las admite |
| scoring | Función para seleccionar configuraciones; debe representar el coste de los errores |
| tune | Activa optimización de hiperparámetros |
| search, n_iter | Tipo y presupuesto de búsqueda, relevantes principalmente para regresión |
| run_rfe, rfe_step | Eliminación recursiva de features; su ajuste debe quedar dentro del protocolo de validación |
| impute_nan | Política para completar faltantes |
| target_transform | Transformación del endpoint; la inversión debe conservar la escala original al evaluar |
| n_jobs | Paralelismo; puede competir con hilos internos de BLAS o estimadores |

### Desbalance y SMOTE en clasificación

Si hay \(n_+\) positivos y \(n_-\) negativos, la prevalencia es

$$
\pi=\frac{n_+}{n_++n_-}
$$

Una accuracy alta puede conseguirse ignorando la clase minoritaria. Las ponderaciones de clase modifican el peso de cada error. SMOTE crea puntos sintéticos por interpolación en el espacio de características; su operación depende de distancia y vecinos, y **no garantiza** que un vector sintetizado corresponda a una molécula físicamente realizable. Debe ajustarse dentro del fold de entrenamiento y evaluarse frente a la alternativa de class weights.

Para QSAR estructural, utilizar grupos en la validación interna cuando el objetivo sea extrapolar a nuevas familias. Una selección de features hecha fuera de CV puede introducir optimismo incluso si test no participa.

### Early stopping

La detención temprana termina el ajuste cuando la pérdida sobre validación deja de mejorar durante una ventana de paciencia. Esta validación debe ser independiente del ajuste, incluida la transformación de features y selección supervisada. Su puntuación no reemplaza el holdout externo.

## Visualización de la validación cruzada

Continuación del ejemplo con `scores` obtenido mediante `cross_validate`. Mostrar dispersión entre folds en lugar de un único promedio.

~~~python
import pandas as pd
import seaborn as sns
import matplotlib.pyplot as plt

rmse_folds = -scores["test_score"]
plot_df = pd.DataFrame({"Fold": range(1, len(rmse_folds) + 1),
                        "RMSE": rmse_folds})
fig, ax = plt.subplots(figsize=(6, 4))
sns.barplot(data=plot_df, x="Fold", y="RMSE", color="steelblue", ax=ax)
ax.axhline(rmse_folds.mean(), ls="--", color="black",
           label=f"Media = {rmse_folds.mean():.3g}")
ax.set(title="RMSE por fold de validación cruzada")
ax.legend()
plt.tight_layout()
plt.show()
~~~

Estas métricas pertenecen al pipeline de Ridge del ejemplo, no al `artifact` de PharmaToxAI. No interpretar `performance_scope="training_resubstitution"` como validación externa.
