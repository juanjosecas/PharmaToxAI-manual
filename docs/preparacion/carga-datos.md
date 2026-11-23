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
