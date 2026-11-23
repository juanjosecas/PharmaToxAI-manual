# Tutorial 04 — Pipeline C: persistencia → inferencia → trazabilidad

**Requiere los [tutoriales 02](02-preparacion-molecular.md) y [03](03-modelado-validacion.md).** Se vuelve a ajustar el estimador con los **mismos datos y semilla** del tutorial 03 para disponer de un objeto `model` en esta sesión. En un flujo productivo, ejecutar los tutoriales 03 y 04 en el mismo proceso o cargar el estimador guardado.

## 1. Recuperar entradas y reglas de preprocesamiento

~~~python
from pathlib import Path
import json
import joblib
import numpy as np
import pandas as pd

OUT = Path("outputs/qsar_pipeline")
train_df = pd.read_csv(OUT / "train_raw.csv")
test_df = pd.read_csv(OUT / "test_raw.csv")
X_train = pd.read_csv(OUT / "train_model.csv")
X_test = pd.read_csv(OUT / "test_model.csv")
y_train = train_df["Activity"]
y_test = test_df["Activity"]
prep = joblib.load(OUT / "preprocessing.joblib")
split_info = json.loads((OUT / "split.json").read_text())
assert list(X_train.columns) == prep["selected_columns"]
assert list(X_test.columns) == prep["selected_columns"]
~~~

## 2. Ajustar el mismo estimador y guardar el paquete

~~~python
from pharmatoxai.training import train_qsar_regression
from pharmatoxai.model_persistence import save_model, load_model

result = train_qsar_regression(
    X_train, y_train, model_type="random_forest",
    tune=False, run_rfe=False, cv=3, random_state=42, n_jobs=1
)
model = result["model"]
save_model(
    model, X_train, X_test, y_train, y_test,
    output_path=str(OUT / "models"),
    model_name="qsar_demo_rf",
    feature_names=list(X_train.columns),
    task_type="regression",
    metadata={
        "dataset": "artificial_rdkit_example",
        "seed": 42,
        "preprocessing_file": "outputs/qsar_pipeline/preprocessing.joblib",
        "split_file": "outputs/qsar_pipeline/split.json",
    },
)
model_dir = OUT / "models" / "qsar_demo_rf"
loaded = load_model(str(model_dir), load_data=False, verbose=False)
np.testing.assert_allclose(
    model.predict(X_test), loaded["model"].predict(X_test),
    rtol=1e-8, atol=1e-10
)
~~~

La serialización `joblib`/`pickle` debe cargarse exclusivamente desde archivos confiables. El preprocesamiento se guarda por separado; conservarlo junto con el modelo al distribuir el paquete.

## 3. Inferencia reproducible sobre features existentes

~~~python
from pharmatoxai.inference import predict

pred = np.asarray(predict(
    str(model_dir), X_new=X_test,
    return_proba=False, verbose=False
)).ravel()
np.testing.assert_allclose(
    pred, model.predict(X_test), rtol=1e-8, atol=1e-10
)
print("Paridad de predicción verificada:", len(pred))
~~~

## 4. Inferencia desde nuevas estructuras

Se aplica la misma curación, el mismo backend y las reglas **aprendidas en train**. El ejemplo usa una estructura no incluida entre las moléculas del tutorial 02.

~~~python
from pharmatoxai.curation import PharmaToxCurator
from pharmatoxai.featurization import calculate_features

new = pd.DataFrame({"ID": ["Q001"], "SMILES": ["CC(C)CO"]})
curator = PharmaToxCurator(keep_stereo=True, tautomers=True, neutral=False)
new["SMILES_clean"] = new["SMILES"].map(curator.curate)
if new["SMILES_clean"].isna().any():
    raise ValueError("SMILES inválido tras curación")
new_features = calculate_features(
    new, smiles_col="SMILES_clean",
    backend=["rdkit2d", "morgan"], verbose=False
)
missing = set(prep["raw_feature_names"]) - set(new_features.columns)
if missing:
    raise ValueError(f"Faltan descriptores: {sorted(missing)[:10]}")
new_X = new_features[prep["filtered_columns"]].replace(
    [np.inf, -np.inf], np.nan
)
new_X = new_X.fillna(prep["medians"]).fillna(0.0)
new_X = new_X[prep["selected_columns"]]
new_pred = predict(str(model_dir), X_new=new_X,
                   return_proba=False, verbose=False)
print("Predicción artificial:", new_pred)
~~~

La predicción no tiene interpretación biológica. **No** recalcular medianas, filtros, escalado ni AD con las moléculas nuevas.

## 5. Registrar QMRF del modelo

~~~python
from pharmatoxai.generate_qmrf import generate_qmrf_report, export_qmrf

report = generate_qmrf_report(
    model=loaded["model"],
    X_train=X_train, y_train=y_train,
    X_test=X_test, y_test=y_test, y_pred=pred,
    train_idx=split_info["train_index"],
    test_idx=split_info["test_index"],
    model_name="QSAR demostrativo",
    endpoint="Activity (respuesta artificial, sin unidades)",
    task_type="regression",
    extra={
        "dataset": "artificial_rdkit_example",
        "split": "random_holdout",
        "source_commit": "REGISTRAR_COMMIT_REAL",
        "preprocessing": str(OUT / "preprocessing.joblib"),
        "limitations": "No validación experimental ni uso regulatorio",
    },
)
export_qmrf(report, OUT / "qmrf_demo.json")
assert (OUT / "qmrf_demo.json").exists()
~~~

El generador QMRF recopila campos y métricas, pero **no acredita** por sí mismo los cinco principios OECD. Un QPRF describe la fiabilidad de una predicción individual; no debe confundirse con el QMRF del modelo.

## 6. Auditoría final

~~~python
import sklearn
from rdkit import rdBase

manifest = {
    "seed": 42,
    "n_train": len(X_train),
    "n_test": len(X_test),
    "feature_columns": X_train.columns.tolist(),
    "scikit_learn": sklearn.__version__,
    "rdkit": rdBase.rdkitVersion,
    "outputs": sorted(p.name for p in OUT.iterdir()),
    "external_validation": False,
    "experimental_endpoint": False,
}
(OUT / "manifest.json").write_text(
    json.dumps(manifest, indent=2, ensure_ascii=False), encoding="utf-8"
)
print("Pipeline terminado:", OUT.resolve())
~~~

Para trasladarlo a un proyecto real, sustituir el dataset artificial por mediciones auditadas, definir la estrategia de partición antes del modelado, ejecutar validación externa independiente, establecer el dominio de aplicabilidad químico y documentar el uso previsto. Para Ames/ICH M7, continuar con [Genotoxicidad COM G11](../aplicaciones/genotoxicidad-com-g11-ich-m7.md).

**Estado:** las firmas se contrastaron con el código fuente; no se ejecutó una prueba integral de las cuatro etapas en este cambio.
