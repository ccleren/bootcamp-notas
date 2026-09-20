# Módulo 37 — Fundamentos de IA y Machine Learning (ML)

## Resumen

Módulo teórico que sienta las bases de IA/ML antes de entrar en los servicios de IA de AWS. Cubre la jerarquía de conceptos, los tipos de aprendizaje, cómo se entrena y evalúa un modelo, y una introducción a redes neuronales y NLP.

### Jerarquía de conceptos

`IA ⊃ ML ⊃ Deep Learning`, y por separado la **IA Generativa** (modelos fundacionales tipo GPT que generan contenido nuevo, no solo clasifican o predicen).

- **IA**: cualquier sistema que simula capacidades humanas (razonar, reconocer patrones, tomar decisiones). Ejemplo clásico: Deep Blue venciendo a Kasparov jugando por reglas, sin "aprender".
- **ML**: subconjunto de la IA que aprende patrones a partir de datos en lugar de seguir reglas programadas explícitamente. Ejemplo típico: distinguir un muffin de un chihuahua a partir de miles de fotos etiquetadas, no de reglas de "si tiene ojos negros redondos...".
- **Deep Learning**: subconjunto de ML basado en redes neuronales con varias capas; necesita muchos datos y capacidad de cómputo (normalmente GPU).
- **IA Generativa**: modelos fundacionales que generan texto, imágenes, audio o código nuevo (ChatGPT, etc.), en vez de solo predecir una etiqueta o un valor.

Casos de uso típicos: asistentes virtuales (Siri, Alexa, Google Assistant), recomendaciones (Netflix, Amazon, Spotify), reconocimiento facial (Amazon Rekognition, Google Cloud Vision, Face++), conducción autónoma (Tesla Autopilot, Waymo).

### Qué es un modelo

Una representación matemática entrenada con datos (etiquetados o no) que aprende una función interna (parámetros + estructura) para hacer predicciones sobre datos nuevos.

### Formato de los datos

| Tipo | Descripción |
|---|---|
| Estructurados | Formato predefinido y consistente, normalmente en tablas de BBDD relacionales |
| No estructurados | Sin formato fijo: texto libre, imágenes, audio, vídeo — más difíciles de procesar |
| Etiquetados | Cada instancia tiene una categoría/clase asociada — necesarios para aprendizaje supervisado |
| No etiquetados | Sin categoría asignada — se usan en aprendizaje no supervisado, el algoritmo busca los patrones solo |

### Entrenamiento, validación y test

Un dataset se divide normalmente así:

- **Training set** (60-80%): con lo que el modelo aprende los patrones.
- **Validation set** (10-20%): para ajustar el modelo durante el desarrollo, comparando predicciones contra el valor real conocido.
- **Test set** (10-20%): evaluación final con datos que el modelo no ha visto nunca, simula el rendimiento en producción.

Ejemplo del curso: un modelo que predice si un jugador de fútbol es un buen fichaje (entrenado con estadísticas de jugadores reales — goles, asistencias, años de experiencia). Con 100.000 filas: 70k de training (88% accuracy), 20k de validación (82% accuracy), 10k de test (85% accuracy). La diferencia entre estos porcentajes es la señal de si el modelo generaliza bien o no.

### Tipos de aprendizaje

| Tipo | Idea | Casos de uso |
|---|---|---|
| **Supervisado** | Se entrena con datos etiquetados; aprende a mapear entrada → salida | Clasificación (detección de spam, clasificación de imágenes), regresión (predicción de precios, pronóstico del tiempo) |
| **No supervisado** | Encuentra patrones/estructura en datos sin etiquetar | Clustering (objetivos de marketing), reducción de dimensionalidad (compresión de imágenes)|
| **Por refuerzo** | Un agente aprende por ensayo y error interactuando con un entorno, maximizando una recompensa | Robótica, conducción autónoma, videojuegos |

**Clasificación vs regresión** (dentro de supervisado): clasificación asigna una categoría (¿es spam? sí/no), regresión predice un valor continuo (precio de una casa).

**Clasificación binaria**: caso particular de clasificación con solo dos clases posibles (sí/no, 0/1) — diagnóstico médico, detección de spam, etc.

**RLHF** (Reinforcement Learning from Human Feedback): variante de aprendizaje por refuerzo donde el feedback viene de evaluaciones humanas en vez de una función de recompensa automática. Se usa para ajustar modelos de lenguaje y que sus respuestas sean más útiles y estén mejor alineadas con lo que espera un humano.

### Overfitting, underfitting y balance bias/varianza

- **Overfitting (sobreajuste)**: el modelo se ajusta demasiado a los datos de entrenamiento y falla con datos nuevos. Asociado a **varianza alta**.
- **Underfitting (subajuste)**: el modelo es demasiado simple y ni siquiera rinde bien con los datos de entrenamiento. Asociado a **bias (sesgo) alto**.
- **Balanceado**: el punto ideal donde el modelo generaliza bien.

