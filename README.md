# Manual de PharmaToxAI

Manual práctico y progresivo para aplicar métodos QSAR a problemas de seguridad
química. Empieza por [la guía de instalación](docs/introduccion/instalacion.md)
con la [gestión de entornos](docs/introduccion/entornos.md), la [configuración GPU](docs/introduccion/gpu.md) y la [verificación](docs/introduccion/verificacion.md). Continúa con el [primer QSAR](docs/introduccion/primer-qsar.md), el
[Bloque A de preparación molecular y exploración](docs/preparacion/index.md), el
[entrenamiento](docs/modelado/entrenamiento.md) y la
[evaluación](docs/evaluacion/validacion.md).

La [guía de fuentes del código](docs/referencia/fuentes-del-codigo.md) vincula cada capítulo con módulos, documentación y notebooks del proyecto original. Los capítulos se amplían de forma modular y sus ejemplos deben verificarse contra una revisión concreta del código.

El manual se publica con MkDocs Material y GitHub Pages. Consulta el
[contenido completo](docs/index.md) o sigue el [tutorial reproducible](docs/tutoriales/01-primer-qsar.md).

## Publicación en GitHub Pages

Un administrador debe activar GitHub Pages en
[Settings → Pages](https://github.com/juanjosecas/PharmaToxAI-manual/settings/pages)
y seleccionar **GitHub Actions** como fuente en **Build and deployment**.
Después, ejecuta de nuevo el workflow **Publish manual** desde la pestaña
**Actions** o espera al siguiente cambio en `main`. Sin esta configuración,
el job `deploy` falla con un error 404.

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
