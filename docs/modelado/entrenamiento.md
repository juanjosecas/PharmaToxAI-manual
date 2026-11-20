# Entrenamiento y validación interna

Empieza con un baseline que puedas explicar y comparar. Separa las moléculas
antes de ajustar transformaciones o modelos; de lo contrario, información del
conjunto de evaluación puede filtrarse al entrenamiento.

## Partición de datos

Una partición aleatoria puede sobreestimar el rendimiento si moléculas muy
similares aparecen en entrenamiento y prueba. Escoge la estrategia en función
del uso previsto: por ejemplo, particiones por scaffold para explorar
generalización a esqueletos distintos o particiones temporales para simular
predicción futura. Conserva las asignaciones de cada partición.

La validación cruzada estima variabilidad dentro de los datos disponibles, pero
no reemplaza una evaluación externa. Si seleccionas hiperparámetros, usa una
validación interna separada dentro de cada fold externo.

## Tubería sin fuga de información

Mantén la transformación y el estimador en una sola tubería de modelado. El
ejemplo siguiente muestra el patrón de scikit-learn usado en el
[tutorial](../tutoriales/01-primer-qsar.md):

```python
from sklearn.linear_model import Ridge
from sklearn.model_selection import KFold, cross_validate
from sklearn.pipeline import make_pipeline
from sklearn.preprocessing import StandardScaler

modelo = make_pipeline(StandardScaler(), Ridge(alpha=1.0))
cv = KFold(n_splits=4, shuffle=True, random_state=17)
resultados = cross_validate(
    modelo,
    X,
    y,
    cv=cv,
    scoring="neg_root_mean_squared_error",
)
```

La tubería ajusta el escalado por separado en cada fold. `X` e `y` deben
contener exclusivamente el conjunto de entrenamiento de la evaluación que se
está ejecutando.

## Registro del experimento

Registra la semilla, particiones, representación molecular, algoritmo,
hiperparámetros, versiones, métricas y decisiones de selección. No elijas un
modelo por una única puntuación: considera la variabilidad, el desbalance y el
ámbito de aplicación.
