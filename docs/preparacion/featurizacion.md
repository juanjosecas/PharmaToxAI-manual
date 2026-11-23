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


## Fundamentos de las representaciones moleculares

La featurización transforma el grafo curado \(G_i\) en un vector:

$$
\phi:G_i\mapsto x_i\in\mathbb{R}^{p},\qquad
X=[x_1^\top;\ldots;x_n^\top]
$$

El algoritmo solo puede aprender de la información retenida por \(\phi\). Descriptores 2D, fingerprints, geometría 3D y propiedades QM codifican propiedades distintas, con costes y supuestos diferentes.

### Descriptores fisicoquímicos

La masa molecular suele expresarse en g/mol, TPSA en unidades de superficie y las variables como logP son adimensionales. Para la especie neutra, el coeficiente de reparto octanol/agua se define como

$$
\log P=\log_{10}\frac{c_{\mathrm{octanol}}}{c_{\mathrm{agua}}}
$$

No equivale a logD a pH específico, que depende de ionización. Descriptores computados no sustituyen directamente datos experimentales.

### Morgan: radio, bits y hashing

Los fingerprints circulares codifican entornos atómicos refinados iterativamente hasta un radio \(r\). Un identificador de entorno activa una posición en un vector binario de tamaño \(m\):

$$
b_j=\mathbf{1}(\text{algún entorno se asigna al bit }j),\quad j=1,\ldots,m
$$

| Parámetro | Significado |
| --- | --- |
| radius=2 | Entornos a hasta dos enlaces del átomo central; diámetro nominal cuatro (ECFP4) |
| n_bits=2048 | Dimensión y riesgo de colisiones por hashing |
| fp_type="morgan" | Huella circular de entornos atómicos |
| fp_type="maccs" | Claves de subestructuras predefinidas; RDKit produce 167 bits |
| Política de curación | Define qué grafo y qué cargas reciben los algoritmos |

Una colisión ocurre cuando entornos distintos activan el mismo bit. Aumentar n_bits reduce ciertas colisiones esperadas, pero no garantiza mejores métricas predictivas.

### Tanimoto/Jaccard y densidad

Para conjuntos de bits activos \(A,B\):

$$
T(A,B)=\frac{|A\cap B|}{|A\cup B|}=\frac{c}{a+b-c},
\qquad d_J=1-T
$$

\(a\) y \(b\) son números de bits activos; \(c\), bits compartidos. Cuando ambos vectores están vacíos, la expresión no está definida: calculate_jaccard_distance() **devuelve 1.0** como convención de implementación.

La densidad de un fingerprint binario es

$$
\rho_i=\frac{1}{m}\sum_{j=1}^{m}b_{ij}
$$

fingerprint_bit_density() calcula la media por fila: pasar **solo columnas binarias**, sin target u otros descriptores.

### Dimensionalidad y representación

El cociente \(p/n\) resume dimensionalidad relativa, pero no determina por sí solo la generalización. Un fingerprint de 2048 bits con pocos compuestos exige regularización y validación cuidadosa. La distancia euclidiana entre descriptores escalados y Tanimoto sobre bits comparan geometrías diferentes.

Los descriptores electrónicos xTB dependen también de conformación, carga, estado de espín, método y solvente. Compararlos con variables 2D requiere registrar esa configuración y la política de fallos. Una similitud estructural alta no implica similitud farmacológica: pueden existir activity cliffs.
