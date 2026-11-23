# Featurización molecular

**Módulo:** `pharmatoxai.featurization`. Una representación depende de la estructura curada y de las opciones de generación: registrarlas como parte del modelo.

## Descriptores RDKit

~~~python
import pandas as pd
from pharmatoxai.featurization import calculate_descriptors_dataframe

df = pd.DataFrame({
    "SMILES_clean": ["CCO", "CCN", "c1ccccc1"],
    "Activity": [1.0, 1.4, 2.0],
})
features = calculate_descriptors_dataframe(
    df, smiles_col="SMILES_clean", rdkit_all=False
)
print(features.shape)
print(features.columns[:12].tolist())
~~~

`rdkit_all=True` habilita el conjunto completo de descriptores 2D de RDKit. Algunas variables pueden contener NaN o infinitos; no sustituirlos arbitrariamente sin registrar el procedimiento.

## Interfaz unificada

~~~python
from pharmatoxai.featurization import calculate_features

features = calculate_features(
    df,
    smiles_col="SMILES_clean",
    backend=["rdkit2d", "morgan"],
    verbose=False,
)
print(features.shape)
~~~

Los backends aceptados actualmente son `rdkit2d`, `rdkit_full`, `mordred`, `padel`, `morgan` y `maccs`. La interfaz prefija nombres para distinguir fuentes de características. Revisar entradas inválidas y filas ausentes después de cualquier merge.

## Fingerprints

~~~python
from pharmatoxai.featurization import generate_fingerprints_dataframe

fp = generate_fingerprints_dataframe(
    df,
    smiles_col="SMILES_clean",
    fp_type="morgan",
    radius=2,
    n_bits=2048,
)
cols = [c for c in fp.columns if c.startswith("MorganFP_")]
assert len(cols) == 2048
print(fp[cols].shape)
~~~

En la API actual el parámetro es `fp_type`, **no** `type`. Morgan radius=2 identifica entornos atómicos hasta dos enlaces; el vector de 2048 bits aplica hashing y admite colisiones. MACCS utiliza un conjunto fijo de claves y no es directamente intercambiable con Morgan.

## Mordred, PaDEL y representaciones avanzadas

- **Mordred:** amplia biblioteca de descriptores; auditoría de valores faltantes y descriptores indefinidos.
- **PaDEL:** requiere `padelpy` y una instalación funcional de Java.
- **xTB:** se documentará separadamente como cálculo de propiedades dependiente de geometría, protonación, estado electrónico y configuración computacional.
- **Molfeat/embeddings:** módulos opcionales; las dependencias y versiones pueden diferir del entorno básico.

No concatenar representaciones sin comprobar escalas, duplicidad de columnas, identificación de filas y cambios de dimensionalidad. Compararlas mediante el **mismo split externo**; seleccionar la mejor representación mirando el test introduce sesgo.

## Entregables

Tabla de características alineada con identificadores, manifiesto de descriptores (backend, opciones, versión RDKit), matriz de faltantes y recuento de filas descartadas. El siguiente paso es la [limpieza de características](limpieza.md).

**Fuentes:** [featurization.py](https://github.com/juanjosecas/PharmaToxAI/blob/main/src/pharmatoxai/featurization.py), [documentación](https://github.com/juanjosecas/PharmaToxAI/blob/main/docs/featurization.md), [notebook 02](https://github.com/juanjosecas/PharmaToxAI/blob/main/notebooks/02_Featurization_Cleaning_Splitting%20of%20Molecular%20Dataset.ipynb).
