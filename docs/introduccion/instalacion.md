# Instalación de PharmaToxAI

Esta guía diferencia la instalación del **paquete PharmaToxAI** y la del **sitio del manual**. Se recomienda aislar ambos entornos: sus restricciones de NumPy no coinciden.

El empaquetado actual de PharmaToxAI declara **Python >=3.11**, versión **0.4.0a1** y **NumPy >=1.26,<2.0**. Para los ejemplos se utiliza Python **3.11**, coherente con el archivo environment.yml del repositorio. La versión declarada en setup.py no demuestra que exista una distribución en PyPI.

!!! note "Repositorio privado"
    PharmaToxAI es actualmente privado. Para clonarlo o instalarlo por URL Git se necesita acceso autorizado y autenticación GitHub configurada. El manual es público; no se deben incluir tokens o credenciales en sus archivos.

## Elegir cómo instalar

| Caso | Orden desde la raíz del repositorio PharmaToxAI |
| --- | --- |
| Instalación local estándar | python -m pip install . |
| Desarrollo local (recomendado) | python -m pip install -e . |
| Desarrollar con extras de notebooks y tests | python -m pip install -e ".[notebooks,test]" |
| Dependencias instaladas previamente con Conda | python -m pip install --no-deps -e . |

El indicador **-e** significa **--editable**. El punto **.** representa el directorio actual, que debe contener setup.py. Una instalación editable importa los módulos desde el árbol fuente. Los cambios en archivos Python se reflejan sin reinstalar; los cambios de metadatos, dependencias y ciertos puntos de entrada pueden requerir una nueva instalación. Una instalación estándar copia una distribución del proyecto al entorno. Ambas opciones instalan dependencias declaradas, salvo cuando se utiliza --no-deps.

## Opción A: pip y venv

### Linux (Fedora, Ubuntu, Debian) y macOS

Se requiere Git, Python 3.11 y el soporte venv del intérprete:

~~~bash
git clone https://github.com/juanjosecas/PharmaToxAI.git
cd PharmaToxAI

python3.11 -m venv .venv
source .venv/bin/activate

python -m pip install --upgrade pip setuptools wheel
python -m pip install -e .
~~~

Para instalar sin edición del código fuente, sustituir la última orden por:

~~~bash
python -m pip install .
~~~

!!! tip "Python del sistema"
    En Fedora o Ubuntu es preferible utilizar un entorno aislado, sin sudo pip. Si no existe python3.11, instalar ese intérprete o utilizar Miniforge. Algunas distribuciones requieren instalar el paquete python3.11-venv por separado.

### Windows PowerShell

~~~powershell
git clone https://github.com/juanjosecas/PharmaToxAI.git
cd PharmaToxAI

py -3.11 -m venv .venv
.\.venv\Scripts\Activate.ps1

python -m pip install --upgrade pip setuptools wheel
python -m pip install -e .
~~~

Si PowerShell bloquea Activate.ps1, utilizar el ejecutable del entorno directamente, sin cambiar permanentemente la política de ejecución:

~~~powershell
.\.venv\Scripts\python.exe -m pip install -e .
~~~

## Opción B: Miniforge, Conda o Mamba

