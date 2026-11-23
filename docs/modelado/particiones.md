# Partición de datasets moleculares

**Módulo:** `pharmatoxai.splitting`. La estrategia depende del uso previsto: interpolación entre compuestos relacionados, predicción de nuevas series químicas o extrapolación temporal.

| Estrategia | Ventaja | Limitación |
| --- | --- | --- |
| Random | Baseline de referencia | Puede compartir análogos estrechos entre train y test |
| Stratified | Conserva proporciones de clases o rangos de actividad | No controla similitud estructural |
| Scaffold | Separa grupos estructurales | Distribuciones y tamaños pueden desequilibrarse |
| Scaffold-stratified | Intenta balancear actividad entre scaffolds | Requiere suficientes grupos y bins viables |
| Butina | Separa grupos de similitud | Depende del fingerprint y umbral |
| Kennard–Stone | Cubre regiones del espacio de descriptores | Depende de escalado, distancia y representación |
| Temporal | Simula predicción prospectiva | Requiere fechas fiables; no está implementado como estrategia específica del módulo |

## Ejemplo: baseline aleatorio

~~~python
from pharmatoxai.splitting import random_split

train_idx, test_idx, train_df, test_df = random_split(
    df, test_size=0.20, random_state=42
)
assert set(train_idx).isdisjoint(test_idx)
~~~

El resultado consta de **cuatro elementos**: índices de entrenamiento/test y sus respectivos DataFrames.

## Scaffold y moléculas acíclicas

El `scaffold_split` actual admite la columna de scaffolds y el SMILES original:

~~~python
from pharmatoxai.splitting import scaffold_split
from rdkit.Chem.Scaffolds import MurckoScaffold

df = df.copy().reset_index(drop=True)
# En producción conservar un ID original antes del reset.
df["Scaffold"] = df["SMILES_clean"].map(
    lambda s: MurckoScaffold.MurckoScaffoldSmiles(smiles=s)
    if isinstance(s, str) and s else None
)

train_idx, test_idx, train_df, test_df = scaffold_split(
    df,
    scaffold_col="Scaffold",
    smiles_col="SMILES_clean",
    acyclic_similarity_threshold=0.8,
    test_size=0.2,
    random_state=42,
)
~~~

En vez de poner todas las estructuras sin anillos en un único grupo, `scaffold_group_keys` calcula fingerprints Morgan y agrupa moléculas acíclicas mediante componentes conectados con similitud Tanimoto por encima del umbral. Este agrupamiento es transitivo: A~B y B~C pueden formar un solo grupo aunque A y C no superen directamente el umbral. Su coste para estructuras acíclicas es aproximadamente cuadrático. Los grupos permanecen completos, por lo que el tamaño efectivo de test puede desviarse de `test_size`.

**Distinción importante:** `scaffold_stratified_split` de regresión actualmente elimina registros con scaffold o target faltante; no debe suponerse que reutiliza el tratamiento de acíclicas de `scaffold_split`. No interpretar cualquiera de estos splits como una exigencia universal de OECD/ICH.

## Diagnóstico tras particionar

~~~python
assert len(train_df) + len(test_df) == len(df)
assert set(train_df.index).isdisjoint(set(test_df.index))
print("Train/test:", len(train_df), len(test_df))
print("Endpoint train:", train_df["Activity"].describe())
print("Endpoint test:", test_df["Activity"].describe())
~~~

Registrar además la distribución por scaffold, densidad de similitud interpartición, clase minoritaria, compuestos repetidos y, si corresponde, pares de activity cliffs repartidos entre particiones. Los grupos deben definirse antes de seleccionar features supervisadas.

En validación cruzada, conservar el principio estructural del holdout: un split scaffold externo acompañado de KFold aleatorio interno estima una tarea distinta.

**Fuente:** [splitting.py](https://github.com/juanjosecas/PharmaToxAI/blob/main/src/pharmatoxai/splitting.py).
