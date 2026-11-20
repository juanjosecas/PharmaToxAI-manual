# Tutorial 01: un baseline QSAR reproducible

Este ejercicio recorre la carga de datos, el ajuste de un baseline y una
validación cruzada. Los valores de
[`datasets/synthetic_qsar.csv`](https://github.com/juanjosecas/PharmaToxAI-manual/blob/main/datasets/synthetic_qsar.csv)
son descriptores y respuestas **artificiales**: no corresponden a moléculas ni
a ensayos reales. El resultado no estima el desempeño de PharmaToxAI ni tiene
interpretación toxicológica.

## Preparación

Desde la raíz del repositorio, instala las dependencias descritas en
[instalación](../introduccion/instalacion.md). El notebook puede ejecutarse en
Jupyter o validarse automáticamente durante la compilación del sitio:

```bash
python -m pip install -r requirements-docs.txt
```

## Ejecución

Abre el
[notebook `01_primer_qsar.ipynb`](https://github.com/juanjosecas/PharmaToxAI-manual/blob/main/notebooks/01_primer_qsar.ipynb)
desde la raíz del repositorio y ejecuta sus celdas en orden (por ejemplo, con
`jupyter lab notebooks/01_primer_qsar.ipynb`). El notebook evalúa una
regresión Ridge mediante validación cruzada y reporta errores de raíz cuadrada
de error cuadrático medio (RMSE). No hay datos externos, estructuras químicas
ni etiquetas de seguridad en este ejercicio.

## Para continuar con datos reales

Antes de adaptar el ejemplo, define el endpoint y el uso previsto, verifica que
los datos pueden redistribuirse, documenta la curación y diseña particiones
acordes con la pregunta científica. El tutorial no implementa funciones de la
API de PharmaToxAI: cada ejemplo futuro que lo haga deberá especificar una
versión pública exacta en la que se haya comprobado.