[Miniforge](https://conda-forge.org/download/) incluye conda y mamba y usa el ecosistema conda-forge. Es útil para instalar dependencias científicas compiladas como RDKit y NumPy. Hay dos estrategias distintas.

### B1. Crear un entorno mínimo y dejar que pip resuelva dependencias

~~~bash
mamba create -n pharmatoxai python=3.11 pip -c conda-forge
conda activate pharmatoxai

git clone https://github.com/juanjosecas/PharmaToxAI.git
cd PharmaToxAI

python -m pip install -e .
~~~

En este caso pip resuelve las dependencias de setup.py y requirements.txt. Se puede utilizar conda en lugar de mamba.

### B2. Utilizar el environment.yml mantenido por el proyecto

Desde la raíz del repositorio:

~~~bash
mamba env create -f environment.yml
conda activate pharmatoxai

python -m pip install --no-deps -e .
~~~

Equivalente con Conda:

~~~bash
conda env create -f environment.yml
conda activate pharmatoxai
python -m pip install --no-deps -e .
~~~

**Por qué usar --no-deps:** environment.yml instala primero bibliotecas con Conda y algunas dependencias mediante pip. El archivo **no instala PharmaToxAI como paquete local**. La orden editable lo registra sin permitir que pip reemplace las dependencias previamente seleccionadas por Conda.

Este método presupone que el YAML contiene las dependencias necesarias. Si hay módulos ausentes, hay que resolverlos explícitamente; --no-deps no los instalará. El YAML actual mezcla canales conda-forge y defaults, por lo que es conveniente registrar los canales y las versiones resueltas.

!!! warning "Conda y pip en el mismo entorno"
    Instalar primero las bibliotecas compiladas mediante Conda/Mamba, y usar pip después para paquetes que falten o para instalar PharmaToxAI. Conda puede perder visibilidad de modificaciones posteriores realizadas por pip. Si una actualización rompe dependencias, es más reproducible reconstruir el entorno que seguir mezclando instaladores.

### Entorno GPU

El repositorio incluye environment-gpu.yml con dependencias de PyTorch/CUDA. No equivale a compatibilidad universal con GPU. Se debe comprobar controlador NVIDIA, versión efectiva de PyTorch, compatibilidad CUDA y plataforma. No combinar directamente las versiones CPU y GPU sin verificar el resultado del resolvedor.

## Opción C: instalar desde Git

Pip admite instalar una revisión concreta directamente desde una URL de control de versiones:

~~~bash
python -m pip install "git+https://github.com/juanjosecas/PharmaToxAI.git@COMMIT_SHA"
~~~

Reemplazar COMMIT_SHA por un hash real. Es una instalación no editable. Para editar código o depurar cambios, clonar el repositorio y usar -e:

~~~bash
git clone https://github.com/juanjosecas/PharmaToxAI.git
cd PharmaToxAI
git checkout COMMIT_SHA
python -m pip install -e .
~~~

Se requiere autenticación para acceder al repositorio privado. No incrustar un token en la URL. **No suponer** que se puede ejecutar pip install pharmatoxai desde PyPI hasta comprobar que la versión está efectivamente publicada.

## Dependencias opcionales

setup.py declara los extras notebooks, test, molfeat, sql, pint, reports, onnx y deepchem. También declara all, que incluye integraciones opcionales que pueden generar conflictos.

~~~bash
# En la raíz de PharmaToxAI, con el entorno activado
python -m pip install -e ".[notebooks,test]"
# Ejemplos alternativos, solo si se necesitan:
python -m pip install -e ".[molfeat]"
python -m pip install -e ".[reports]"
~~~

**PaDEL requiere Java.** Cuando se utilice padelpy, comprobar java -version; el YAML de Conda incluye openjdk>=11. La instalación del extra all no garantiza disponibilidad ni compatibilidad de CUDA.

## Comprobar la instalación

Ejecutar en el entorno seleccionado:

~~~bash
python --version
python -m pip --version
python -m pip show pharmatoxai
python -m pip check
python -c "import pharmatoxai; print(pharmatoxai.__file__)"
~~~

Para la instalación editable, la ruta importada normalmente termina en **PharmaToxAI/src/pharmatoxai/**.

~~~bash
python - <<'PY'
import sys
import numpy
import sklearn
from rdkit import rdBase
import pharmatoxai

print("Python:", sys.version.split()[0])
print("NumPy:", numpy.__version__)
print("scikit-learn:", sklearn.__version__)
print("RDKit:", rdBase.rdkitVersion)
print("Paquete:", pharmatoxai.__file__)
PY
~~~

Un pip check correcto confirma coherencia de dependencias declaradas, pero no certifica el funcionamiento de todas las integraciones.

## JupyterLab, VS Code y kernels

Instalar JupyterLab y el kernel en el mismo entorno que contiene PharmaToxAI:

~~~bash
python -m pip install jupyterlab ipykernel
python -m ipykernel install --user --name pharmatoxai --display-name "Python (PharmaToxAI)"
jupyter lab
~~~

En VS Code o Jupyter seleccionar el intérprete/kernel **Python (PharmaToxAI)**. Si se edita un módulo ya importado, reiniciar el kernel para cargar la versión nueva. Instalar -e . en una terminal distinta no cambia automáticamente el intérprete utilizado por un notebook abierto.

## Instalar el manual por separado

El sitio de documentación utiliza requirements-docs.txt, que actualmente fija versiones de NumPy 2.x y pandas 3.x. **No reutilizar el entorno científico** de PharmaToxAI, que requiere NumPy <2.

### Con venv

~~~bash
git clone https://github.com/juanjosecas/PharmaToxAI-manual.git
cd PharmaToxAI-manual

python3.12 -m venv .venv
source .venv/bin/activate
python -m pip install -r requirements-docs.txt
mkdocs serve
~~~

### Con Miniforge

~~~bash
mamba create -n pharmatoxai-docs python=3.12 pip -c conda-forge
conda activate pharmatoxai-docs

git clone https://github.com/juanjosecas/PharmaToxAI-manual.git
cd PharmaToxAI-manual
python -m pip install -r requirements-docs.txt
mkdocs serve
~~~

Comprobar el sitio con:

~~~bash
mkdocs build --strict
~~~

El notebook introductorio del manual usa datos sintéticos y scikit-learn; por ahora **no requiere importar PharmaToxAI**. Los tutoriales posteriores deberán fijar versiones o revisiones concretas del paquete y verificar su ejecución.

## Solución de problemas

| Síntoma | Comprobación |
| --- | --- |
| ModuleNotFoundError: pharmatoxai | python -m pip show pharmatoxai; verificar que el intérprete y kernel pertenecen al mismo entorno |
| pip install -e . no encuentra el proyecto | Entrar en el directorio que contiene setup.py |
| GitHub rechaza el clon | Confirmar permisos y autenticación de Git para el repositorio privado |
| Conflictos de NumPy | PharmaToxAI exige NumPy <2; no mezclar con requirements-docs.txt |
| Se modificó un módulo pero no cambia la ejecución | Comprobar pharmatoxai.__file__ y reiniciar el kernel |
| Se modificó setup.py o requirements.txt | Repetir pip install -e . y revisar la resolución |
| conda activate no funciona | Inicializar la shell con conda init y abrir una nueva terminal |
| Fallo al ejecutar PaDEL | Comprobar java -version |
| Fallo en PyTorch/CUDA | Comprobar controlador, nvidia-smi y versiones instaladas |

## Registrar el entorno para reproducibilidad

Desde el entorno y el repositorio fuente utilizados en el experimento:

~~~bash
python -m pip freeze > requirements-resolved.txt
git rev-parse HEAD > source-commit.txt
~~~

Si se usó Conda:

~~~bash
conda env export > environment-resolved.yml
conda list --explicit > conda-explicit.txt
~~~

Registrar también dataset, identificadores, particiones train/validation/test, semillas, representaciones moleculares y parámetros. El historial de dependencias por sí solo no permite reproducir un resultado QSAR.

## Referencias técnicas

- [pip: proyectos locales e instalación editable](https://pip.pypa.io/en/stable/topics/local-project-installs/).
- [pip: instalación desde Git](https://pip.pypa.io/en/stable/topics/vcs-support/).
- [Miniforge](https://conda-forge.org/download/).
- [Conda: entornos](https://docs.conda.io/projects/conda/en/stable/user-guide/tasks/manage-environments.html).
- [Conda: gestión de paquetes](https://docs.conda.io/projects/conda/en/stable/user-guide/tasks/manage-pkgs.html).
