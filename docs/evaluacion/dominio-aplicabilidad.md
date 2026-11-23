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


## Fundamento matemático de las distancias AD

El dominio de aplicabilidad intenta cuantificar si un compuesto nuevo está representado por el espacio de descriptores de entrenamiento. PharmaToxAI estandariza, proyecta con PCA y estima umbrales sobre las puntuaciones \(z\).

### Hotelling T²

La distancia cuadrática con covarianza clásica del espacio PCA se calcula como

$$
T^2(z)=z^\top\Sigma_{\mathrm{train}}^{+}z
$$

donde \(\Sigma^+\) es la pseudoinversa; el cálculo actual supone que las puntuaciones están centradas mediante PCA. Valores altos indican posiciones inusuales respecto de la dispersión de entrenamiento. No significan automáticamente errores predictivos altos.

### Mahalanobis robusto

Una distancia cuadrática robusta utiliza centro \(\hat\mu_R\) y covarianza \(\hat\Sigma_R\) estimados con Minimum Covariance Determinant (MinCovDet):

$$
D_M^2(z)=(z-\hat\mu_R)^\top
\hat\Sigma_R^{-1}(z-\hat\mu_R)
$$

Puede ser menos sensible a observaciones extremas que la covarianza clásica. Requiere suficientes observaciones y una dimensionalidad PCA compatible para estimaciones estables.

### Distancia a vecinos

Para los \(k\) vecinos más cercanos de \(z\) dentro de la referencia:

$$
d_{\mathrm{kNN}}(z)=\frac{1}{k}
\sum_{j\in N_k(z)}\lVert z-z_j\rVert_2
$$

El valor de k controla la escala local: k pequeño es sensible a vecinos individuales; k grande refleja la densidad de regiones más amplias. En espacios con alta dimensionalidad, distancias euclidianas pueden concentrarse.

**Limitación de implementación:** al calibrar el umbral de entrenamiento, knn_mean_distance(Z_train, Z_train) incluye la propia observación entre vecinos, con distancia cero. Para moléculas nuevas no existe ese vecino idéntico por definición. El umbral kNN puede resultar demasiado bajo; para calibración rigurosa, excluir auto-vecinos o usar referencias de calibración separadas.

### Umbrales actuales

calibrate_ad_thresholds produce cuatro valores:

| Clave | Cálculo en código | Interpretación |
| --- | --- | --- |
| T2_emp | Cuantil alpha de T² de train | Umbral empírico |
| T2_theory | Cuantil chi-cuadrado con grados de libertad iguales a componentes PCA | Referencia teórica aproximada |
| MD2 | Cuantil alpha de Mahalanobis robusto de train | Umbral robusto empírico |
| KNN | Cuantil fijo 0.95 de distancias kNN de train | **No** depende de alpha |

El flag T2 se activa si la observación supera el máximo de T2_emp y T2_theory. MD2_flag compara con MD2 y kNN_flag con KNN. Por tanto, aumentar alpha modifica umbrales T² y MD², pero no el umbral kNN fijado en 0.95.

### Otros parámetros

| Parámetro | Significado |
| --- | --- |
| n_components=0.95 | Retiene aproximadamente el 95 % de la varianza en PCA; con entero especifica cantidad de componentes |
| max_nan_ratio | Máximo de faltantes permitido por descriptor en el entrenamiento |
| var_threshold | Descarta variables casi constantes antes de PCA |
| manage_nan | Media, mediana, moda o ninguna imputación; se calcula desde train |
| random_state | Fija componentes aleatorios de estimación robusta |
| alpha | Cuantil para T² y MD², no cobertura nominal de un intervalo predictivo |

### Uso y validación

Un criterio de dominio no debe calibrarse con moléculas que se usarán para calificar el desempeño externo. Informar cobertura AD y métricas por región (dentro/fuera), con denominadores suficientes. Las distancias PCA pueden perder información de fragmentos estructurales raros; complementar con similitud Tanimoto y scaffolds. Tres flags no representan tres probabilidades independientes.

La pertenencia al AD y el intervalo de predicción calibrado responden preguntas distintas: cobertura estructural frente a error predictivo. Un compuesto dentro del dominio puede seguir presentando gran error por activity cliffs, ruido experimental o mecanismos ausentes.
