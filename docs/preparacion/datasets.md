# Datos, estructuras y curación

La calidad de un QSAR está limitada por la calidad y el ámbito de sus datos.
Conserva una copia inmutable del conjunto de origen y registra cada operación
de curación para poder rastrear las observaciones incluidas o excluidas.

## Esquema mínimo

Una tabla analítica debería poder vincular cada registro con:

| Campo | Descripción |
| --- | --- |
| `compound_id` | Identificador estable en el proyecto |
| `structure` | Estructura química de entrada y formato de origen |
| `endpoint` | Ensayo o propiedad objetivo, con protocolo |
| `value` / `unit` | Medición original y unidad explícita |
| `source` | Publicación, base de datos u origen autorizado |
| `curation` | Normalización, exclusiones y resolución de duplicados |

No sobrescribas silenciosamente la estructura ni la medida originales. Guarda
la transformación aplicada y el motivo de cada exclusión.

## Lista de control de curación

1. Comprueba que los identificadores y estructuras se puedan interpretar.
2. Define cómo tratar sales, mezclas, estereoquímica, tautómeros y cargas.
3. Convierte unidades con reglas documentadas y verifica rangos plausibles.
4. Identifica mediciones repetidas y conflictos entre ensayos.
5. Cuantifica valores ausentes y documenta cualquier imputación.
6. Revisa el balance, la cobertura química y la procedencia.
7. Conserva un registro reproducible de las decisiones.

La estandarización química puede cambiar la identidad representada. Elige las
reglas según el endpoint y el uso previsto, y evalúa su impacto antes de
aplicarlas en lote.

## Descriptores y fingerprints

Los descriptores resumen propiedades químicas; los fingerprints codifican
patrones estructurales en vectores. Registra la herramienta, versión,
parámetros y tratamiento de estructuras inválidas. Ajusta cualquier
transformación que aprenda de los datos —incluida la imputación o selección de
variables— exclusivamente con el conjunto de entrenamiento en cada partición.

El archivo de datos del tutorial contiene números **sintéticos**, no estructuras
ni resultados experimentales. Sirve únicamente para comprobar que la tubería
de análisis puede ejecutarse.
