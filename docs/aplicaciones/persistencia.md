# Persistencia y versionado de modelos

**Módulo:** `pharmatoxai.model_persistence`. `save_model` necesita modelo, `X_train`, `X_test`, `y_train` e `y_test`, y guarda un directorio con modelo serializado, tablas y metadatos.

## Guardar un modelo entrenado

~~~python
from pharmatoxai.model_persistence import save_model, load_model

saved = save_model(
    model,
    X_train=X_train, X_test=X_test,
    y_train=y_train, y_test=y_test,
    output_path="outputs/models",
    model_name="baseline_rf",
    feature_names=X_train.columns.tolist(),
    metadata={"endpoint": "Activity", "split": "random", "seed": 42},
)
print(saved)
restored = load_model(
    "outputs/models/baseline_rf",
    load_data=False, load_metrics=True, verbose=False,
)
restored_model = restored["model"]
~~~

La ruta se interpreta respecto del directorio de ejecución. Guardar el modelo **junto con el pipeline realmente aplicado**: un scaler externo que no se persistió puede invalidar inferencia aunque las columnas coincidan. Registrar clase del endpoint, orden exacto de features, backend, radio/número de bits y política de SMILES.

## Distribución portable

`save_model_package(model_path, package_path=...)` empaqueta objetos Python y archivos del directorio usando pickle. `load_model_package` puede reconstruirlos. **Solo cargar paquetes de origen confiable**: joblib/pickle pueden ejecutar código durante la deserialización. Portabilidad de rutas no implica compatibilidad binaria ni de versiones.

La opción `output_format="onnx"` de `save_model` intenta exportar a ONNX cuando el tipo de estimador y sus conversores lo soportan; no garantiza una exportación funcional de modelos arbitrarios. Comprobar forma de entrada, opset, diferencias numéricas y versiones mediante un test de paridad.

## Comprobación tras guardar

~~~python
import numpy as np

before = model.predict(X_test)
after = restored_model.predict(X_test)
np.testing.assert_allclose(before, after, rtol=1e-8, atol=1e-10)
~~~

Registrar checksum de los artefactos, commit Git, versiones de RDKit/scikit-learn y fecha de entrenamiento. No guardar datasets confidenciales o credenciales dentro de paquetes destinados a distribución. El código actual guarda por defecto matrices `X/y` junto con el modelo.

**Fuente:** [model_persistence.py](https://github.com/juanjosecas/PharmaToxAI/blob/main/src/pharmatoxai/model_persistence.py) y [guía](https://github.com/juanjosecas/PharmaToxAI/blob/main/docs/model_persistence.md).
