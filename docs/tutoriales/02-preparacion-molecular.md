# Tutorial 02 — Pipeline A: estructuras → curación → featurización → split

**Continúa desde el [mapa del pipeline](01-primer-qsar.md).** Las salidas se consumen sin modificaciones en el [tutorial 03](03-modelado-validacion.md). Todos los bloques se ejecutan en orden desde la raíz del manual.

## 1. Generar y auditar el dataset

Se emplean SMILES de moléculas pequeñas y un endpoint **artificial** calculado con RDKit. Esto permite verificar la infraestructura sin descargar datos ni atribuir significado biológico a los resultados.

~~~python
from pathlib import Path
import json
import numpy as np
import pandas as pd
from rdkit import Chem, rdBase
from rdkit.Chem import Descriptors
from pharmatoxai.data_io import save_data, load_data

OUT = Path("outputs/qsar_pipeline")
OUT.mkdir(parents=True, exist_ok=True)
smiles = [
    "CCO", "CCCO", "CCCCO", "CC(C)O", "CCOC", "CCOCC",
    "CCN", "CCCN", "CCCCN", "CC(C)N", "CCNC", "CCNCC",
    "CC(=O)O", "CCC(=O)O", "CCCC(=O)O", "CC(=O)OC",
    "CC(=O)OCC", "CC(=O)N", "CCC(=O)N", "CC(C)=O",
    "c1ccccc1", "Cc1ccccc1", "CCc1ccccc1", "Oc1ccccc1",
    "COc1ccccc1", "Nc1ccccc1", "O=C(O)c1ccccc1",
    "c1ccncc1", "Cc1ccncc1", "Oc1ccncc1",
    "c1ccoc1", "c1ccsc1", "C1CCCCC1", "OC1CCCCC1",
    "C1CCNCC1", "N1CCCCC1", "O1CCCCC1", "CC1CCCCC1",
    "CC(=O)Nc1ccccc1", "CCOc1ccccc1",
]
rows = []
for i, smi in enumerate(smiles):
    mol = Chem.MolFromSmiles(smi)
    assert mol is not None, smi
    # Etiqueta determinista y artificial: no es una propiedad experimental.
    y = 0.015 * Descriptors.MolWt(mol) - 0.009 * Descriptors.TPSA(mol)
    rows.append({"ID": f"M{i:03d}", "SMILES": smi, "Activity": float(y)})
raw = pd.DataFrame(rows)
save_data(raw, str(OUT / "raw.csv"))
df = load_data(str(OUT / "raw.csv"), sep=",")
assert {"ID", "SMILES", "Activity"} <= set(df.columns)
print(df.shape, df.isna().sum().to_dict())
~~~

## 2. Curación molecular y control de identidades

~~~python
from pharmatoxai.curation import PharmaToxCurator, deduplicate_smiles

curator = PharmaToxCurator(keep_stereo=True, tautomers=True, neutral=False)
df["SMILES_clean"] = df["SMILES"].map(curator.curate)
invalid = df.loc[df["SMILES_clean"].isna()].copy()
valid = df.dropna(subset=["SMILES_clean", "Activity"]).copy()
curated = deduplicate_smiles(
    valid, smiles_col="SMILES_clean",
    target_col="Activity", method="median"
).reset_index(drop=True)
assert curated["SMILES_clean"].is_unique
assert len(curated) >= 20
print("Rechazadas:", len(invalid), "Identidades únicas:", len(curated))
curated.to_csv(OUT / "curated.csv", index=False)
~~~

En un estudio real, conservar resultados discordantes, información de ensayos y políticas de neutralización/tautomería. La agregación por mediana solo es defendible para mediciones comparables del mismo endpoint.

## 3. EDA y gráficos integrados

~~~python
import matplotlib.pyplot as plt
import seaborn as sns
from pharmatoxai.eda import target_summary
from pharmatoxai.visualization import visualize_descriptor_distribution

print(target_summary(curated, "Activity"))
sns.histplot(data=curated, x="Activity", kde=True)
plt.title("Distribución del endpoint artificial")
plt.tight_layout()
plt.show()

fig, axes = visualize_descriptor_distribution(
    curated["SMILES_clean"].tolist()
)
plt.show()
~~~

## 4. Featurización con PharmaToxAI

~~~python
from pharmatoxai.featurization import calculate_features

features = calculate_features(
    curated, smiles_col="SMILES_clean",
    backend=["rdkit2d", "morgan"], verbose=False
)
assert len(features) == len(curated)
features.to_csv(OUT / "features.csv", index=False)
print("Features:", features.shape)
~~~

La matriz incluye columnas originales. **Excluir identificadores y target** de las predictoras; revisar también columnas de metadatos numéricos para evitar leakage.

## 5. Split antes de ajustar limpieza

Para este conjunto demostrativo, el split es aleatorio y fijo. En datos reales contrastar scaffold/cluster, fechas, lotes y series químicas; el test no participa en ninguna decisión posterior.

~~~python
from pharmatoxai.splitting import random_split

train_idx, test_idx, train_df, test_df = random_split(
    features, test_size=0.25, random_state=42
)
assert set(train_idx).isdisjoint(set(test_idx))
train_df.to_csv(OUT / "train_raw.csv", index=False)
test_df.to_csv(OUT / "test_raw.csv", index=False)
(OUT / "split.json").write_text(json.dumps({
    "method": "random_split", "random_state": 42,
    "train_index": [int(i) for i in train_idx],
    "test_index": [int(i) for i in test_idx],
    "rdkit_version": rdBase.rdkitVersion,
}, indent=2), encoding="utf-8")

plot_df = pd.concat([
    train_df.assign(Conjunto="Train"),
    test_df.assign(Conjunto="Test"),
], ignore_index=True)
sns.histplot(data=plot_df, x="Activity", hue="Conjunto",
             element="step", stat="density", common_norm=False)
plt.title("Endpoint por partición")
plt.tight_layout()
plt.show()
~~~

## 6. Verificación y contrato con el siguiente tutorial

~~~python
for filename in ["raw.csv", "curated.csv", "features.csv",
                 "train_raw.csv", "test_raw.csv", "split.json"]:
    assert (OUT / filename).exists(), filename
print("Preparación completa:", OUT.resolve())
~~~

**Salida:** estructuras auditadas, matriz calculada y partición congelada. La imputación, selección por faltantes y eliminación de constantes se harán **solo con train** en el tutorial siguiente. No se reportan métricas predictivas en esta etapa.
