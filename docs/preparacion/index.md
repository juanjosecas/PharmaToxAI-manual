# Bloque A — Preparación molecular y exploración

Este bloque recorre desde los registros originales hasta una matriz de descriptores lista para aplicar una partición y un preprocesamiento controlado. Los capítulos se pueden consultar por separado.

| Etapa | Capítulo | Resultado esperado |
| --- | --- | --- |
| 1 | [Carga y auditoría de datos](carga-datos.md) | Dataset y esquema revisados |
| 2 | [EDA](eda.md) | Distribuciones, calidad y espacio de características |
| 3 | [Curación molecular](curacion.md) | Identidades normalizadas y exclusiones auditables |
| 4 | [Featurización](featurizacion.md) | Descriptores/fingerprints y metadatos |
| 5 | [Limpieza](limpieza.md) | Transformaciones reproducibles sin leakage |

La auditoría del endpoint y de la procedencia debe preceder al modelado. La curación química precede a la generación de fingerprints. La selección de variables aprendida debe realizarse **después de establecer train/validation/test**. Los gráficos de EDA sobre el dataset completo son exploratorios y no validan un modelo.

## Dependencias

Utilizar el entorno de [instalación](../introduccion/instalacion.md) y comprobarlo con la [verificación](../introduccion/verificacion.md). Para funciones y firmas, consultar el [mapa de fuentes](../referencia/fuentes-del-codigo.md). PharmaToxAI es actualmente un repositorio privado: para ejecutar ejemplos que lo importen se necesita acceso autorizado.

Los datos de demostración no constituyen evidencia experimental. Los ejemplos usan firmas contrastadas con el código fuente; **no se han ejecutado como test de integración del entorno de la persona lectora**.
