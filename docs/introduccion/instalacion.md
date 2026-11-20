# Instalación y reproducibilidad

## Entorno del manual

El sitio y el tutorial inicial requieren Python 3.10 o posterior. Desde la raíz
del repositorio, crea un entorno aislado e instala las dependencias fijadas:

```bash
python -m venv .venv
source .venv/bin/activate
python -m pip install -r requirements-docs.txt
```

En Windows PowerShell, activa el entorno con
`.venv\Scripts\Activate.ps1`. Para previsualizar el sitio:

```bash
mkdocs serve
```

## Entorno de PharmaToxAI

Los tutoriales que utilicen el paquete deben especificar la versión exacta
distribuida públicamente y sus dependencias. Cuando haya una versión compatible
disponible, instálala con el mecanismo publicado por el proyecto; por ejemplo:

```bash
python -m pip install "pharmatoxai==X.Y.Z"
```

Sustituye `X.Y.Z` por una versión real documentada. No uses este ejemplo como
una orden literal ni asumas que una versión privada o no publicada puede
instalarse desde PyPI. El manual público no debe incluir módulos internos,
credenciales ni datos con restricciones de redistribución.

## Registrar el entorno

Guarda la versión de Python, la versión exacta de PharmaToxAI y las versiones
de bibliotecas de modelado junto al resultado. Fijar versiones hace más fácil
reproducir un análisis, pero no garantiza por sí solo que los datos, el código
y las conclusiones sean válidos.
