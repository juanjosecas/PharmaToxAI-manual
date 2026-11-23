# Construcción de datasets QSAR a partir de fuentes públicas

Este capítulo continúa [Fuentes y búsqueda](fuentes-y-busqueda.md) y conecta con [carga y auditoría](../preparacion/carga-datos.md), [curación](../preparacion/curacion.md) y [particiones](../modelado/particiones.md).

## Esquema recomendado

| Campo | Propósito |
| --- | --- |
| source, source_record_id | Procedencia del registro |
| query, retrieval_utc | Reproducibilidad de consulta |
| molecule_id, assay_id | Identidad y contexto experimental |
| smiles_original, smiles_curated | Estructura recibida y estructura estandarizada |
| endpoint_type | IC50, Ki, logP, LD50, clasificación, etc. |
| endpoint_value, endpoint_unit | Valor y unidad originales |
| relation | Exacto, límite superior/inferior |
| endpoint_transformed | Escala usada para modelar |
| organism, assay_description, conditions | Comparabilidad de ensayos |
| replicate_id, quality_flag | Replicación y calidad |

## Proceso recomendado

1. **Definir endpoint** antes de buscar: matriz biológica, especie, método, unidad, dirección de la respuesta y condiciones relevantes.
2. **Recuperar y congelar datos crudos**, con IDs y metadatos. Guardar tanto los éxitos como los fallos de búsqueda.
3. **Auditar estructura y relaciones:** una respuesta `IC50 > 100 µM` está censurada; no convertirla en 100 µM exactos.
4. **Normalizar unidades**, sin mezclar IC50 con Ki, EC50, porcentajes o resultados binarios.
5. **Curar SMILES y revisar duplicados**, preservando estereoquímica cuando sea relevante. El deduplicador actual puede agrupar enantiómeros.
6. **Agrupar réplicas compatibles** conservando dispersión y protocolo. No promediar entre ensayos heterogéneos sin justificación.
7. **Definir splits y grupos químicos** antes de seleccionar características o hiperparámetros. Si el proveedor aporta particiones oficiales de benchmark, conservarlas para comparabilidad.
8. **Registrar auditoría final:** exclusiones, tamaños, prevalencia, rangos, estructuras inválidas, NA y procedencia.

### Conversión de IC50 a pIC50

Para IC50 estrictamente positiva en nanomolar:

$$
\mathrm{pIC}_{50}=9-\log_{10}(\mathrm{IC}_{50}[\mathrm{nM}])
$$

No aplicar esta fórmula a valores con `standard_relation` distinta de `=`, ni a valores no positivos.

~~~python
import numpy as np
import pandas as pd

def add_pic50_checked(df):
    out = df.copy()
    value = pd.to_numeric(out["value_nM"], errors="coerce")
    valid = (
        out["standard_type"].eq("IC50")
        & out["standard_relation"].eq("=")
        & value.gt(0)
    )
    out["pIC50"] = np.nan
    out.loc[valid, "pIC50"] = 9 - np.log10(value.loc[valid])
    return out
~~~

### Auditoría tras descargar

~~~python
def audit_dataset(df, id_col=None, smiles_col="SMILES"):
    report = {
        "rows": len(df),
        "columns": len(df.columns),
        "duplicate_rows": int(df.duplicated().sum()),
        "missing_by_column": df.isna().sum().to_dict(),
    }
    if id_col in df.columns:
        report["duplicate_ids"] = int(df[id_col].duplicated().sum())
    if smiles_col in df.columns:
        report["missing_smiles"] = int(df[smiles_col].isna().sum())
    return report
~~~

Este conteo de SMILES no valida la estructura química. Aplicar después RDKit y la curación del Bloque A.

## Riesgos específicos por fuente

**PubChem:** los sinónimos pueden resolver múltiples CIDs; BioAssay puede incluir resultados de screening y readouts no equivalentes. **ChEMBL:** múltiples medidas por molécula, ensayos, organismos y relaciones de censura. **TDC/MoleculeNet:** datasets agregados con definiciones y splits que deben conservarse; tareas multitarget con faltantes. **ZINC:** bibliotecas estructurales sin etiquetas de actividad. **DSSTox/CIR:** resolución de identidad no implica medición experimental de un endpoint.

## Dependencias y comprobaciones

~~~bash
python -m pip install chembl_webresource_client PyTDC
# DeepChem se instala por separado según su compatibilidad con el entorno.
python -m pip show pharmatoxai chembl_webresource_client PyTDC deepchem
~~~

Los tests de [database_download.py](https://github.com/juanjosecas/PharmaToxAI/blob/main/scripts/test_database_download.py) usan mocks HTTP; validan procesamiento de respuestas simuladas, **no** disponibilidad real de endpoints ni vigencia de URLs.

**Código:** [database_download.py](https://github.com/juanjosecas/PharmaToxAI/blob/main/src/pharmatoxai/database_download.py), [moleculenet.py](https://github.com/juanjosecas/PharmaToxAI/blob/main/src/pharmatoxai/moleculenet.py).
