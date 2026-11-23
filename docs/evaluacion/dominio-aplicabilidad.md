# Dominio de aplicabilidad e incertidumbre

**Módulo:** `pharmatoxai.evaluation`. Una predicción puede ser numéricamente precisa en la evaluación global y aun así corresponder a una molécula fuera de la cobertura de entrenamiento.

## Dominio en espacio de descriptores

~~~python
from pharmatoxai.evaluation import applicability_domain

ad_flags, thresholds, artifacts = applicability_domain(
    X_train, X_test,
    alpha=0.975,
    k=5,
    n_components=0.95,
    manage_nan="median",
    random_state=42,
    verbose=False,
)
print(ad_flags.head())
print(thresholds)
~~~

La API devuelve **tres objetos**: indicadores por muestra, umbrales calibrados y artefactos de proyección/imputación. La función ajusta escalado, PCA y umbrales a partir de train; al trasladar el procedimiento a nuevas moléculas deben reutilizarse los artefactos guardados, no recalibrar sobre las moléculas de consulta. Para ello existe `applicability_domain_from_artifacts`.

El procedimiento incluye criterios derivados de Hotelling T², Mahalanobis robusto y distancia media kNN. Sus indicadores no son probabilidades de toxicidad, ni garantizan fiabilidad en otra familia química. El parámetro `alpha` controla calibración interna, no un nivel de confianza universal sobre el error predictivo.

## Similitud estructural

La cobertura del espacio de descriptores no reemplaza una auditoría de Tanimoto, scaffolds y fragmentos químicos. Analizar similitud train–test, presencia de análogos y pares de activity cliffs. Los métodos de distancia dependen de representación y umbral; registrar Morgan radius, número de bits y selección de columnas.

## Incertidumbre predictiva

No confundir dominio de aplicabilidad con intervalos de predicción. Un enfoque de conformal prediction necesita un conjunto de calibración y supuestos de intercambiabilidad; cambios de distribución pueden invalidar la cobertura nominal. Revisar la implementación en [uncertainty.py](https://github.com/juanjosecas/PharmaToxAI/blob/main/src/pharmatoxai/uncertainty.py).

## Informe mínimo

Reportar fracción dentro/fuera de dominio, umbrales, definición del espacio, desempeño separado por condición AD y criterios operativos cuando la molécula está fuera de dominio. Persistir transformadores, columnas originales y referencia de entrenamiento.

**Fuente:** [evaluation.py](https://github.com/juanjosecas/PharmaToxAI/blob/main/src/pharmatoxai/evaluation.py) y [documentación](https://github.com/juanjosecas/PharmaToxAI/blob/main/docs/evaluation.md).
