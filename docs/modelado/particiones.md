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


## Fundamentos y parámetros de las particiones

Sea un dataset con \(n\) moléculas. La fracción de test indica un **objetivo de tamaño**:

$$
n_{\mathrm{test}}\approx \lfloor n\,f_{\mathrm{test}}\rfloor,\qquad
n_{\mathrm{train}}=n-n_{\mathrm{test}}
$$

Para un split aleatorio, cada fila puede asignarse independientemente, pero el resultado depende de la semilla. La reproducibilidad de random_state no corrige un diseño experimental inadecuado. Al mantener agrupaciones químicas completas, el tamaño real puede diferir del objetivo.

### Estratificación del endpoint

En regresión, stratified_split divide la actividad en cuantiles mediante qcut. Para cada bin \(b\), busca conservar aproximadamente su frecuencia relativa:

$$
\frac{n_{b,\mathrm{train}}}{n_{\mathrm{train}}}
\approx
\frac{n_{b,\mathrm{test}}}{n_{\mathrm{test}}}
$$

El parámetro n_bins es el número **solicitado** de intervalos, no necesariamente el número conseguido: qcut puede reducirlo si hay cuantiles repetidos. Demasiados bins producen estratos con menos observaciones que las necesarias para repartirlos; pocos bins ocultan heterogeneidad de la respuesta. Stratified splitting no garantiza separación de análogos.

### Agrupación por similitud estructural

Para fingerprints binarios \(A,B\), la similitud Tanimoto es

$$
T(A,B)=\frac{|A\cap B|}{|A\cup B|},\qquad d(A,B)=1-T(A,B)
$$

En scaffold_split, un scaffold Bemis–Murcko define un grupo de moléculas cíclicas. Para moléculas acíclicas sin scaffold, scaffold_group_keys utiliza fingerprints Morgan, radio 2 y 2048 bits, y agrupa por componentes conectados cuando \(T\geq\) acyclic_similarity_threshold. Con 0.8 se unen análogos que alcanzan 80 % de similitud bajo esa representación. Debido a transitividad, **no implica** que todos los pares de un mismo grupo superen 0.8. El coste de comparar \(m\) moléculas acíclicas crece aproximadamente como \(O(m^2)\).

En Butina, distance_cutoff es una **distancia**, no similitud. Su valor predeterminado 0.3 corresponde al criterio local \(d\leq0.3\), equivalente a \(T\geq0.7\) en Morgan/Tanimoto; el algoritmo y el armado de clusters no equivalen al procedimiento de componentes conectados de scaffold_split.

### Kennard–Stone: cobertura geométrica

La implementación actual estandariza los descriptores y comienza por el punto más próximo al centroide. Luego añade iterativamente una observación que maximiza su distancia mínima al conjunto seleccionado:

$$
i^*=\arg\max_{i\notin S}\min_{j\in S}\lVert z_i-z_j\rVert_2
$$

La función asigna los puntos seleccionados a **test**; esta elección de implementación merece documentarse porque otros flujos Kennard–Stone construyen primero el conjunto de calibración/entrenamiento. El escalado se estima sobre **todo el dataset antes de seleccionar**: sirve para diseño de partición basado en covariables, pero no debe confundirse con un escalador entrenado sin acceso al test para el modelo predictivo.

### Parámetros: efecto y diagnóstico

| Parámetro | Qué controla | Qué verificar |
| --- | --- | --- |
| test_size | Fracción objetivo del holdout | Tamaño final y cantidad de grupos completos |
| random_state | Reproducibilidad del azar | Sensibilidad entre semillas; no garantiza representatividad |
| n_bins | Resolución de estratificación | Número real de bins y observaciones por bin |
| acyclic_similarity_threshold | Agrupación de acíclicas por Tanimoto | Tamaño de componentes, puentes transitivos |
| radius, n_bits | Representación Morgan en Butina | Dependencia del criterio de similitud |
| distance_cutoff | Corte de distancia en Butina | No confundir con similitud Tanimoto |
| descriptor_cols | Espacio para Kennard–Stone | NaN, escala y dominio químico cubierto |

**Particiones diferentes estiman objetivos diferentes.** El rendimiento aleatorio suele aproximarse a interpolación química, mientras que el scaffold split mide extrapolación a otros esqueletos bajo las reglas de agrupamiento elegidas. Ninguno representa automáticamente generalización temporal o interensayo.

### Recomendación de diagnóstico

Guardar ID, grupo, método, semilla y asignación de cada registro. Verificar además la distribución del endpoint, la mínima distancia train–test, identidad química compartida y proporción de grupos grandes. Si se particiona a nivel de grupos, la CV interna también debe respetar grupos para estimar una tarea comparable.

## Gráfico de balance train/test

Continuación del ejemplo que define `train_df` y `test_df`. Para un endpoint continuo:

~~~python
import pandas as pd
import seaborn as sns
import matplotlib.pyplot as plt

plot_df = pd.concat([
    train_df[["Activity"]].assign(Partición="Train"),
    test_df[["Activity"]].assign(Partición="Test")
], ignore_index=True)
plot_df["Activity"] = pd.to_numeric(plot_df["Activity"], errors="coerce")
fig, ax = plt.subplots(figsize=(7, 4))
sns.histplot(data=plot_df.dropna(), x="Activity", hue="Partición",
             stat="density", common_norm=False, element="step",
             fill=False, bins=20, ax=ax)
ax.set(title="Distribución del endpoint por partición", xlabel="Activity")
plt.tight_layout()
plt.show()
~~~

Para endpoints categóricos sustituir por `sns.countplot`. Comparar también scaffolds compartidos y similitud máxima train–test; una distribución similar del endpoint no demuestra independencia química.
