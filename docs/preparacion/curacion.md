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


## Fundamentos químicos de la estandarización

La normalización transforma una cadena SMILES \(s\) según una política \(\theta\):

$$
s_{\mathrm{curado}}=C(s;\theta)
$$

Múltiples SMILES pueden representar un mismo grafo molecular. Sin embargo, diferentes tautómeros, estados protónicos, mezclas y estereoisómeros pueden conservar diferencias científicamente relevantes. La definición de identidad depende del endpoint.

### Orden de operaciones y opciones

PharmaToxCurator.curate aplica desconexión de metales, selección del fragmento mayor, limpieza/normalización, neutralización opcional, reionización, canonización tautomérica opcional, eliminación opcional de H explícitos y gestión estereoquímica.

| Parámetro | Valor por defecto | Interpretación |
| --- | --- | --- |
| keep_stereo | True | Si False, elimina quiralidad tetraédrica R/S pero conserva E/Z |
| remove_hs | True | Elimina H explícitos; no significa quitar carga o protonación |
| tautomers | True | Selecciona un tautómero canónico según reglas, no la especie necesariamente más poblada |
| neutral | False | No activa Uncharger en esa etapa; puede haber reionización después |
| n_jobs de standard_smiles | -1 | Solicita todos los núcleos; afecta coste y uso de recursos |
| chunksize | 1000 | Tamaño de lotes; afecta paralelización, no equivalencia química |

La elección del fragmento mayor puede eliminar contraiones y otros componentes. No es adecuada universalmente para sales, complejos metálicos, mezclas o formulaciones.

### Ionización y tautomería

Para un ácido monoprótico:

$$
\mathrm{pH}=\mathrm{p}K_a+\log_{10}\frac{[\mathrm{A^-}]}{[\mathrm{HA}]},
\qquad f_{\mathrm{A^-}}=\frac{1}{1+10^{\mathrm{p}K_a-\mathrm{pH}}}
$$

Una representación neutral no garantiza que esa forma predomine al pH experimental. Para tautómeros \(T_1\rightleftharpoons T_2\):

$$
K=\frac{[T_2]}{[T_1]}=\exp\left(-\frac{\Delta G^\circ}{RT}\right)
$$

Un tautómero canónico es una elección informática, no un cálculo termodinámico de poblaciones.

### Política de duplicados y estereoquímica

**Diferencia verificada en el código:** deduplicate_smiles() calcula identidad utilizando el fragmento mayor, Uncharger y **eliminación de quiralidad tetraédrica**, preservando E/Z. Puede fusionar enantiómeros incluso si la primera curación usó keep_stereo=True. Esto es problemático para endpoints mediados por receptores quirales: evaluar el criterio antes de agregar resultados.

Para réplicas comparables, la dispersión se puede describir mediante

$$
CV=\frac{s_y}{|\bar y|},\qquad
MAD=\mathrm{mediana}_i\left|y_i-\mathrm{mediana}(y)\right|
$$

El CV deja de ser estable si la media se aproxima a cero. max_target_std en deduplicate_smiles impone un límite de desviación estándar **en unidades del endpoint**, no un porcentaje universal. No mezclar ensayos ni escalas de actividad.

### Control de invariancia

Un objetivo práctico es que una curación estable sea idempotente:

$$
C(C(s;\theta);\theta)=C(s;\theta)
$$

Esta propiedad necesita tests sobre el conjunto químico de interés: no está garantizada para todas las estructuras y versiones. Auditar filas descartadas, cambios en carga, componentes eliminados y diferencias entre estereoisómeros.
