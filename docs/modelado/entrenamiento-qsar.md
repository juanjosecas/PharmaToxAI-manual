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