| | Causa | Solución |
|---|---|---|
| Bias alto → underfitting | Modelo demasiado simple | Aumentar la complejidad del modelo |
| Varianza alta → overfitting | Modelo demasiado sensible a los datos de entrenamiento | Simplificar el modelo, regularización, más datos |

### Redes neuronales y Deep Learning

Una red neuronal se organiza en capas: **entrada** (recibe los datos), **ocultas** (procesan y aprenden representaciones intermedias) y **salida** (produce el resultado). Cada neurona recibe entradas, procesa y pasa una salida a la siguiente capa — una red puede tener miles de millones de neuronas.

- **CNN (Convolutional Neural Network)**: pensada para datos en forma de cuadrícula (imágenes). Usa filtros (kernels) que detectan bordes, texturas y patrones locales, evitando tener que diseñar manualmente qué características mirar.
- **RNN (Recurrent Neural Network)**: pensada para datos secuenciales/temporales (texto, series temporales, vídeo). Tiene bucles internos que retroalimentan la salida de una neurona a sí misma, permitiendo "recordar" entradas anteriores — útil para traducción o generación de texto.

### NLP (Natural Language Processing)

Rama de la IA centrada en que las máquinas entiendan y generen lenguaje humano. Pasos habituales:

- **Tokenización**: dividir el texto en palabras/unidades (`"Los gatos persiguen..."` → `["Los", "gatos", "persiguen", ...]`).
- **Lematización / Stemming**: reducir cada palabra a su raíz o forma base.
- **Modelo de lenguaje**: modelo estadístico que predice la probabilidad de una secuencia de palabras.

Casos de uso: asistentes de voz, análisis de sentimiento, traducción automática.

**Modelo Transformer**: arquitectura de NLP que procesa la frase completa en paralelo (no palabra a palabra como una RNN), usando **self-attention** para evaluar qué peso tiene cada palabra respecto a las demás en el contexto. Es la base de modelos como ChatGPT o BERT.

### Evaluación de modelos de clasificación: matriz de confusión

Compara lo predicho contra lo real en cuatro categorías:

| | Predicción Positivo | Predicción Negativo |
|---|---|---|
| **Real Positivo** | Verdadero Positivo (TP) | Falso Negativo (FN) |
| **Real Negativo** | Falso Positivo (FP) | Verdadero Negativo (TN) |

A partir de ahí:

- **Precision** = TP / (TP + FP) — de lo que predije como positivo, cuánto acerté.
- **Recall** = TP / (TP + FN) — de todos los positivos reales, cuántos detecté.
- **F1** = 2 · (Precision · Recall) / (Precision + Recall) — balance entre ambas.
- **Accuracy** = (TP + TN) / (TP + TN + FP + FN) — acierto global.

**AUC-ROC**: mide la capacidad del modelo para distinguir entre clases. La curva ROC compara tasa de verdaderos positivos vs tasa de falsos positivos; el AUC es el área bajo esa curva (1.0 = modelo perfecto, 0.5 = equivalente al azar). Útil para comparar modelos de clasificación entre sí.

### Evaluación de modelos de regresión

| Métrica | Qué mide |
|---|---|
| MSE | Error cuadrático medio — penaliza más los errores grandes |
| RMSE | Raíz del MSE — en la misma unidad que los datos originales |
| MAE | Error absoluto medio |
| R² | Qué porcentaje de la variación de los datos explica el modelo |
| MAPE | Error absoluto medio en porcentaje, útil para comparar entre escalas distintas |

### Conceptos de gobernanza de IA

- **Responsabilidad**: sistemas transparentes, justos y sin sesgos, evaluando riesgos y evitando daños no intencionados.
- **Cumplimiento**: adherencia a leyes y normativas del sector (salud, finanzas, legal).
- **Gobernanza**: políticas y supervisión para que la IA aporte valor de forma alineada con los objetivos de la organización.
- **Seguridad**: protección contra amenazas, accesos no autorizados y riesgos a la integridad/privacidad de los datos.

## Comandos clave

Módulo teórico, sin comandos ni CLI.

## Notas y gotchas

- No todo problema necesita ML: si se puede resolver con una fórmula o regla determinista (p. ej. convertir km a millas), usar ML es sobrecoste innecesario. ML tiene sentido cuando el patrón es complejo o no es obvio.
- Un ratio 88% / 82% / 85% entre training/validation/test no es alarmante por sí solo — lo que hay que vigilar es una caída grande entre training y validation/test, señal de overfitting.
- Precision y Recall suelen ir en tensión: subir uno tiende a bajar el otro, por eso se usa F1 como métrica de compromiso cuando las clases están desbalanceadas.
- CNN para datos con estructura espacial (imágenes), RNN para datos con estructura secuencial (texto, series temporales) — la arquitectura se elige según el tipo de dato, no al revés.

## Recursos

- [Transformers en IA — AWS](https://aws.amazon.com/es/what-is/transformers-in-artificial-intelligence/)
- [Overfitting — MathWorks](https://la.mathworks.com/discovery/overfitting.html)
