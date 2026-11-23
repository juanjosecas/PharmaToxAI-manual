# Carga de datos y auditoría inicial

**Módulo:** `pharmatoxai.data_io`. **Entrada:** archivos tabulares con una columna de estructuras y, cuando corresponda, endpoints. **Salida:** tabla inspeccionada sin pérdidas silenciosas.

## 1. Esquema mínimo

Para un conjunto QSAR, registrar al menos: identificador original, estructura (SMILES), valor y unidad del endpoint, condiciones del ensayo, fuente y, si existen, réplicas y censura (`<`, `>`). No mezclar IC50, pIC50, porcentajes de inhibición y valores de ensayos distintos como si fueran una misma variable.

~~~python
from pathlib import Path
import pandas as pd
from pharmatoxai.data_io import load_data, save_data, validate_dataframe

path = Path("data/compounds.csv")
df = load_data(path, sep=",")  # El separador explícito evita inferencias incorrectas.
validate_dataframe(df)
required = {"SMILES", "Activity"}
missing = required - set(df.columns)
if missing:
    raise ValueError(f"Columnas faltantes: {sorted(missing)}")

print(df.shape)
print(df.dtypes)
print(df[["SMILES", "Activity"]].isna().sum())
print("Filas duplicadas:", df.duplicated().sum())
~~~

`load_data` admite CSV, TSV, Excel, Parquet, Pickle y HDF5; el módulo incluye lectores adicionales para otras fuentes. Utilizar `sheet_name` con Excel cuando la hoja sea relevante. **Pickle solo debe cargarse de fuentes confiables** porque deserializarlo puede ejecutar código.

## 2. Tipos, unidades y faltantes

~~~python
import pandas as pd

raw = df["Activity"]
value = pd.to_numeric(raw, errors="coerce")
unparsed = raw.notna() & value.isna()
print("Valores no numéricos:", df.loc[unparsed, ["SMILES", "Activity"]].head(20))
print("NA reales:", raw.isna().sum())
print("Valores no interpretables:", int(unparsed.sum()))
~~~

No convertir automáticamente símbolos de censura como `>100` a `100`. Separar operador y valor, y definir el tratamiento estadístico acorde con el ensayo. Verificar unidades antes de transformar concentraciones.

## 3. Identificadores y duplicados

Las filas duplicadas no equivalen necesariamente a moléculas duplicadas: una misma estructura puede representar varios ensayos, lotes, condiciones o mediciones independientes.

~~~python
print("Filas:", len(df))
print("SMILES únicos textuales:", df["SMILES"].nunique(dropna=True))
print("Repeticiones textuales:", df["SMILES"].duplicated(keep=False).sum())
~~~

Las variantes SMILES de una misma molécula se reconocerán en [curación](curacion.md). Conservar una copia de los registros crudos y un identificador estable para auditar exclusiones.

## 4. Persistencia

~~~python
from pathlib import Path
from pharmatoxai.data_io import save_data

Path("outputs").mkdir(exist_ok=True)
save_data(df, "outputs/tabla_auditada.parquet")
~~~

Parquet preserva tipos tabulares y suele ser apropiado para matrices grandes; CSV facilita el intercambio, pero pierde parte de la información de tipos y metadatos.

**Control de salida:** cantidad de filas, columnas, porcentaje de NA por variable, identificadores repetidos, unidades y procedencia de cada endpoint.

**Referencia técnica:** [data_io.py](https://github.com/juanjosecas/PharmaToxAI/blob/main/src/pharmatoxai/data_io.py) y [documentación](https://github.com/juanjosecas/PharmaToxAI/blob/main/docs/data_io.md).


## Fundamentos: unidad estadística, censura y escala del endpoint

Una fila puede representar una sustancia, una medición, una réplica o un ensayo. La observación se describe como \((s_i,a_i,c_i,y_i)\): estructura, ensayo, condiciones y respuesta. Tener igual estructura no hace equivalentes dos mediciones realizadas bajo diferentes condiciones.

### Censura y unidades

Los resultados pueden ser exactos, censurados a izquierda (menores que un límite), censurados a derecha (mayores que un límite) o acotados por un intervalo. Una observación expresada como «>100» no es una medida exacta de 100: guardar por separado operador, límite y unidad.

Para concentraciones molares positivas:

$$
\mathrm{pIC}_{50}=-\log_{10}\left(\frac{\mathrm{IC}_{50}}{1\ \mathrm{mol\,L^{-1}}}\right)
$$

Una IC50 de \(1\ \mu\mathrm{M}=10^{-6}\ \mathrm{mol\,L^{-1}}\) equivale a pIC50 = 6. La transformación invierte la ordenación (menor IC50 corresponde a mayor pIC50). No se debe aplicar a porcentajes de inhibición ni a IC50 con unidades sin normalizar.

### Completitud

Para \(n\) filas y la variable \(j\), su fracción de datos ausentes es

$$
m_j=\frac{1}{n}\sum_{i=1}^n\mathbf{1}(x_{ij}\ {\rm faltante}),\qquad c_j=1-m_j
$$

La completitud no identifica el mecanismo de ausencia: MCAR, MAR y MNAR requieren supuestos distintos. El módulo valida que exista una tabla no vacía, pero no verifica automáticamente unidades, calidad del ensayo, SMILES ni semántica del endpoint.

### Agregación de réplicas

Para \(r\) medidas comparables:

$$
\bar y=\frac{1}{r}\sum_{k=1}^{r}y_k,\qquad
s_y=\sqrt{\frac{\sum_k(y_k-\bar y)^2}{r-1}}
$$

La media es sensible a valores extremos; la mediana ofrece robustez. Si las observaciones provienen de distintos lotes o ensayos, una agregación directa puede ocultar efectos experimentales. Conservar identificadores de réplica, procedencia y dispersión; resolver las réplicas antes del split para evitar la presencia de mediciones relacionadas en train y test.

### Parámetros de lectura

| Parámetro | Qué controla | Qué revisar |
| --- | --- | --- |
| sep | Separador CSV/TSV | Una selección incorrecta puede crear columnas erróneas |
| encoding | Codificación | Símbolos de unidades y caracteres especiales |
| dtype, como argumento de pandas | Tipado | IDs con ceros iniciales y columnas mixtas |
| sheet_name | Hoja de Excel | Que se haya leído el ensayo correcto |
| validate | Chequeo de tabla mínima | No equivale a validación científica |

La detección automática de separadores en el código actual compara coma, punto y coma, tabulación y barra vertical, contando las columnas en las primeras cinco líneas. Este método heurístico puede fallar en textos con delimitadores embebidos.

### Auditoría compacta

~~~python
audit = pd.DataFrame({
    "dtype": df.dtypes.astype(str),
    "missing_fraction": df.isna().mean(),
    "n_unique": df.nunique(dropna=True),
})
print(audit.sort_values("missing_fraction", ascending=False))
~~~
