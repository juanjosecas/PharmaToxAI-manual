# Read-across cuantitativo

**Módulo:** `pharmatoxai.read_across`. El modelo `WeightedTanimotoReadAcross` busca análogos por Morgan/Tanimoto y calcula una media ponderada del endpoint, con opción de abstención ante soporte estructural insuficiente.

~~~python
from pharmatoxai.read_across import WeightedTanimotoReadAcross

reference_smiles = ["CCO", "CCCO", "CCCOC", "CCOC", "CCN"]
reference_y = [1.0, 1.3, 1.8, 1.5, 1.1]  # Endpoint ficticio

ra = WeightedTanimotoReadAcross(
    n_neighbors=3, radius=2, n_bits=2048,
    min_similarity=0.2, similarity_power=2,
    min_effective_neighbors=1,
)
ra.fit(reference_smiles, reference_y)
print(ra.predict(["CCCO"]))
print(ra.explain("CCCO"))
~~~

La explicación devuelve identidad del análogo, similitud, pesos y valores experimentales utilizados. Los valores del ejemplo son inventados; no representan ninguna propiedad fisicoquímica real.

## Criterios y evaluación

La predicción requiere al menos un análogo por encima de `min_similarity` y soporte efectivo suficiente. Si no se cumplen los criterios, la implementación puede lanzar `ValueError` y abstenerse. Un umbral bajo sirve para probar la mecánica, **no** demuestra confianza química.

En validación, evitar que una misma sustancia, tautómero o réplica figure en referencias y test. `leave_one_out_predict()` excluye el registro consultado, pero puede conservar otros registros equivalentes; deduplicar por identidad química antes de utilizarlo para estimar rendimiento.

El módulo también incluye utilidades para PubChem, ChEMBL y otras fuentes; sus consultas remotas requieren conectividad y sus resultados deben auditarse por endpoint, especies, condiciones y procedencia. Una similitud alta no garantiza actividad similar cuando existen activity cliffs.

**Fuente:** [read_across.py](https://github.com/juanjosecas/PharmaToxAI/blob/main/src/pharmatoxai/read_across.py).

## Gráfico de los análogos de referencia

Para el ejemplo sintético, mostrar la distribución del endpoint de referencia. El gráfico no mide precisión predictiva.

~~~python
import pandas as pd
import seaborn as sns
import matplotlib.pyplot as plt

analogos = pd.DataFrame({"SMILES": reference_smiles, "Endpoint": reference_y})
fig, ax = plt.subplots(figsize=(7, 4))
sns.barplot(data=analogos, x="SMILES", y="Endpoint", ax=ax)
ax.set(title="Valores de referencia para read-across", ylabel="Endpoint ficticio")
plt.tight_layout()
plt.show()
~~~
