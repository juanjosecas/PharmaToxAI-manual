# Verificar una instalación

La instalación termina cuando el paquete puede importarse **desde el intérprete correcto** y las dependencias necesarias para la tarea funcionan. `pip check` detecta incompatibilidades de metadatos, no errores de ejecución ni compatibilidad de todas las integraciones.

## Comprobación mínima

En el entorno activado:

~~~bash
python --version
python -m pip --version
python -m pip show pharmatoxai
python -m pip check
python - <<'PY'
import sys
import importlib.metadata as md
import numpy
from rdkit import rdBase
import pharmatoxai

print("Python:", sys.version.split()[0])
print("Ejecutable:", sys.executable)
print("PharmaToxAI:", md.version("pharmatoxai"))
print("Origen:", pharmatoxai.__file__)
print("NumPy:", numpy.__version__)
print("RDKit:", rdBase.rdkitVersion)
PY
~~~

Para una instalación editable, `Origen` debería apuntar al directorio `src/pharmatoxai` del clon. Si apunta a otra ruta, se está ejecutando otro intérprete o se importó otra instalación.

## Prueba funcional pequeña

~~~bash
python - <<'PY'
from rdkit import Chem
from rdkit.Chem import Descriptors
from pharmatoxai.data_io import load_data

mol = Chem.MolFromSmiles("CCO")
assert mol is not None
assert Descriptors.MolWt(mol) > 40
assert callable(load_data)
print("OK: RDKit y pharmatoxai.data_io")
PY
~~~

Esta prueba comprueba solamente importación y una operación sencilla de RDKit; no reemplaza la batería de tests.

## Tests del repositorio de código

Desde el directorio que contiene `setup.py`:

~~~bash
python -m pip install -e ".[test]"
python -m pytest -q
~~~

La ejecución completa puede incluir módulos opcionales o servicios externos según los tests presentes en la revisión utilizada. Registrar el commit, la cantidad de tests y los fallos concretos; no afirmar que toda la suite pasa sin haberla ejecutado.

## Integraciones externas

~~~bash
java -version     # PaDEL
xtb --version     # xTB, si se utiliza
nvidia-smi        # NVIDIA, si corresponde
~~~

`padelpy` es el wrapper Python y necesita Java para ejecutar PaDEL. Los flujos cuánticos que invocan el binario `xtb` requieren ese ejecutable accesible por `PATH`; instalar una biblioteca Python no garantiza su presencia. Comprobar cada dependencia externa solamente cuando se vaya a usar ese módulo.

## Registrar la instalación utilizada

~~~bash
python -m pip freeze > requirements-resolved.txt
git rev-parse HEAD > source-commit.txt
python -m pip check > pip-check.txt
~~~

Los comandos Git deben ejecutarse dentro del repositorio **PharmaToxAI**, no dentro del manual. Evitar subir exportaciones que contengan rutas personales, URLs privadas o credenciales.

## Problemas frecuentes

| Error | Causa posible y prueba |
| --- | --- |
| `ModuleNotFoundError` | Comparar `sys.executable` y `python -m pip show pharmatoxai` |
| Un módulo parece desactualizado | Consultar `pharmatoxai.__file__` y reiniciar kernel |
| `pip install -e .` no encuentra paquete | Ejecutar desde la raíz con `setup.py` |
| Conflicto NumPy 1.x/2.x | Revisar `pip check`; PharmaToxAI declara `numpy<2` |
| PaDEL falla | `java -version` y configuración Java |
| xTB falla | `command -v xtb` y `xtb --version` |
| CUDA no está disponible | Consultar [GPU](gpu.md) |

Las instrucciones para crear entornos están en [instalación](instalacion.md).
