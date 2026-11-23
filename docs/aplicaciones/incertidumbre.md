# Intervalos de predicción y calibración

**Módulo:** `pharmatoxai.uncertainty`. Un intervalo de predicción para un endpoint numérico exige observaciones de calibración separadas del ajuste del modelo.

## Split conformal

~~~python
from pharmatoxai.uncertainty import (
    fit_split_conformal, apply_regression_uncertainty,
)

# y_cal y pred_cal proceden de un conjunto de calibración independiente.
artifact = fit_split_conformal(
    y_cal, pred_cal, confidence=0.90,
)
intervals = apply_regression_uncertainty(
    pred_test, artifact,
)
print(intervals[["prediction", "lower", "upper"]])
~~~

La calibración usa residuos absolutos para construir un intervalo de anchura constante; los métodos escalados o por bins requieren artefactos y supuestos adicionales. La cobertura objetivo es marginal bajo condiciones como intercambiabilidad, no una probabilidad individual de que una molécula concreta tenga el valor observado dentro del intervalo.

## Diseño experimental y diagnóstico

Separar train, calibration y test antes de ajustar parámetros. No calibrar con predicciones sobre entrenamiento ni reutilizar test para elegir método o confianza. Evaluar cobertura y anchura en test, incluyendo estratos por actividad, scaffold y dominio de aplicabilidad.

Un cambio de distribución entre calibración y aplicación puede invalidar la cobertura esperada. Un intervalo estrecho puede ser poco fiable fuera de dominio estructural.

**Fuente:** [uncertainty.py](https://github.com/juanjosecas/PharmaToxAI/blob/main/src/pharmatoxai/uncertainty.py).
