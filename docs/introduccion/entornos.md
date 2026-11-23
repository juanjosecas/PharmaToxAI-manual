# Gestión de entornos

PharmaToxAI declara Python >=3.11, restringe NumPy a `>=1.26,<2.0` y distribuye una definición CPU (`environment.yml`) y otra GPU (`environment-gpu.yml`). En ambos casos, crear el entorno no instala el paquete local: después hay que ejecutar `pip install -e .` o, si las dependencias ya están resueltas, `pip install --no-deps -e .`.

## Qué estrategia usar

| Situación | Base | Instalación final |
| --- | --- | --- |
| Ejecutar análisis con pip | `venv` y Python 3.11 | `python -m pip install .` |
| Desarrollar y ejecutar notebooks | `venv` y Python 3.11 | `python -m pip install -e ".[notebooks,test]"` |
| Resolver dependencias científicas con Conda | Miniforge, `environment.yml` | `python -m pip install --no-deps -e .` |
| Probar PyTorch con NVIDIA | Entorno independiente, según plataforma | Ver [GPU](gpu.md) |
| Compilar este manual | Entorno Python 3.12 independiente | `pip install -r requirements-docs.txt` |

No instalar las dependencias del manual en el entorno científico: `requirements-docs.txt` utiliza NumPy 2.x y pandas 3.x, mientras PharmaToxAI fija NumPy <2.

## Instalación editable y modificaciones

Desde la raíz del repositorio del código:

~~~bash
python -m pip install -e .
python -c "import pharmatoxai; print(pharmatoxai.__file__)"
~~~

`-e` registra el paquete apuntando a `src/pharmatoxai`; para cambios en archivos Python normalmente basta con reiniciar el proceso o kernel que importó el módulo. Si cambia `setup.py`, `requirements.txt` o la definición de extras, repetir la instalación. Para experimentar sin modificar código fuente, `pip install .` resulta más apropiado.

Para trabajar con versiones estables del análisis, fijar **commit del repositorio** y **resolución del entorno**, no solo la versión declarada en `setup.py`. La etiqueta `0.4.0a1` es un dato de empaquetado; el código de `main` puede seguir cambiando.

## Crear, actualizar y eliminar

~~~bash
# Crear desde el YAML oficial (en el repositorio del código)
mamba env create -f environment.yml
conda activate pharmatoxai
python -m pip install --no-deps -e .

# Inspeccionar
conda info --envs
python -m pip check
conda list

# Exportar el estado efectivo
conda env export > environment-resolved.yml
conda list --explicit > conda-explicit.txt

# Eliminar, luego de desactivar
conda deactivate
conda env remove -n pharmatoxai
~~~

Las especificaciones de `environment.yml` mezclan canales `conda-forge` y `defaults`; una nueva resolución en otra fecha o sistema puede cambiar versiones. `conda list --explicit` es específico de plataforma. No interpretar una exportación como garantía de portabilidad binaria.

## Trabajar con varios proyectos y kernels

~~~bash
conda activate pharmatoxai
python -m pip install ipykernel
python -m ipykernel install --user --name pharmatoxai --display-name "Python (PharmaToxAI)"
python -m jupyter kernelspec list
~~~

Dentro de Jupyter, comprobar:

~~~python
import sys
import pharmatoxai
print(sys.executable)
print(pharmatoxai.__file__)
~~~

El kernel debe utilizar el intérprete donde se instaló PharmaToxAI. Si aparecen múltiples kernels con el mismo nombre visible, comprobar `sys.executable`, no confiar solamente en el selector de la interfaz.

## Reproducir un entorno

Guardar junto al análisis: commit Git, `python --version`, `pip freeze`, exportación Conda (si aplica), plataforma, versión RDKit, semillas y parámetros del pipeline. Para un entorno nuevo, reconstruir desde las especificaciones originales y verificar antes de ejecutar los notebooks.

Consultar [instalación](instalacion.md) y [verificación](verificacion.md).
