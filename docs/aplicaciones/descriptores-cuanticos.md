# Descriptores cuánticos con xTB

**Módulo:** `pharmatoxai.qm_featurization`. Se obtienen características electrónicas a partir de una geometría 3D generada para la estructura seleccionada. No confundir el estado de protonación del SMILES con la carga y el número de electrones desapareados utilizados por el cálculo.

## Requisitos

Verificar el ejecutable externo antes de comenzar:

~~~bash
command -v xtb
xtb --version
~~~

La instalación de Python no garantiza que xTB esté en `PATH`. Registrar versión xTB, geometría inicial, carga total, UHF, solvente, método, optimización y criterios de convergencia.

## Una estructura

~~~python
from pharmatoxai.qm_featurization import calculate_xtb_features

features = calculate_xtb_features(
    "CCO",
    work_root="outputs/xtb",
    job_name="ethanol_demo",
    charge=0, uhf=0,
    gfn_level=2, do_opt=False,
    wbo=True, esp=False, fukui=False,
    n_threads=2,
)
if features is None:
    raise RuntimeError("Cálculo xTB fallido; revisar logs y archivos de trabajo")
print(features)
~~~

`do_opt=False` no significa que la geometría sea experimental ni relajada al nivel xTB. Para propiedades dependientes de la conformación se necesita definir conformeros, optimización y selección de geometrías.

## Por lotes con diagnóstico de fallos

~~~python
import pandas as pd
from pharmatoxai.qm_featurization import calculate_xtb_features_dataframe

df = pd.DataFrame({"SMILES": ["CCO", "CCN"]})
features_df = calculate_xtb_features_dataframe(
    df, smiles_col="SMILES",
    work_root="outputs/xtb_batch",
    default_charge=0, default_uhf=0,
    gfn_level=2, do_opt=False,
    keep_failed=True,
)
print(features_df.filter(regex="xtb_status|xtb_error").head())
~~~

Con `keep_failed=True` se preservan filas y causas de errores; por defecto pueden descartarse. Comprobar el número de estructuras procesadas y evitar interpretar missing por fallo computacional como ausencia de propiedad física.

### Coste y reproducibilidad

La generación de geometrías y cálculos xTB puede ser costosa: limitar el ejemplo inicial y registrar semilla del generador 3D, entorno, hilos y rutas. Las propiedades QM no son automáticamente comparables entre estados de protonación o multiplicidades distintos.

**Fuente:** [qm_featurization.py](https://github.com/juanjosecas/PharmaToxAI/blob/main/src/pharmatoxai/qm_featurization.py) y [documentación](https://github.com/juanjosecas/PharmaToxAI/blob/main/docs/qm_featurization.md).
