# Evaluación, límites y comunicación

Evalúa el modelo con una estrategia que represente el uso previsto. La
validación interna mide consistencia bajo una partición definida; una prueba
externa e independiente aporta evidencia más fuerte sobre generalización.
Evita ajustar el modelo o elegir hiperparámetros mirando el conjunto externo.

## Métricas adecuadas al problema

- En regresión, informa MAE o RMSE con la unidad del endpoint; incluye \(R^2\)
  con cautela y explica la distribución del objetivo.
- En clasificación, informa sensibilidad, especificidad, precisión y
  balanced accuracy; PR-AUC puede ser útil cuando la clase positiva es rara.
- Indica prevalencia, intervalos o variación entre particiones y matriz de
  confusión en el umbral utilizado.

Las métricas de folds no son observaciones independientes. No las presentes
como intervalos de confianza sin un método estadístico justificado.

## Ámbito de aplicación e incertidumbre

Compara las moléculas de interés con la cobertura estructural y el rango de
endpoint del entrenamiento. Define qué significa estar fuera de dominio y qué
acción tomar ante predicciones inciertas. Una puntuación alta no demuestra que
una molécula nueva esté dentro del ámbito de aplicación.

Investiga duplicados, homólogos o estructuras relacionadas entre particiones,
fugas de variables, etiquetas conflictivas y sesgos de selección. Explica las
limitaciones de cualquier método de interpretación; una explicación local no
establece causalidad.

## Comunicar resultados

Incluye el uso previsto, procedencia de datos, curación, representación,
partición, versiones, métricas, incertidumbre, ámbito y limitaciones. Distingue
resultados computacionales de evidencia experimental y somete el modelo a
revisión experta antes de utilizarlo en decisiones de seguridad o regulatorias.

Este manual es educativo: no sustituye evaluación experimental, asesoramiento
profesional ni guías regulatorias vigentes. Consulta la
[bibliografía](../referencia/bibliografia.md) y los requisitos de la
jurisdicción correspondiente.
