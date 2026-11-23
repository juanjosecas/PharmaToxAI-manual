# Fuentes externas: moléculas, propiedades y ensayos

**Módulos auditados:** [database_download.py](https://github.com/juanjosecas/PharmaToxAI/blob/main/src/pharmatoxai/database_download.py) y [moleculenet.py](https://github.com/juanjosecas/PharmaToxAI/blob/main/src/pharmatoxai/moleculenet.py). Los métodos de este capítulo realizan solicitudes de red, salvo los listados de catálogos. Los resultados dependen de la disponibilidad y versión de cada servicio.

## Elegir la fuente según la pregunta

| Fuente | Qué recupera PharmaToxAI | Identificador o entrada | Uso apropiado |
| --- | --- | --- | --- |
| [PubChem](https://pubchem.ncbi.nlm.nih.gov/) | Identidad, propiedades calculadas y resultados de BioAssay | Nombre, SMILES, CID, InChIKey, InChI; AID para bioensayos | Resolver estructuras y explorar ensayos |
| [ChEMBL](https://www.ebi.ac.uk/chembl/) | Metadatos de ensayos, actividades, estructuras | Palabras en descripción, CHEMBL assay ID y molecule ID | Construir datasets bioactivos con condiciones trazables |
| [TDC](https://tdcommons.ai/) | Datasets de benchmark ADME/toxicidad | Nombre de dataset del catálogo interno | Comparaciones reproducibles |
| [MoleculeNet](https://moleculenet.org/) mediante [DeepChem](https://deepchem.io/) | Datasets con SMILES y una o varias etiquetas | Nombre de dataset | Benchmarks, incluidos endpoints multitarea |
| [ZINC](https://zinc.docking.org/) | Catálogos de estructuras y propiedades de compuestos | Subset o ZINC ID | Bibliotecas químicas; **no** es un dataset de actividad experimental por sí mismo |
| [EPA DSSTox](https://comptox.epa.gov/dashboard/) | Mapeos DTXSID, CAS, nombres, InChIKey y SMILES | Descarga tabular | Resolver identidad y enlazar registros |
| [NCI Chemical Identifier Resolver](https://cactus.nci.nih.gov/chemical/structure) | Conversión entre nombre, CAS, SMILES y nombre IUPAC | Nombre, CAS o SMILES | Completar identificadores, sujeto a verificación |

**Diferenciar siempre:** propiedades calculadas, identificadores químicos, actividades experimentales, resultados de clasificación y datasets de benchmark. No mezclar endpoints de ensayos distintos solo porque tengan la misma unidad.

## PubChem: buscar y obtener propiedades

~~~python
from pharmatoxai.database_download import (
    search_pubchem_by_name,
    fetch_pubchem_compound_properties,
    fetch_pubchem_bioassay,
)

matches = search_pubchem_by_name("aspirin", max_records=5)
print(matches[["CID", "MolecularWeight"]].head())

properties = fetch_pubchem_compound_properties(
    ["aspirin", "caffeine"],
    namespace="name",
    properties=["MolecularFormula", "MolecularWeight", "XLogP", "TPSA"],
    request_delay=0.3,
)
print(properties)

# Reemplazar por un AID válido identificado en PubChem:
# assay = fetch_pubchem_bioassay(ASSAY_ID, activity_outcome="Active")
~~~

**Parámetros:** `namespace` admite `name`, `smiles`, `cid`, `inchikey`, `inchi`; `properties` define campos PUG-REST; `request_delay` pausa entre consultas; `max_records` limita coincidencias en búsqueda por nombre. `fetch_pubchem_bioassay` descarga registros por AID y opcionalmente filtra `Active`, `Inactive` o `Inconclusive`.

**Advertencia:** `fetch_pubchem_compound_properties` omite silenciosamente consultas fallidas y puede devolver varias filas por identificador. Auditar la columna `_query`, CIDs recuperados y consultas sin coincidencia. Los campos de SMILES disponibles pueden cambiar según la API; inspeccionar columnas y usar propiedades admitidas por el servicio.

## ChEMBL: desde una pregunta biológica hasta un dataset

El flujo implementado busca **descripciones de ensayos** y recupera sus actividades. No equivale a una búsqueda general por estructura o target.

~~~python
from pharmatoxai.database_download import (
    search_chembl_assays_by_description,
    fetch_chembl_activities_for_assays,
    add_chembl_smiles,
    filter_chembl_standard_type,
    add_chembl_nanomolar_values,
    add_chembl_pic50,
)

assays = search_chembl_assays_by_description(
    ["kinase inhibition"], deduplicate=True
)
print(assays[["assay_chembl_id", "description"]].head())

# Revisar manualmente el protocolo antes de seleccionar ensayos.
selected_ids = assays["assay_chembl_id"].dropna().head(3).tolist()
if selected_ids:
    activities = fetch_chembl_activities_for_assays(selected_ids)
    ic50 = filter_chembl_standard_type(activities, ["IC50"])
    ic50 = add_chembl_nanomolar_values(ic50)
    ic50 = add_chembl_pic50(ic50)
    ic50 = add_chembl_smiles(ic50)
    print(ic50.head())
~~~

Se necesita `chembl_webresource_client` para los métodos ChEMBL. `require_equal_relation=True` excluye por defecto actividades censuradas (`<`, `>`); `require_standard_value=True` descarta valores no numéricos. Esto facilita un dataset inicial, pero puede introducir sesgo por selección.

La conversión a nanomolar admite pM, nM, µM/uM y mM. Si la unidad no está reconocida, el valor convertido queda faltante. **`add_chembl_pic50` calcula (-\log_{10}(\mathrm{valor\ nM}\times10^{-9})) sin comprobar que `standard_type` sea IC50.** Aplicarla solo a IC50 válidas; para Ki u otros tipos usar el nombre de transformación correspondiente.

`build_chembl_pic50_dataset(assays_df, standard_types=["IC50"], add_smiles=True)` encadena recuperación, filtrado, unidades, transformación y unión de SMILES. No reemplaza la auditoría de ensayos ni verifica equivalencia entre protocolos.

### Extraer condiciones desde descripciones

~~~python
from pharmatoxai.database_download import add_chembl_assay_description_features
annotated = add_chembl_assay_description_features(assays)
print(annotated.filter(regex="concentration|time").head())
~~~

La extracción usa expresiones regulares para concentración y tiempo; son **candidatos** obtenidos de texto libre, no metadatos experimentales validados.

## TDC y MoleculeNet: datasets listos para explorar

~~~python
from pharmatoxai.database_download import list_tdc_datasets, fetch_tdc_dataset

catalog = list_tdc_datasets()
print([d for d in catalog if d["category"] == "Toxicity"])
ames = fetch_tdc_dataset("ames")
print(ames.head())
# split="train", "valid" o "test" usa el split ofrecido por el loader.
~~~

Requiere `PyTDC`. Los nombres aceptados están restringidos al catálogo codificado en PharmaToxAI, que incluye, entre otros, `ames`, `herg`, `caco2_wang`, `ld50_zhu`, `clintox`, `tox21` y `pampa_ncats`. Que un nombre figure en el catálogo **no garantiza** compatibilidad con una versión futura de PyTDC ni descarga exitosa.

~~~python
from pharmatoxai.moleculenet import load_moleculenet

tox21 = load_moleculenet("tox21")
print(tox21.shape, tox21.attrs)
print(tox21.isna().mean().sort_values(ascending=False).head())
~~~

Requiere DeepChem. `load_moleculenet` acepta `esol`, `freesolv`, `lipophilicity`, `bbbp`, `tox21`, `clintox`, `sider` y `hiv`. Devuelve un DataFrame sin particionar, con columna `SMILES`; convierte etiquetas de peso cero en NaN para **preservar tareas no medidas**. No imputar esos NA como negativos. El clasificador multilabel actual de PharmaToxAI no admite etiquetas faltantes: resolver esa limitación antes de entrenar.

## ZINC, DSSTox y conversión de identificadores

~~~python
from pharmatoxai.database_download import (
    list_zinc_subsets, fetch_zinc_compounds,
    fetch_zinc_compound_by_id, fetch_epa_dsstox,
    cas_to_info, name_to_info,
)

print(list_zinc_subsets())
# Requiere que el endpoint ZINC20 configurado siga disponible:
# zinc = fetch_zinc_compounds(subset="druglike", max_records=100)

# Descarga potencialmente grande; ejecutar solo si se necesita:
# dsstox = fetch_epa_dsstox(output_path="data/dsstox.csv")

print(name_to_info("aspirin"))
print(cas_to_info("50-78-2"))
~~~

`fetch_zinc_compounds` utiliza el dominio `zinc.docking.org` y rutas de ZINC20 codificadas; comprobar disponibilidad del servicio antes de una descarga masiva. `fetch_epa_dsstox` usa un ID de archivo Figshare fijo, por lo que **no garantiza la versión más reciente** de DSSTox. `is_valid_cas` comprueba formato y dígito verificador, no existencia ni correspondencia química.

Las funciones NCI CIR incluyen `batch_cas_to_info`, `batch_name_to_info`, `batch_smiles_to_info`, `name_column_to_smiles`, `name_column_to_cas`, `smiles_column_to_name`, `smiles_column_to_cas` y `cas_column_to_smiles`. Devuelven identificadores resueltos cuando el servicio responde; contrastar identidad, sales y estereoisómeros antes de fusionar tablas.

## Qué conservar en cada descarga

Guardar consulta, fecha UTC, URL o identificador de fuente, versión de biblioteca, número de registros, errores, IDs originales, estructura original, unidad, tipo de endpoint, relación (`=`, `<`, `>`), ensayo, condiciones y licencia. La ausencia de un registro no equivale a actividad negativa. Ver [construcción y auditoría de datasets](construccion-datasets.md).

## Visualizaciones de los datos recuperados

Distribución de propiedades PubChem; el código continúa el ejemplo que creó `properties`.

~~~python
import pandas as pd
import seaborn as sns
import matplotlib.pyplot as plt

sns.set_theme(style="whitegrid")
plot_df = properties.copy()
for col in ["MolecularWeight", "XLogP", "TPSA"]:
    if col in plot_df:
        plot_df[col] = pd.to_numeric(plot_df[col], errors="coerce")
cols = [c for c in ["MolecularWeight", "XLogP", "TPSA"] if c in plot_df]
if cols and not plot_df.empty:
    long = plot_df[cols].melt(var_name="Propiedad", value_name="Valor").dropna()
    if not long.empty:
        g = sns.displot(data=long, x="Valor", col="Propiedad",
                        col_wrap=3, facet_kws={"sharex": False, "sharey": False},
                        bins=15, height=3)
        g.set_titles("{col_name}")
        g.figure.suptitle("Propiedades recuperadas de PubChem", y=1.05)
        plt.show()
~~~

El histograma con dos o tres moléculas carece de valor estadístico; aumentar el tamaño del conjunto antes de interpretar distribuciones.

Para inspeccionar el desbalance de AMES (`ames` obtenido en el ejemplo TDC):

~~~python
import seaborn as sns
import matplotlib.pyplot as plt

if "Y" in ames.columns:
    fig, ax = plt.subplots(figsize=(6, 4))
    sns.countplot(data=ames.assign(Y=ames["Y"].astype(str)),
                  x="Y", order=ames["Y"].astype(str).value_counts().index, ax=ax)
    ax.set(xlabel="Etiqueta Y", ylabel="Registros", title="Distribución de clases: AMES")
    plt.tight_layout()
    plt.show()
~~~
