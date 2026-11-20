# Primer QSAR: definir la pregunta

QSAR (*quantitative structure–activity relationship*) relaciona una
representación de la estructura química con una propiedad o actividad medida.
Antes de elegir un algoritmo, define qué se predice, para qué moléculas y con
qué evidencia.

## Especifica el endpoint

Documenta el protocolo experimental, la unidad, las condiciones, el umbral de
actividad si se discretiza y el tratamiento de mediciones repetidas. No mezcles
endpoints que solo comparten un nombre pero difieren en ensayo, especie,
tiempo de exposición o unidad.

Para regresión, el conjunto etiquetado puede expresarse como
\(\{(x_i, y_i)\}_{i=1}^{n}\), donde \(x_i\) representa una molécula y \(y_i\)
su valor experimental. En clasificación, la etiqueta depende además de un
criterio explícito, por ejemplo \(y_i = 1\) si una medida supera un umbral
predefinido.

## Delimita el uso previsto

- ¿Qué decisión o priorización apoyará el modelo?
- ¿Qué estructuras, endpoints y rangos de actividad cubren los datos?
- ¿Qué errores son más costosos: falsos positivos o falsos negativos?
- ¿Qué evidencia experimental adicional se necesita antes de actuar?

## Comprueba los datos antes de modelar

Verifica identificadores, unidades, duplicados, valores faltantes, balance de
clases y consistencia química. Conserva la procedencia de las estructuras y
mediciones. El capítulo de [datos y curación](../preparacion/datasets.md)
propone una lista inicial.

El [tutorial reproducible](../tutoriales/01-primer-qsar.md) ilustra el flujo
con descriptores artificiales. No representa un endpoint toxicológico ni
demuestra rendimiento predictivo real.
