# Manual de PharmaToxAI

Manual práctico y progresivo para aplicar métodos QSAR a problemas de seguridad
química. Empieza por [la guía de instalación](docs/introduccion/instalacion.md)
y continúa con el [primer QSAR](docs/introduccion/primer-qsar.md), la
[preparación de datos](docs/preparacion/datasets.md), el
[entrenamiento](docs/modelado/entrenamiento.md) y la
[evaluación](docs/evaluacion/validacion.md).

El manual se publica con MkDocs Material y GitHub Pages. Consulta el
[contenido completo](docs/index.md) o sigue el [tutorial reproducible](docs/tutoriales/01-primer-qsar.md).

## Desarrollo local

Requiere Python 3.10 o posterior:

```bash
python -m venv .venv
source .venv/bin/activate
python -m pip install -r requirements-docs.txt
mkdocs serve
```

Para comprobar la compilación:

```bash
mkdocs build --strict
```

Los ejemplos con datos sintéticos son didácticos y no describen compuestos ni
resultados experimentales reales. Cada tutorial que use PharmaToxAI deberá
indicar una versión publicada y distribuible del paquete; este repositorio no
incluye código interno ni datos de distribución restringida.

Las predicciones QSAR son evidencia computacional, no asesoramiento médico ni
una evaluación regulatoria por sí solas. Consulta las limitaciones y
responsabilidades en el capítulo de [evaluación y validación](docs/evaluacion/validacion.md).
