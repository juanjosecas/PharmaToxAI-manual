# Curación molecular y estandarización

**Módulo:** `pharmatoxai.curation`. Separar la validez sintáctica de una representación SMILES de la identidad química relevante para el endpoint.

## Estandarizar una molécula

~~~python
from pharmatoxai.curation import PharmaToxCurator

curator = PharmaToxCurator(
    keep_stereo=True,
    remove_hs=True,
    tautomers=True,
    neutral=False,
)
for smiles in ["CCO", "C[C@H](O)C(=O)O", "not_a_smiles"]:
    print(smiles, "→", curator.curate(smiles))
~~~

Estas opciones controlan estereoquímica, hidrógenos, tautómeros y neutralización. El procedimiento puede modificar representación y composición de fragmentos. Mantener estructura original y reglas de curación utilizadas; no asumir que todos los contraiones o mezclas son irrelevantes para el ensayo.

## Curar un DataFrame

`standard_smiles` recibe un **DataFrame**, no un string, y utiliza procesos en paralelo. El ajuste explícito `n_jobs=1` reduce la sobrecarga en conjuntos pequeños.

~~~python
import pandas as pd
from pharmatoxai.curation import standard_smiles

df = pd.DataFrame({
    "sample_id": ["A", "B", "C"],
    "SMILES": ["CCO", "OCC", "CCN"],
    "Activity": [1.2, 1.4, 2.1],
})
curated = standard_smiles(
    df.copy(),
    smiles_col="SMILES",
    target_col="Activity",
    n_jobs=1,
    deduplicate=False,
)
print(curated[["SMILES", "SMILES_clean", "Activity"]])
~~~

Ejecutar scripts que utilizan multiprocessing desde archivos `.py` con `if __name__ == "__main__":` cuando lo exija el sistema operativo. Para notebooks y pocas moléculas, `PharmaToxCurator.curate` permite inspeccionar las transformaciones una a una.

## Deduplicación: decisión científica

~~~python
from pharmatoxai.curation import deduplicate_smiles

dedup = deduplicate_smiles(
    curated,
    smiles_col="SMILES_clean",
    target_col="Activity",
    method="median",
)
~~~

`deduplicate_smiles` determina identidad mediante estandarización RDKit y SMILES canónico. La mediana **solo** es una agregación admisible cuando las mediciones son comparables; no fusionar endpoints, tiempos, especies o sistemas experimentales diferentes. Antes de colapsar réplicas, cuantificar discrepancias, definir tolerancias y conservar el registro de origen.

## Control de calidad y exclusiones

Revisar, como mínimo, SMILES inválidos, fragmentos múltiples, variaciones de protonación, tautomería, estereoisomería, estructuras con metales, compuestos duplicados y moléculas con más de una medición. Comparar filas antes/después y emitir una tabla de motivos de exclusión. El orden de curación y la política de sales alteran la identidad química efectiva.

**Advertencia:** nunca calcular fingerprints antes de fijar la política de curación si se busca equivalencia estructural entre diferentes representaciones.

**Fuentes:** [curation.py](https://github.com/juanjosecas/PharmaToxAI/blob/main/src/pharmatoxai/curation.py) y [guía técnica](https://github.com/juanjosecas/PharmaToxAI/blob/main/docs/curation.md).
