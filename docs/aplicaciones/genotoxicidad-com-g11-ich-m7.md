# Evaluación de genotoxicidad mediante (Q)SAR: COM G11 e ICH M7

## Alcance y fuentes

La [guía COM G11 (23 de febrero de 2026)](https://www.gov.uk/government/publications/guidance-on-the-use-of-qsar-models-to-predict-genotoxicity/com-guidance-statement-g11-guidance-on-the-use-of-qsar-models-to-predict-genotoxicity) propone un proceso de **formulación del problema, evaluación (Q)SAR y evaluación regulatoria**. Recomienda modelos científicamente válidos, dominio de aplicabilidad pertinente, predicciones adecuadas para el propósito y documentación suficiente. La [ICH M7(R2)](https://www.ich.org/page/multidisciplinary-guidelines) trata **impurezas mutagénicas reactivas con ADN** y, para la evaluación (Q)SAR de mutagenicidad bacteriana, exige dos metodologías complementarias: una estadística y otra basada en reglas expertas.

**Distinguir endpoints:** Ames estima reversión mutacional bacteriana; no equivale a una evaluación integral de genotoxicidad (micronúcleos, aberraciones cromosómicas, aneugenicidad, etc.). Un modelo estadístico de Ames y una colección de SMARTS genéricos **no constituyen automáticamente** el par de modelos validado que requiere ICH M7. Tampoco un resultado negativo en Ames autoriza, por sí solo, clasificar una impureza como clase 5.

El flujo siguiente es **demostrativo y reproducible**; no es una evaluación regulatoria completa ni genera una conclusión ICH M7 automática. Consultar también [curación](../preparacion/curacion.md), [particiones](../modelado/particiones.md), [dominio de aplicabilidad](../evaluacion/dominio-aplicabilidad.md), [alertas](alertas.md), [read-across](read-across.md) e [informes QMRF](informes-qmrf.md).

## 1. Datos Ames: descarga y auditoría

El módulo `pharmatoxai.database_download` ofrece `fetch_tdc_dataset("ames")`. La distribución y la definición de etiquetas deben verificarse en el dataset descargado. No se asume que incluya protocolos, cepas, condiciones ±S9 o identificadores experimentales.

```python
import pandas as pd
import seaborn as sns
import matplotlib.pyplot as plt
from pharmatoxai.database_download import fetch_tdc_dataset

ames = fetch_tdc_dataset("ames")
required = {"Drug", "Y"}
if not required.issubset(ames.columns):
    raise ValueError(f"Columnas requeridas ausentes: {required - set(ames.columns)}")

ames = ames.dropna(subset=["Drug", "Y"]).copy()
ames["Y"] = pd.to_numeric(ames["Y"], errors="raise").astype(int)
if not set(ames["Y"].unique()).issubset({0, 1}):
    raise ValueError("El endpoint no es binario 0/1; revisar la codificación.")

print(ames.shape, ames["Y"].value_counts(dropna=False).to_dict())
sns.countplot(data=ames, x="Y")
plt.title("Distribución de etiquetas Ames")
plt.tight_layout()
plt.show()
```

**Auditoría indispensable:** procedencia y fecha de descarga, criterio de positivo/negativo, duplicados y resultados discordantes, sales, estereoisómeros, mezclas, controles, límites de extrapolación. Antes de un split, evitar que una misma identidad química aparezca en train y test.

## 2. Clasificador estadístico: protocolo mínimo

La siguiente celda muestra una línea base con RDKit y scikit-learn, deliberadamente explícita para controlar el preprocesamiento. **No es un modelo validado para ICH M7.** Los resultados dependerán de los datos descargados y del split; no se aportan métricas ficticias.

```python
import numpy as np
from rdkit import Chem
from rdkit.Chem import rdFingerprintGenerator
from sklearn.model_selection import GroupShuffleSplit
from sklearn.ensemble import RandomForestClassifier
from sklearn.metrics import classification_report, confusion_matrix
from pharmatoxai.visualization import fingerprint_similarity_matrix

gen = rdFingerprintGenerator.GetMorganGenerator(radius=2, fpSize=2048)
records = []
for smiles, label in zip(ames["Drug"], ames["Y"]):
    mol = Chem.MolFromSmiles(str(smiles))
    if mol is None:
        continue
    canonical = Chem.MolToSmiles(mol, isomericSmiles=True)
    fp = gen.GetFingerprintAsNumPy(mol)
    records.append((canonical, int(label), fp))

# En este ejemplo, los duplicados con etiquetas discordantes se excluyen
# para evitar imponer una agregación arbitraria.
tmp = pd.DataFrame([(s, y) for s, y, _ in records], columns=["SMILES", "Y"])
consistent = tmp.groupby("SMILES")["Y"].nunique()
allowed = set(consistent[consistent == 1].index)
unique = {}
for smiles, label, fp in records:
    if smiles in allowed and smiles not in unique:
        unique[smiles] = (label, fp)

smiles_all = np.array(list(unique))
y = np.array([unique[s][0] for s in smiles_all])
X = np.stack([unique[s][1] for s in smiles_all])

splitter = GroupShuffleSplit(n_splits=1, test_size=0.2, random_state=42)
train_idx, test_idx = next(splitter.split(X, y, groups=smiles_all))
clf = RandomForestClassifier(
    n_estimators=300, class_weight="balanced",
    random_state=42, n_jobs=-1
)
clf.fit(X[train_idx], y[train_idx])
pred = clf.predict(X[test_idx])
print(classification_report(y[test_idx], pred, zero_division=0))

cm = confusion_matrix(y[test_idx], pred, labels=[0, 1])
sns.heatmap(cm, annot=True, fmt="d", cmap="Blues",
            xticklabels=["Negativo", "Positivo"],
            yticklabels=["Negativo", "Positivo"])
plt.xlabel("Predicho")
plt.ylabel("Observado")
plt.title("Ames: matriz de confusión (holdout ilustrativo)")
plt.tight_layout()
plt.show()
```

**Limitación:** `GroupShuffleSplit` agrupado por SMILES canónico evita duplicados idénticos, pero **no** garantiza independencia de series químicas. Para una evaluación sólida, aplicar partición por scaffold o agrupamiento por similitud, comparar con partición aleatoria y mantener un test externo sin usarlo para seleccionar hiperparámetros. La matriz de confusión debe acompañarse de sensibilidad, especificidad, precisión, MCC, calibración y cobertura AD, con intervalos de confianza cuando corresponda.

## 3. Alertas estructurales: exploración, no sistema experto ICH M7

`calculate_alerts` incluye alertas de propiedades, carga, tamaño, SMARTS, *warheads* covalentes y núcleos estructurales. **No se verificó que constituya un sistema experto validado específicamente para mutagenicidad bacteriana.** Un `0` en estas alertas **no equivale** a predicción Ames negativa.

```python
from pharmatoxai.alerts import calculate_alerts

target_smiles = "CC(=O)Oc1ccccc1C(=O)O"  # ejemplo estructural; no impureza evaluada
alerts = calculate_alerts(target_smiles)
if alerts is None:
    raise ValueError("SMILES inválido")
active = {key: value for key, value in alerts.items() if value}
print("Alertas activadas:", active)
```

Para una evaluación ICH M7 registrar además el nombre, versión, cobertura, reglas activadas/desactivadas, mecanismo propuesto, referencias y justificación del **sistema experto independiente** utilizado. Una segunda predicción estadística no sustituye automáticamente esa metodología.

## 4. Dominio de aplicabilidad (AD)

`applicability_domain` de PharmaToxAI utiliza proyección PCA y criterios T², Mahalanobis robusto y kNN. El ejemplo siguiente muestra un control **estructural complementario** con el mismo fingerprint del clasificador; no reemplaza la evaluación AD completa del modelo.

```python
from rdkit import DataStructs

query = Chem.MolFromSmiles(target_smiles)
if query is None:
    raise ValueError("SMILES de consulta inválido")
query_fp = gen.GetFingerprint(query)
train_fps = [gen.GetFingerprint(Chem.MolFromSmiles(s)) for s in smiles_all[train_idx]]
similarities = np.asarray(DataStructs.BulkTanimotoSimilarity(query_fp, train_fps))
nearest = np.argsort(similarities)[::-1][:5]
analogues = pd.DataFrame({
    "SMILES": smiles_all[train_idx][nearest],
    "Ames_Y": y[train_idx][nearest],
    "Tanimoto": similarities[nearest],
})
print(analogues.to_string(index=False))

sns.barplot(data=analogues, x="Tanimoto", y="SMILES", hue="Ames_Y",
            orient="h")
plt.xlim(0, 1)
plt.title("Análogos de entrenamiento más próximos")
plt.tight_layout()
plt.show()
```

El máximo Tanimoto no define por sí mismo un AD validado. Comparar también fragmentos no representados, metabolismo, mecanismo de reactividad, vecindad de respuesta y errores observados en análogos. Si el modelo queda fuera del AD, registrar **indeterminado** y solicitar revisión experta, sin convertirlo automáticamente en positivo o negativo.

## 5. Read-across y búsqueda externa de análogos

PharmaToxAI dispone de `pubchem_similarity_search`, `chembl_similarity_search`, `chembl_molecule_activities` y `WeightedTanimotoReadAcross`. El regresor ponderado está orientado a **endpoints continuos**: no utilizarlo directamente para clasificar Ames binario.

```python
from pharmatoxai.read_across import pubchem_similarity_search

# Requiere conexión a PubChem; puede devolver solo CID, sin puntuación
# individual de similitud ni evidencia Ames.
hits = pubchem_similarity_search(
    target_smiles, namespace="smiles", threshold=85, max_records=20
)
print(hits.head())
```

La evidencia de un análogo debe vincularse con **un estudio Ames concreto**, identidad química, calidad del ensayo, activación metabólica, concordancia estructural y diferencias potencialmente relevantes. La mera presencia en PubChem o ChEMBL no constituye evidencia de mutagenicidad.

## 6. Discordancias y revisión experta

| Modelo estadístico | Sistema experto validado | AD | Tratamiento documental |
|---|---|---|---|
| Negativo | Negativo | Dentro de ambos | Revisar calidad, cobertura y condiciones del uso regulatorio; no asignar clase automáticamente. |
| Positivo | Positivo | Dentro | Revisar alertas, plausibilidad mecanística y evidencia experimental. |
| Positivo | Negativo | Cualquiera | Discordancia: analizar análogos, reglas, entrenamiento y posibles falsos positivos/negativos. |
| Negativo | Positivo | Cualquiera | Discordancia: inspeccionar alerta, metabolismo y contraejemplos experimentales. |
| Cualquiera | Indeterminado/no cubierto | Fuera o incierto | Revisión experta documentada; posible evidencia adicional o ensayo. |
| Sin predicción | Cualquiera | Fuera | No asignar una etiqueta binaria por defecto. |

La clasificación final de impurezas ICH M7 depende también de datos experimentales, relación con el API, naturaleza de la impureza y contexto regulatorio. **La tabla es un esquema de revisión, no una tabla automática de asignación de clases ICH M7.**

## 7. Registro de evidencia individual: borrador QPRF

`generate_qmrf_report` documenta el **modelo** (QMRF); no genera automáticamente un QPRF regulatorio por **sustancia y predicción**. Para mantener trazabilidad, el siguiente registro JSON conserva campos de entrada, resultados y revisión sin inventar valores. Los `None` deben completarse con evidencia antes de emitir un informe.

```python
import json
from datetime import date
from pathlib import Path

qprf_draft = {
    "document_type": "QPRF_DRAFT_NOT_REGULATORY",
    "date": date.today().isoformat(),
    "substance": {
        "input_smiles": target_smiles,
        "canonical_smiles": Chem.MolToSmiles(query, isomericSmiles=True),
        "identity_verified": False,
        "identifier": None,
        "impurity_context": None,
    },
    "endpoint": "Bacterial reverse mutation (Ames)",
    "statistical_model": {
        "name": "RandomForest illustrative baseline",
        "model_version": None,
        "dataset_provenance": "TDC Ames; verify dataset version",
        "prediction": int(clf.predict(gen.GetFingerprintAsNumPy(query).reshape(1, -1))[0]),
        "positive_probability": float(clf.predict_proba(
            gen.GetFingerprintAsNumPy(query).reshape(1, -1)
        )[0, list(clf.classes_).index(1)]),
        "validated_for_ich_m7": False,
    },
    "expert_rule_based_model": {
        "name": None, "version": None, "prediction": None,
        "triggered_rules": [], "validated_for_ich_m7": None,
    },
    "applicability_domain": {
        "statistical_model_status": "not_established",
        "expert_model_status": "not_established",
        "nearest_training_tanimoto": float(similarities.max()),
    },
    "analogue_evidence": analogues.to_dict(orient="records"),
    "structural_screening": {
        "pharmatoxai_alerts": active,
        "is_validated_ames_expert_system": False,
    },
    "expert_review": {
        "reviewer": None,
        "assessment": None,
        "discordances_resolved": None,
        "references": [],
    },
    "regulatory_conclusion": None,
    "limitations": [
        "Demonstration only; not an ICH M7-compliant assessment",
        "No independent expert rule-based Ames prediction",
        "No validated applicability domain for this query",
    ],
}
Path("qprf_borrador.json").write_text(
    json.dumps(qprf_draft, indent=2, ensure_ascii=False), encoding="utf-8"
)
```

**Importante:** el campo `positive_probability` es una salida de Random Forest, no una probabilidad calibrada ni una medida de confianza regulatoria. El JSON es una **plantilla de trazabilidad**, no un QPRF oficial completo.

## 8. Matriz de cobertura: funciones existentes frente a requisitos

| Elemento | Disponible en PharmaToxAI | Qué falta acreditar o desarrollar |
|---|---|---|
| Datos Ames | Descarga TDC y herramientas de curación | Procedencia experimental, calidad, conflictos y trazabilidad por ensayo. |
| QSAR estadístico | Clasificación y evaluación; ejemplo RF anterior | Validación externa por dominio químico, calibración y adecuación al propósito ICH M7. |
| Alertas SMARTS | `calculate_alerts`, `calculate_alerts_dataframe` | **Sistema experto de mutagenicidad Ames validado** y justificación mecanística por regla. |
| Dos metodologías complementarias | Se pueden combinar resultados manualmente | Evidencia de que se utilizaron un modelo estadístico y otro basado en reglas, ambos apropiados. |
| AD | T², Mahalanobis, kNN, herramientas de similitud | Cobertura estructural/mecanística, validación de umbrales y evidencia por molécula. |
| Read-across | Búsqueda PubChem/ChEMBL; regresor de similitud | Selección crítica de análogos con datos Ames experimentales y revisión de mecanismos. |
| Discordancias | Salidas y herramientas de inspección | Procedimiento documentado de revisión experta, reglas de decisión y firmas. |
| QMRF | `generate_qmrf_report` | Revisión humana y documentación específica del modelo, versiones y datos. |
| QPRF individual | Registro JSON ilustrativo en este capítulo | Generador QPRF conforme a la plantilla pertinente, evidencias verificables y aprobación experta. |
| Asignación de clase ICH M7 | No implementada como decisión regulatoria | Contexto de impureza, datos experimentales y revisión regulatoria integral. |

**Estado de verificación:** se comprobó la existencia de las APIs mencionadas en el repositorio; no se ejecutaron descargas, entrenamiento, llamadas a PubChem ni validaciones regulatorias durante la redacción.

## Referencias principales

- [UK COM (2026). Guidance Statement G11: Guidance on the use of (Q)SAR models to predict genotoxicity](https://www.gov.uk/government/publications/guidance-on-the-use-of-qsar-models-to-predict-genotoxicity/com-guidance-statement-g11-guidance-on-the-use-of-qsar-models-to-predict-genotoxicity).
- [ICH M7(R2): Assessment and control of DNA reactive (mutagenic) impurities in pharmaceuticals](https://www.ich.org/page/multidisciplinary-guidelines).
- [OECD (Q)SAR Assessment Framework](https://www.oecd.org/en/topics/sub-issues/assessment-of-chemicals/quantitative-structure-activity-relationships-qsars.html).
