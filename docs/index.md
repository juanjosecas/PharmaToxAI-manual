# Manual de PharmaToxAI

Este manual presenta un flujo QSAR desde la pregunta científica hasta la
interpretación de resultados. Está dirigido a quienes conocen química o
toxicología y necesitan diseñar, validar y comunicar modelos de forma
reproducible.

## Recorrido de aprendizaje

1. [Instala el paquete](introduccion/instalacion.md), [elige un entorno](introduccion/entornos.md) y [comprueba la instalación](introduccion/verificacion.md).
2. [Define la pregunta y el endpoint](introduccion/primer-qsar.md).
3. Recorre el [Bloque A — Preparación molecular y exploración](preparacion/index.md): carga, EDA, curación, featurización y limpieza.
4. Desarrolla el [Bloque B — Modelado y validación](modelado/index.md): particiones, entrenamiento clásico y redes neuronales.
5. Evalúa [métricas](evaluacion/metricas.md) y [dominio de aplicabilidad](evaluacion/dominio-aplicabilidad.md) sobre test externo.
6. Ejecuta el [tutorial con datos sintéticos](tutoriales/01-primer-qsar.md).

Para entrenar con NVIDIA, consultar [configuración GPU](introduccion/gpu.md). La [correspondencia entre capítulos y fuentes del código](referencia/fuentes-del-codigo.md) documenta el material técnico reutilizable para las próximas entregas.

Los [módulos de aplicaciones e integraciones (Bloque C)](aplicaciones/index.md) desarrollan persistencia, inferencia, multilabel, read-across, alertas, xTB, incertidumbre y QMRF. El [tutorial 04](tutoriales/04-persistencia-inferencia-qmrf.md) continúa el de modelado.

## Alcance

La primera edición cubre un flujo QSAR supervisado básico. Los capítulos
posteriores podrán ampliar este recorrido a descriptores y fingerprints,
clasificación multilabel, ADMET, dominio de aplicabilidad, incertidumbre,
explicabilidad y requisitos regulatorios.

La documentación técnica de la API pertenece a la referencia del software; el
manual la complementará con métodos y casos reproducibles, sin duplicarla.
Cada ejemplo que se integre con PharmaToxAI deberá declarar una versión
distribuible concreta. No se presupone que una instalación del paquete esté
disponible públicamente.

!!! warning "Uso responsable"
    Un modelo QSAR depende de la calidad y el ámbito de sus datos. Sus
    predicciones no son evidencia experimental, consejo médico ni una decisión
    regulatoria autónoma. Revisa la [validación](evaluacion/validacion.md)
    antes de interpretar cualquier resultado.
