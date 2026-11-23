# Bloque B — Modelado y validación QSAR

El bloque comienza con una matriz curada y caracterizada del [Bloque A](../preparacion/index.md) y termina con una evaluación independiente, documentación del dominio de aplicabilidad y artefactos reproducibles.

| Capítulo | Producto |
| --- | --- |
| [Particiones](particiones.md) | Asignación de moléculas y diagnóstico de solapamiento |
| [Entrenamiento clásico](entrenamiento-qsar.md) | Artefactos de regresión y clasificación |
| [Deep learning](deep-learning.md) | Redes MLP y trazas de entrenamiento |
| [Evaluación](../evaluacion/metricas.md) | Métricas del holdout, CV y análisis de errores |
| [Dominio de aplicabilidad](../evaluacion/dominio-aplicabilidad.md) | Umbrales, indicadores y límites de interpretación |
| [Tutorial integrado](../tutoriales/03-modelado-validacion.md) | Ejecución con un endpoint sintético |

**Separación obligatoria:** el conjunto de test debe permanecer intacto durante preparación supervisada, ajuste, selección de hiperparámetros, calibración y elección de umbrales. El procesamiento aprendido en CV debe ajustarse en cada fold. Un benchmark aleatorio puede ser optimista ante análogos muy parecidos.

En PharmaToxAI, los campos `performance` de ciertos entrenadores están marcados como `training_resubstitution`: corresponden al entrenamiento y no acreditan generalización. Guardar commit, split, identificadores, semillas, versiones y configuración del modelo.

Documentación técnica de referencia: [splitting](https://github.com/juanjosecas/PharmaToxAI/blob/main/docs/splitting.md), [training](https://github.com/juanjosecas/PharmaToxAI/blob/main/docs/training.md) y [evaluation](https://github.com/juanjosecas/PharmaToxAI/blob/main/docs/evaluation.md).
