# Configuración de GPU

La instalación base de PharmaToxAI incluye PyTorch entre sus dependencias, pero **tener instalado torch no implica tener CUDA disponible**. El soporte GPU depende de la plataforma, los controladores y el backend de cada algoritmo.

## Entorno GPU proporcionado por el proyecto

El archivo [`environment-gpu.yml`](https://github.com/juanjosecas/PharmaToxAI/blob/main/environment-gpu.yml) declara Python 3.11, `pytorch>=2.2.0` y `pytorch-cuda=12.1`, con canales `pytorch`, `nvidia`, `conda-forge` y `defaults`.

En un sistema NVIDIA compatible, desde la raíz de PharmaToxAI:

~~~bash
nvidia-smi
mamba env create -f environment-gpu.yml
conda activate pharmatoxai-gpu
python -m pip install --no-deps -e .
~~~

**Estado de compatibilidad:** ese YAML es una configuración incluida en el repositorio, no una matriz validada para todas las combinaciones de SO, driver y arquitectura. Su resolución puede fallar con versiones actuales o en determinadas plataformas. No convertir `pytorch-cuda=12.1` en una supuesta exigencia universal de PharmaToxAI.

## Verificar CUDA y el dispositivo

~~~bash
python - <<'PY'
import torch
print("PyTorch:", torch.__version__)
print("CUDA de PyTorch:", torch.version.cuda)
print("CUDA disponible:", torch.cuda.is_available())
if torch.cuda.is_available():
    print("GPU:", torch.cuda.get_device_name(0))
    x = torch.rand(1024, 1024, device="cuda")
    print("Operación:", float((x @ x).mean().cpu()))
PY
~~~

Un resultado `False` en `torch.cuda.is_available()` significa que **esa instalación de PyTorch** no puede utilizar CUDA en el entorno evaluado. `nvidia-smi` detecta la GPU/controlador NVIDIA, pero no confirma que el ejecutable Python esté cargando un build CUDA.

## GPU no significa aceleración universal

- Los módulos basados en RDKit, descriptores y curación molecular no se aceleran automáticamente mediante CUDA.
- Los modelos PyTorch pueden utilizar GPU si el código y los tensores se mueven al dispositivo adecuado.
- XGBoost, LightGBM y CatBoost requieren parámetros y builds compatibles propios; no basta con que PyTorch detecte CUDA.
- CUDA de NVIDIA no equivale a soporte de GPU AMD o Apple.

Para comprobar qué backend utiliza una operación, registrar configuración, dispositivo seleccionado y tiempos bajo condiciones comparables. No atribuir una mejora a GPU por la sola presencia de hardware.

## Diagnóstico

| Síntoma | Comprobación |
| --- | --- |
| `nvidia-smi` falla | Controlador NVIDIA, GPU visible y contenedor/sistema anfitrión |
| PyTorch importa pero no detecta GPU | `torch.__version__`, `torch.version.cuda`, `torch.cuda.is_available()` |
| El YAML no resuelve | Revisar canales, restricciones, SO y disponibilidad actual de los paquetes; no mezclar indiscriminadamente entornos CPU/GPU |
| Error de memoria CUDA | Reducir batch size o tamaño del modelo y registrar consumo |
| Entrenamiento sigue en CPU | Verificar opciones de backend y dispositivo del algoritmo concreto |

Ver también [entornos](entornos.md) y [verificación](verificacion.md).
