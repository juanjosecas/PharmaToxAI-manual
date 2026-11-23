# Tutorial 02 — Dataset crudo hasta matriz molecular

Ejemplo autocontenido con moléculas pequeñas **ficticias** y un endpoint artificial. No se entrena ni valida ningún modelo. El objetivo es comprobar el encadenamiento de carga → auditoría → curación → EDA → featurización → partición de ejemplo → limpieza.

Requisitos: instalación de PharmaToxAI desde el código y un entorno con RDKit, NumPy y pandas. Ver [instalación](../introduccion/instalacion.md).

## 1. Dataset sintético

~~~python
from pathlib import Path
import pandas as pd
from pharmatoxai.data_io import load_data, save_data

Path("outputs").mkdir(exist_ok=True)
raw = pd.DataFrame({
    "ID": ["A", "B", "C", "D", "E", "F", "G", "H"],
    "SMILES": ["CCO", "OCC", "CCN", "CC(=O)O", "CCCO", "CCOC", "c1ccccc1", "c1ccncc1"],
    "Activity": [1.0, 1.2, 1.5, 1.4, 1.7, 1.8, 2.5, 2.2],
})
save_data(raw, "outputs/entrada.csv")
df = load_data("outputs/entrada.csv", sep=",")
assert len(df) == 8
~~~

Los valores de `Activity` no son mediciones ni permiten inferencias SAR.

## 2. Auditoría y curación

~~~python
from pharmatoxai.eda import basic_counts
from pharmatoxai.curation import PharmaToxCurator

print(basic_counts(df, "SMILES", "Activity"))
curator = PharmaToxCurator(keep_stereo=True, tautomers=True, neutral=False)
df["SMILES_clean"] = df["SMILES"].map(curator.curate)
rejected = df.loc[df["SMILES_clean"].isna()].copy()
valid = df.dropna(subset=["SMILES_clean"]).copy()
print("Estructuras inválidas:", len(rejected))
print("Identidades repetidas:", valid["SMILES_clean"].duplicated().sum())
~~~

Las representaciones `CCO` y `OCC` corresponden a la misma estructura. En un conjunto real debe decidirse cómo resolver ensayos repetidos **antes** de generar el split; en este ejemplo se utiliza una mediana solamente para ilustrar la operación, asumiendo endpoints comparables.

~~~python
from pharmatoxai.curation import deduplicate_smiles

curated = deduplicate_smiles(
    valid, smiles_col="SMILES_clean", target_col="Activity", method="median"
)
assert curated["SMILES_clean"].is_unique
print("Estructuras únicas:", len(curated))
~~~

## 3. EDA y featurización

~~~python
from pharmatoxai.eda import target_summary
from pharmatoxai.featurization import calculate_features

print(target_summary(curated, "Activity"))
features = calculate_features(
    curated, smiles_col="SMILES_clean",
    backend=["rdkit2d", "morgan"], verbose=False
)
assert len(features) == len(curated)
assert features["SMILES_clean"].notna().all()
print("Matriz:", features.shape)
~~~

`calculate_features` incorpora características generadas. Seleccionar como predictoras **solo** columnas numéricas de descriptores/fingerprints: excluir `Activity`, `ID` y cualquier columna derivada del target.

## 4. Ejemplo de separación y limpieza

Se usa una división explícita y determinista por posiciones para ilustrar la mecánica de preprocesamiento. **No es una estrategia de validación QSAR**, ni debe emplearse para estimar rendimiento o seleccionar modelos.

~~~python
import numpy as np
from pharmatoxai.feature_cleaning import (
    remove_nan_inf, impute_median_train, remove_low_variance
)

excluded = {"Activity", "ID"}
feature_cols = [
    c for c in features.select_dtypes(include="number").columns
    if c not in excluded
]
X = features[feature_cols].replace([np.inf, -np.inf], np.nan)
train = X.iloc[:-2].copy()
test = X.iloc[-2:].copy()

train = remove_nan_inf(train, max_nan_frac=0.3)
test = test[train.columns]
train, test = impute_median_train(train, test)
# Eliminar columnas sin dispersión, calculada solo en train.
keep = remove_low_variance(train, threshold=0.0).columns
train, test = train[keep], test[keep]

assert train.columns.equals(test.columns)
assert not train.isna().any().any()
assert not test.isna().any().any()
print("Train:", train.shape, "Test:", test.shape)
~~~

Con un dataset tan pequeño algunas características pueden quedar constantes o indefinidas. El tutorial usa filtros simples y no selecciona variables por actividad. Para datasets reales, consultar [limpieza](../preparacion/limpieza.md) y el capítulo de particiones del bloque de modelado.

## 5. Guardar resultados

~~~python
from pharmatoxai.data_io import save_data

save_data(curated, "outputs/curated.parquet")
save_data(features, "outputs/features.parquet")
save_data(train, "outputs/train_features.parquet")
save_data(test, "outputs/test_features.parquet")
~~~

Registrar adicionalmente el commit de PharmaToxAI, la versión RDKit y las políticas de curación. Si se usa otro dataset, verificar identidades, alineación de filas, unidades, censura y estructura de replicados antes de reutilizar el procedimiento.

**Estado del ejemplo:** redactado contra firmas del código en `main`; ejecución de integración no verificada en un entorno instalado.

**Referencias:** [documentos del Bloque A](../preparacion/index.md), [notebook original de EDA](https://github.com/juanjosecas/PharmaToxAI/blob/main/notebooks/01_Exploratory%20Data%20Analysis.ipynb) y [notebook 02 de featurización](https://github.com/juanjosecas/PharmaToxAI/blob/main/notebooks/02_Featurization_Cleaning_Splitting%20of%20Molecular%20Dataset.ipynb).
