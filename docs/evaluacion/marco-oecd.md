# Dominio de aplicabilidad y evaluación regulatoria OECD

Este capítulo relaciona la [implementación del dominio de aplicabilidad en PharmaToxAI](dominio-aplicabilidad.md) con los criterios de documentación y evaluación de modelos y predicciones (Q)SAR. Para métodos alternativos y discusión crítica, consultar [comparación de métodos AD](metodos-dominio.md).

## Tres documentos OECD y sus funciones

| Documento | Función | Implicación para el AD |
| --- | --- | --- |
| [Principios OECD de validación (2004)](https://www.oecd.org/content/dam/oecd/en/topics/policy-sub-issues/assessment-of-chemicals/oecd-principles-for-the-validation-for-regulatory-purposes-of-quantitative-structure-activity-relationship-models.pdf) | Cinco principios para considerar un modelo QSAR con fines regulatorios | El tercer principio exige un AD definido |
| [Guidance Document on the Validation of (Q)SAR Models (2007; publicación OECD)](https://www.oecd.org/en/publications/guidance-document-on-the-validation-of-quantitative-structure-activity-relationship-q-sar-models_9789264085442-en.html) | Orientación explicativa sobre la validación | Documentar criterios, procedimientos y evidencia de validación |
| [(Q)SAR Assessment Framework (QAF)](https://www.oecd.org/en/publications/q-sar-assessment-framework-guidance-for-the-regulatory-assessment-of-quantitative-structure-activity-relationship-models-and-predictions_d96118f6-en.html) | Marco para evaluar modelos y predicciones en contextos regulatorios | Considerar por separado la calidad del modelo y la adecuación de una predicción concreta |

El uso de un marco OECD **no certifica automáticamente** un modelo ni convierte sus umbrales numéricos en reglas regulatorias universales. La decisión de aceptación depende del uso y del contexto regulatorio.

## Los cinco principios OECD

1. **Endpoint definido:** especificar efecto, sistema experimental, condiciones, unidades y tratamiento de censura.
2. **Algoritmo inequívoco:** documentar featurización, preprocesamiento, estimador, hiperparámetros, versión y regla de predicción.
3. **Dominio de aplicabilidad definido:** explicar estructuras, propiedades y, cuando corresponda, mecanismos cubiertos; justificar la regla para decidir si una consulta está dentro o fuera.
4. **Medidas adecuadas de ajuste, robustez y predictividad:** separar ajuste interno de evaluación externa y estudiar el desempeño dentro/fuera del AD.
5. **Interpretación mecanística, si es posible:** discutir relación entre representación, endpoint y mecanismo sin exigir una explicación mecanística inexistente.

El tercer principio no se satisface únicamente imprimiendo T2_flag, MD2_flag y kNN_flag. Se necesita evidencia de que esos indicadores son pertinentes para el problema y la región química de interés.

## Del espacio químico al desempeño predictivo

Sea \(d(x)\) una medida de atipicidad y \(\tau\) el umbral calibrado en desarrollo:

$$
A_{\tau}(x)=\mathbf 1[d(x)\leq\tau].
$$

La cobertura empírica del dominio sobre un conjunto independiente es

$$
C(\tau)=\frac{1}{n}\sum_{i=1}^{n}A_{\tau}(x_i).
$$

En regresión, para \(n_{\mathrm{in}}=\sum_i A_{\tau}(x_i)>0\):

$$
RMSE_{\mathrm{in}}(\tau)=
\sqrt{\frac{\sum_i A_{\tau}(x_i)(y_i-\hat y_i)^2}{n_{\mathrm{in}}}}.
$$

Reportar conjuntamente cobertura, RMSE dentro y fuera, número de casos y errores extremos. Un AD más restrictivo puede reducir RMSE simplemente por **abstenerse en casos difíciles**; la comparación debe considerar el coste de la abstención. Si \(n_{\mathrm{in}}=0\), la métrica condicional no está definida.

Para clasificación, sustituir por sensibilidad, especificidad, precisión, MCC, calibración y tasas de abstención según el riesgo de falsos negativos o positivos. Un score de distancia no es una probabilidad de error.

## Qué documentar en PharmaToxAI

| Elemento | Evidencia que conviene conservar |
| --- | --- |
| Estructura de consulta | SMILES original y curado, identidad, estereoisomería, sales |
| Endpoint | Protocolo, unidades, transformación y alcance biológico |
| Representación | Lista y orden de features, fingerprint y parámetros, versiones |
| Referencia | Identificadores del conjunto de entrenamiento y política de duplicados |
| AD numérico | T², MD², kNN, flags, alpha, k, componentes PCA y umbrales |
| AD estructural | Análogos cercanos, similitud Tanimoto, scaffold y fragmentos |
| Evidencia predictiva | Error y cobertura por condición AD sobre datos independientes |
| Excepciones | Activity cliffs, outliers, incertidumbre y decisión de abstención |
| Trazabilidad | Versión de modelo, transformadores, fecha, dataset y código |

La API applicability_domain() implementa T², Mahalanobis robusto y kNN **en espacio PCA**; no implementa por sí sola una validación mecanística ni un análisis regulatorio completo. Los métodos de similitud y leverage se describen en [métodos comparados](metodos-dominio.md), pero no deben atribuirse a esa función.

### Limitaciones que afectan la evaluación

- El umbral kNN actual se calibra usando distancias de train contra train, que incluyen el propio registro como vecino; puede diferir de la distribución de distancias de moléculas verdaderamente nuevas.
- El cuantil de kNN está fijado en 0.95 y **no** cambia con alpha.
- La proyección PCA puede ocultar fragmentos estructurales raros o mecanismos no representados por los descriptores.
- La distancia química y el error del endpoint son magnitudes diferentes; evaluar su asociación empíricamente.
- La validación externa no debe reutilizarse para escoger umbrales ni reglas de abstención.

## Modelo frente a predicción individual

**Evaluación del modelo:** endpoint, dataset, algoritmo, robustez, predictividad, población química y definición reproducible del AD.

**Evaluación de una predicción:** estructura concreta, correspondencia con el endpoint, pertenencia al AD, análogos relevantes, incertidumbre, posibles alertas mecanísticas y justificación de su uso previsto.

Una molécula dentro del AD puede ser un activity cliff; una molécula fuera del AD puede tener una predicción acertada. El estado AD es evidencia contextual, no un veredicto de exactitud.

## Referencias y lectura complementaria

- [OECD: principios de validación (2004)](https://www.oecd.org/content/dam/oecd/en/topics/policy-sub-issues/assessment-of-chemicals/oecd-principles-for-the-validation-for-regulatory-purposes-of-quantitative-structure-activity-relationship-models.pdf).
- [OECD: Guidance Document on the Validation of (Q)SAR Models](https://www.oecd.org/en/publications/guidance-document-on-the-validation-of-quantitative-structure-activity-relationship-q-sar-models_9789264085442-en.html).
- [OECD: (Q)SAR Assessment Framework (QAF)](https://www.oecd.org/en/publications/q-sar-assessment-framework-guidance-for-the-regulatory-assessment-of-quantitative-structure-activity-relationship-models-and-predictions_d96118f6-en.html).
- [OECD QSAR Toolbox: Tutorial 6](https://qsartoolbox.org/file/2020/04/Tutorial_6_Building-a-QSAR-model.pdf): ejemplo práctico de un flujo de construcción de modelos.
- [Sahigara y cols.: comparación de métodos de AD](https://pmc.ncbi.nlm.nih.gov/articles/PMC6268288/): discusión de definiciones y estrategias.
- [Variational AI: postura crítica sobre AD](https://variational.ai/news/blog/applicability-domains-are-common-in-qsar-but-irrelevant-for-conventional-ml-tasks/): perspectiva industrial, no criterio regulatorio.
- [Viganò y cols.: cardiotoxicidad y AOP](https://pmc.ncbi.nlm.nih.gov/articles/PMC10820364/): contexto de representación y endpoint.
- [Covasyn: introducción al AD](https://covasyn.com/en/blog/applicability-domain-qsar-explained) y [Neovarsity: introducción a QSAR](https://neovarsity.org/blogs/cheminformatics-qsar-beginner-guides): material divulgativo, no fuentes normativas.
