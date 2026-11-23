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


## Fundamentos de arquitectura y optimización

Una red multicapa aplica transformaciones afines y activaciones sucesivas:

$$
h^{(\ell+1)}=\phi(W^{(\ell)}h^{(\ell)}+b^{(\ell)}),\qquad
\hat y=W^{(L)}h^{(L)}+b^{(L)}
$$

Los parámetros son pesos \(W\) y sesgos \(b\). hidden_layers=(64, 32) configura dos capas ocultas de esas dimensiones; no define directamente el número de moléculas ni de características de entrada.

### Función de pérdida y clasificación

En regresión con error cuadrático:

$$
MSE=\frac{1}{n}\sum_i(y_i-\hat y_i)^2
$$

Para una clasificación con \(C\) clases mutuamente excluyentes, los logits \(z_c\) se convierten en probabilidades mediante softmax:

$$
p_c=\frac{\exp(z_c)}{\sum_{j=1}^{C}\exp(z_j)},\qquad
\mathcal L_{\mathrm{CE}}=-\sum_i\log p_{i,y_i}
$$

La implementación de MLP clasificadora devuelve logits: no aplicar softmax antes de una pérdida como CrossEntropyLoss que ya lo incorpora internamente.

### Regularización y optimizador

Con dropout de tasa \(p\), cada activación se anula estocásticamente con probabilidad \(p\) **durante entrenamiento**; en evaluación la máscara se desactiva. dropout=0.2 no significa eliminar un 20 % fijo de neuronas del modelo final.

El weight decay de AdamW añade contracción desacoplada de parámetros, esquemáticamente:

$$
\theta_{t+1}=(1-\eta\lambda)\theta_t-\eta\,\widehat g_t
$$

donde \(\eta\) es learning_rate y \(\widehat g_t\) resume el paso adaptativo. batch_size determina cuántas observaciones contribuyen a cada gradiente; epochs representa el máximo de pasadas completas por train, no el número garantizado si actúa early stopping.

| Parámetro | Efecto típico | Riesgo |
| --- | --- | --- |
| hidden_layers | Capacidad de representación | Sobreajuste con pocos compuestos |
| dropout | Regularización aleatoria | Subajuste si es excesivo |
| use_batch_norm | Estabilización de activaciones por lote | Comportamiento sensible a batches pequeños |
| learning_rate | Magnitud de actualización | Inestabilidad o convergencia lenta |
| weight_decay | Penalización/contracción | Subajuste si es alto |
| batch_size | Ruido del gradiente y memoria | Lotes pequeños pueden afectar BatchNorm |
| val_split | Porción interna destinada a validación | Debe quedar aislada del ajuste de preprocesamiento |
| early_stopping_patience | Pasadas sin mejora toleradas | Parada prematura o sobreajuste |
| clip_grad_norm | Cota de norma de gradiente | Cambia dinámica de optimización |
| device, use_amp | Hardware y precisión mixta | Diferencias numéricas y reproducibilidad |

### Limitación específica del entrenador actual

**Verificado en nn_training.py:** train_nn_regression y train_nn_classification calculan imputación y StandardScaler **antes** de separar train y validation internos mediante prepare_data_loaders. Esto introduce información de las características de validación en el preprocesamiento (leakage no supervisado). Por lo tanto, la pérdida interna de validación no debe tratarse como una estimación completamente aislada. Para evaluaciones rigurosas, preparar los splits externos antes de entrenar, ajustar transformaciones únicamente sobre cada train y no seleccionar modelos por el test; si se requiere que el early stopping sea totalmente independiente, hay que corregir el orden dentro del entrenador.

También las métricas de performance del artefacto se calculan sobre el conjunto proporcionado al entrenador, no sobre un test independiente.

### Modelos pequeños y coste

En datasets moleculares modestos suele haber pocos datos frente al número de parámetros. La comparación con Ridge, RandomForest y boosting debe realizarse bajo el mismo split y con incertidumbre del desempeño, en lugar de atribuir ventaja a la complejidad de la arquitectura.
