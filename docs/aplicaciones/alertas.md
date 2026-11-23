# Alertas estructurales y nitrosaminas

**Módulos:** `pharmatoxai.alerts` y `pharmatoxai.nitrosamine`. Se calculan indicadores de reglas fisicoquímicas, cargas, tamaño y patrones SMARTS. Su interpretación depende de contexto, endpoint y propiedades químicas.

## Alertas por estructura

~~~python
import pandas as pd
from pharmatoxai.alerts import calculate_alerts, calculate_alerts_dataframe

single = calculate_alerts("CCO")
print(single)
df = pd.DataFrame({"SMILES_clean": ["CCO", "c1ccccc1", "invalid"]})
flags = calculate_alerts_dataframe(df, smiles_col="SMILES_clean")
print(flags)
~~~

El DataFrame devuelve un subconjunto resumido de indicadores; para detalles de motivos reactivos, comprobar resultados de `calculate_alerts()` y funciones específicas como `smarts_alerts()` y `covalent_warhead_alerts()`. Las entradas inválidas reciben faltantes en la salida tabular.

## Nitrosaminas

~~~python
from pharmatoxai.nitrosamine import analyze_nitrosamine, analyze_nitrosamine_list

print(analyze_nitrosamine("CN(N=O)C"))
results = analyze_nitrosamine_list(["CN(N=O)C", "CCO"])
print(results)
~~~

Una coincidencia estructural o puntuación heurística **no equivale a una clasificación toxicológica validada**, ni determina por sí sola la potencia carcinogénica. Para una evaluación real se necesitan identidad, condiciones de exposición, datos toxicológicos, evidencia de metabolismo y reglas regulatorias aplicables.

## Buenas prácticas

Registrar versión de las reglas/SMARTS, política de protonación/tautomería, conteo de coincidencias y motivo de cada flag. Comparar representaciones alternativas cuando una normalización pueda eliminar el grupo observado.

**Fuentes:** [alerts.py](https://github.com/juanjosecas/PharmaToxAI/blob/main/src/pharmatoxai/alerts.py) y [nitrosamine.py](https://github.com/juanjosecas/PharmaToxAI/blob/main/src/pharmatoxai/nitrosamine.py).
