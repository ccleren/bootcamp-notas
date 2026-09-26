# Módulo 44 — Amazon SageMaker

## Resumen

Amazon SageMaker es el servicio de AWS de ML "de extremo a extremo": cubre todo el ciclo de vida de un modelo (preparar datos, entrenar, ajustar, desplegar, monitorizar) desde una plataforma integrada, con soporte para frameworks como TensorFlow, PyTorch o MXNet, además de contenedores Docker propios. A diferencia del módulo 38 (servicios de IA ya entrenados) o el 41 (Bedrock, modelos fundacionales de terceros vía API), aquí el foco es **construir y gestionar tus propios modelos de ML**.

### Flujo de trabajo típico

Obtención de datos → limpieza y preparación → elección del modelo → entrenamiento → evaluación → despliegue → monitoreo, colección de datos y reevaluación (y vuelta a empezar si hace falta reentrenar). SageMaker da herramientas dedicadas para cada etapa, en vez de tener que montarlas por separado.

### Catálogo de componentes de SageMaker

| Componente | Etapa | Qué hace |
|---|---|---|
| **SageMaker Studio** | Interfaz | IDE unificado para desarrollar, entrenar, desplegar y monitorizar modelos, con ejecución segura desde cualquier lugar |
| **Data Wrangler** | Preparación de datos | Interfaz visual/lenguaje natural para acceder a datos (S3, Athena, Redshift, +50 fuentes), generar informes de calidad, visualizar y aplicar +300 transformaciones predefinidas sin escribir PySpark |
| **Feature Store** | Preparación de datos | Almacén centralizado de *features* reutilizables entre modelos, con almacenamiento offline (entrenamiento) y online (inferencia) sincronizados, seguimiento de linaje y recuperación de datos de un momento pasado ("viaje en el tiempo") |
| **Algoritmos incorporados** | Entrenamiento | Algoritmos ya optimizados por AWS para tareas comunes: aprendizaje supervisado, no supervisado, análisis textual, procesamiento de imágenes |
| **Automatic Model Tuning (AMT)** | Entrenamiento | Ajuste automático de hiperparámetros ejecutando experimentos en paralelo, compatible con los algoritmos integrados |
| **Despliegue e inferencia** | Despliegue | Cuatro modos: tiempo real (baja latencia), por lotes (grandes volúmenes sin urgencia), asíncrona (procesos largos sin respuesta inmediata) y serverless (sin gestionar servidores) |
| **Model Monitor** | Monitoreo | Detecta *data drift* (los datos en producción se alejan de los de entrenamiento), problemas de calidad de datos (valores faltantes/atípicos/inconsistentes) y caída de rendimiento del modelo (precisión, recall), con alertas por CloudWatch |
| **Model Registry** | CI/CD | Repositorio centralizado y versionado de modelos, con control de acceso por roles y registro de cambios/aprobaciones para auditoría |
| **Pipelines** | CI/CD | Automatiza y versiona todo el flujo de ML (preprocesamiento → entrenamiento → despliegue) como código reutilizable |
| **Role Manager** | Seguridad | Plantillas para crear y gestionar roles IAM específicos de SageMaker con permisos granulares |
| **Model Cards** | Gobernanza | Documento centralizado por modelo: propósito, datos de entrenamiento, métricas y limitaciones — exportable a PDF |
| **Model Dashboard** | Gobernanza | Portal para visualizar y buscar todos los modelos de la cuenta, con monitoreo de rendimiento y alertas |
| **JumpStart** | Arranque rápido | Biblioteca de modelos fundacionales y soluciones preconfiguradas: explorar, experimentar, personalizar con datos propios y desplegar en pocos clics |
| **Canvas** | Sin código | Construcción de modelos de ML con arrastrar y soltar, sin programar, usando modelos de Bedrock/JumpStart y Data Wrangler integrado |
| **MLflow en SageMaker** | Experimentación | Plataforma open source para rastrear y comparar experimentos (métricas, parámetros); los modelos resultantes se registran en Model Registry y se despliegan en endpoints de SageMaker |
| **Clarify** | Gobernanza/ética | Detecta sesgos en los datos y modelos (ej. desequilibrio de género en el dataset) y explica qué factores pesaron en una decisión del modelo |
| **Ground Truth** | Etiquetado | Crea datasets etiquetados usando etiquetadores humanos (Mechanical Turk o proveedores externos), con flujos de trabajo personalizables y control de calidad |

### SageMaker Clarify — ejemplos prácticos

- **Detección/mitigación de sesgo**: un dataset con 80% hombres / 20% mujeres produce un modelo sesgado hacia los hombres; Clarify detecta el desequilibrio, sugiere balancear los datos (ej. 50/50) y el modelo reentrenado trata a ambos grupos de forma más equitativa.
- **Explicabilidad**: en un modelo de aprobación de préstamos, Clarify puede explicar que una solicitud fue rechazada por un 70% debido al puntaje crediticio bajo y un 30% por la relación deuda/ingreso alta — no solo da el resultado, explica el porqué.

### SageMaker Model Cards — ejemplo

Para un modelo que predice elegibilidad de crédito: descripción del modelo, datos de entrenamiento usados (5 años de historial, +100.000 clientes), métricas (92% precisión, F1 0.89) y limitaciones conocidas (menos preciso con historiales crediticios cortos). Todo centralizado en un documento versionable.

## Comandos clave

Módulo de consola/SDK — sin comandos CLI específicos cubiertos.

## Notas y gotchas

- SageMaker Feature Store separa almacenamiento offline (para entrenar, con histórico completo) y online (para inferencia en tiempo real) pero los mantiene sincronizados — es fácil pensar que son dos sistemas independientes cuando en realidad son dos vistas del mismo dato.
- El "viaje en el tiempo" de Feature Store (recuperar el valor de una feature en un momento pasado) existe específicamente para evitar la fuga de datos futuros (*feature leakage*) al entrenar con datos históricos.
- Model Monitor no sustituye a Clarify: Model Monitor vigila que el modelo en producción se comporte como se espera (drift, calidad de datos, métricas de rendimiento); Clarify analiza sesgo y explicabilidad, normalmente antes o durante el desarrollo del modelo.
- JumpStart y Canvas dejan claro que los datos de entrenamiento/inferencia del usuario no se comparten ni se usan por AWS ni por los proveedores de modelos fundacionales — punto relevante si se manejan datos sensibles.

## Recursos

- [Catálogo de algoritmos integrados — AWS Docs](https://docs.aws.amazon.com/sagemaker/latest/dg/algos.html)
- [SageMaker Pipelines](https://aws.amazon.com/es/sagemaker/pipelines/)
- [SageMaker Model Monitor](https://www.amazonaws.cn/en/sagemaker/model-monitor/)
- [Mejorar la gobernanza de modelos de ML con SageMaker (AWS Blog)](https://aws.amazon.com/es/blogs/machine-learning/improve-governance-of-your-machine-learning-models-with-amazon-sagemaker/)
