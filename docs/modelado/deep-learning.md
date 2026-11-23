# Deep learning con PyTorch

**Módulos:** `pharmatoxai.nn_models` y `pharmatoxai.nn_training`. La implementación actual proporciona MLP para regresión y clasificación sobre matrices de descriptores/fingerprints; no asumir que estas redes aprovechan directamente la conectividad de grafos moleculares.

## Entrenamiento de una MLP

~~~python
from pharmatoxai.nn_training import train_nn_regression

artifact = train_nn_regression(
    X_train, y_train,
    hidden_layers=(64, 32),
    dropout=0.2,
    use_batch_norm=True,
    epochs=50,
    batch_size=32,
    learning_rate=1e-3,
    weight_decay=1e-4,
    val_split=0.2,
    early_stopping_patience=8,
    device="cpu",
    random_state=42,
    verbose=False,
)
model = artifact["model"]
print(artifact["training"]["n_epochs_run"])
~~~

El entrenador devuelve `model`, historial y metadatos; conserva además un scaler en `artifact["preprocessing"]["scaler_object"]`. Antes de inferir, consultar qué normalización y tratamiento de NaN aprendió el entrenamiento. **No pasar directamente un DataFrame crudo a la red sin aplicar el mismo preprocesamiento.**

## Clasificación

~~~python
from pharmatoxai.nn_training import train_nn_classification

artifact = train_nn_classification(
    X_train, y_train,
    hidden_layers=(64, 32),
    epochs=50,
    batch_size=32,
    val_split=0.2,
    early_stopping_patience=8,
    device="cpu",
    random_state=42,
    verbose=False,
)
~~~

La arquitectura interna devuelve logits; para obtener probabilidades deben utilizarse herramientas compatibles con la tarea y el preprocesamiento guardado. No aplicar directamente `sigmoid` a logits multiclase como si representaran probabilidades independientes.

## Reproducibilidad y comparación

Registrar semilla, librerías, dispositivo, batch size, epochs, criterio de parada, tamaño de cada partición y curva de pérdida. La semilla controla parte del azar, pero la ejecución GPU puede no ser determinista en todas las operaciones.

Las redes neuronales con pocas muestras moleculares pueden presentar alta varianza. Evaluar contra baselines clásicos con el mismo holdout, evitando elegir el modelo por métricas calculadas durante entrenamiento. Para GPU, consultar [instalación GPU](../introduccion/gpu.md).

**Fuentes:** [nn_models.py](https://github.com/juanjosecas/PharmaToxAI/blob/main/src/pharmatoxai/nn_models.py), [nn_training.py](https://github.com/juanjosecas/PharmaToxAI/blob/main/src/pharmatoxai/nn_training.py).
