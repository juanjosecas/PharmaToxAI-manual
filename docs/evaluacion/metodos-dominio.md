# Métodos de dominio de aplicabilidad: comparación crítica

Este documento complementa el [dominio de aplicabilidad de PharmaToxAI](dominio-aplicabilidad.md). Distingue lo que implementa el paquete de otros enfoques reportados en la bibliografía.

## Qué intenta delimitar el AD

El dominio de aplicabilidad (AD) puede considerarse en diferentes espacios: descriptores numéricos, estructuras y fingerprints, condiciones del endpoint y **fiabilidad predictiva observada**. Estar dentro de un espacio de descriptores no acredita automáticamente un error pequeño. Los dominios definidos por distintos métodos pueden discrepar para la misma sustancia.

[Sahigara y cols., *Molecules* (2012)](https://pmc.ncbi.nlm.nih.gov/articles/PMC6268288/) comparan métodos de AD en modelos de bioconcentración e identifican diferencias entre estrategias de umbral y conjuntos externos.

## Familias de métodos

| Método | Criterio | Ventaja | Límite |
| --- | --- | --- | --- |
| Rangos / bounding box | Todos los descriptores dentro de mínimos y máximos de train | Transparente | Admite regiones multidimensionales sin muestras |
| Bounding box en PCA | Rangos de scores en componentes principales | Atenúa colinealidad | Pierde información de componentes descartados |
| Convex hull | Consulta dentro de la envolvente convexa de train | Interpolación geométrica | Coste alto en dimensión elevada; incluye huecos |
| Euclidiana o Mahalanobis | Distancia a referencia, con umbral | Diagnóstico continuo | Depende de escala, covarianza y criterio de corte |
| kNN / Tanimoto | Proximidad a moléculas análogas | Evidencia estructural interpretable | Activity cliffs; dependencia de fingerprint |
| Densidad | Región de probabilidad estimada elevada | Detecta parte de los vacíos internos | Sensible a ancho de banda y dimensionalidad |
| Basado en error | Error condicionado a distancia o atipicidad | Directamente ligado a fiabilidad | Requiere predicciones independientes para calibrar |

**Implementado actualmente en PharmaToxAI:** Hotelling T², Mahalanobis robusto y distancia media kNN después de PCA. El dominio por rangos, convex hull, densidad y leverage clásico no forman parte de esta función. Esta distinción es esencial cuando se leen comparativas externas.

### Dominios rectangulares y regiones vacías

Para un vector de descriptores \(x\), un criterio por rango marginal exige:

$$
\min_i x_{ij}\le x_j\le\max_i x_{ij},\qquad \forall j.
$$

Puede aceptar una combinación nunca observada aunque cada descriptor por separado esté dentro de rango. Una envolvente convexa también puede abarcar regiones de baja densidad.

### Leverage y Williams plot

Para regresión lineal, con matriz de diseño \(X\) que incluya intercepto cuando corresponda:

$$
H=X(X^\top X)^{-1}X^\top,\qquad h_i=H_{ii}.
$$

El warning leverage heurístico se expresa como:

$$
h^*=\frac{3p}{n},
$$

donde \(p\) es el número de parámetros de la matriz de diseño y \(n\) el número de observaciones de entrenamiento. La expresión presupone un diseño de rango completo; si existe colinealidad perfecta, debe reconsiderarse el cálculo. No es un umbral universal ni equivale al T² que utiliza el paquete. Un Williams plot representa residuos estandarizados frente a leverage. [Comparación metodológica de Sahigara y cols.](https://pmc.ncbi.nlm.nih.gov/articles/PMC6268288/).

### Cómo elegir un umbral

Sea \(d_i\) una distancia de AD y \(\tau\) un límite fijado en desarrollo:

$$
\operatorname{Coverage}(\tau)=
\frac{\sum_i\mathbf 1(d_i\le\tau)}{n},
$$

$$
\operatorname{RMSE}_{\rm in}(\tau)=
\sqrt{\frac{\sum_{i:d_i\le\tau}(y_i-\hat y_i)^2}
{\sum_i\mathbf 1(d_i\le\tau)}}.
$$

Informar también el error fuera del dominio, tamaños de ambos subconjuntos y el número de abstenciones. Una disminución del error dentro del AD puede ser consecuencia de excluir casos difíciles o muy pocos registros. **El umbral debe elegirse sin utilizar el test externo final**. Revisar desempeño condicionado a distancia, scaffold y actividad, preferentemente con resultados fuera de fold o un conjunto de desarrollo independiente.

## Extrapolación: un debate que requiere datos

[Variational AI](https://variational.ai/news/blog/applicability-domains-are-common-in-qsar-but-irrelevant-for-conventional-ml-tasks/) argumenta que los modelos modernos podrían generalizar más allá de AD conservadores. El artículo examina predicciones de inhibidores de quinasas y muestra también que el error tiende a crecer con distancia al entrenamiento. Es una postura publicada en un blog empresarial con ejemplos específicos: **no demuestra que el AD sea irrelevante en toxicología ni elimina la necesidad de declarar límites del modelo**.

Conviene distinguir tres objetos: medida continua de atipicidad, política de abstención y error observado. Una predicción fuera del AD puede ser útil cuando se ha validado extrapolación; una predicción dentro del AD puede fallar por diferencias mecanísticas, endpoints heterogéneos o activity cliffs.

## Alcance mecanístico y OECD QSAR Toolbox

El [Tutorial 6 de OECD QSAR Toolbox v4.4.1 (2020)](https://qsartoolbox.org/file/2020/04/Tutorial_6_Building-a-QSAR-model.pdf) presenta un ejemplo de toxicidad de aldehídos en *Tetrahymena pyriformis*: definición del endpoint, selección de análogos, perfiles de unión a proteínas, exclusión o revisión de outliers, evaluación y visualización del AD. Es un caso operativo de **ese software**, no una descripción de la API de PharmaToxAI ni una regla de umbral universal.

El estudio de [Viganò, Ballabio y Roncaglioni (2024)](https://pmc.ncbi.nlm.nih.gov/articles/PMC10820364/) compara representaciones moleculares en predicción de eventos asociados a cardiotoxicidad dentro de un marco AOP. Sirve para contextualizar **endpoint y representación**; no es una revisión comparativa de métodos de AD.

## Procedimiento para reportar y evaluar un AD

1. Definir endpoint, sistema experimental y población química.
2. Fijar representación, política de curación y distancia antes de ver el test.
3. Calibrar umbrales con entrenamiento o desarrollo; distinguir decisión de AD y calibración del error.
4. Evaluar cobertura, errores dentro/fuera y abstenciones sobre predicciones independientes.
5. Comunicar sensibilidad a umbral, grupos químicos, actividad y ejemplos de error extremo.
6. Congelar transformadores, columnas y umbrales para inferencia sobre nuevas moléculas.

## Lecturas y calidad de evidencia

- **Comparación académica principal:** [Sahigara y cols., *Comparison of Different Approaches to Define the Applicability Domain of QSAR Models* (2012)](https://pmc.ncbi.nlm.nih.gov/articles/PMC6268288/).
- **Tutorial institucional:** [OECD QSAR Toolbox, Tutorial 6 (2020)](https://qsartoolbox.org/file/2020/04/Tutorial_6_Building-a-QSAR-model.pdf).
- **Perspectiva crítica:** [Variational AI, *Applicability domains are common in QSAR but irrelevant for conventional ML tasks*](https://variational.ai/news/blog/applicability-domains-are-common-in-qsar-but-irrelevant-for-conventional-ml-tasks/).
- **Aplicación de ML a endpoints mecanísticos:** [Viganò y cols. (2024), cardiotoxicidad y AOP](https://pmc.ncbi.nlm.nih.gov/articles/PMC10820364/).
- **Introducción divulgativa general:** [Neovarsity, *A Beginner's Guide to QSAR Modeling*](https://neovarsity.org/blogs/cheminformatics-qsar-beginner-guides). No es referencia primaria de umbrales AD.
- **Fuente proporcionada, sin contenido verificable en esta revisión:** [Covasyn, *Applicability Domain QSAR Explained*](https://covasyn.com/en/blog/applicability-domain-qsar-explained). No se le atribuyen resultados específicos.
