# Módulo 40 — Fundamentos de IA Generativa (GenAI)

## Resumen

Bases teóricas de la IA Generativa: qué la diferencia de la IA/ML "tradicional", modelos fundacionales, LLMs, cómo generan texto, y las dos arquitecturas de modelos generativos más relevantes (GANs y modelos de difusión).

### Qué es la IA Generativa

Campo de la IA centrado en **crear contenido nuevo** (conversaciones, imágenes, vídeo, música), a diferencia del ML "clásico" que se centra en tareas específicas como clasificar o predecir. Usa los datos de entrenamiento para resolver problemas que no ha visto antes, en vez de limitarse a repetir un patrón fijo.

Casos de uso: chatbots y asistentes virtuales, generación de imágenes, generación de código, composición musical, síntesis de voz, generación de vídeo (incluye deepfakes y animación generada por IA).

### Modelos fundacionales (FM)

Modelos de IA muy grandes, entrenados con cantidades masivas de datos, capaces de realizar múltiples tareas sin necesidad de reentrenarse por completo (se entrenan una vez en un corpus enorme y luego se afinan para tareas concretas). Entrenar uno desde cero cuesta del orden de millones de dólares.

Ejemplos: BERT (Google), GPT (OpenAI), Claude (Anthropic), DALL-E (OpenAI), LLaMA (Meta).

Referencia de escala: BERT (2018) se entrenó con 340 millones de parámetros sobre ~3.300 millones de tokens (texto sin formato + Wikipedia). GPT-3 se entrenó con más de 175.000 millones de parámetros y 45 TB de datos.

### Large Language Models (LLM)

Modelos fundacionales especializados en comprender, generar e interactuar con lenguaje humano (ej. ChatGPT). Entrenados con enormes volúmenes de texto y miles de millones de parámetros, lo que les permite captar matices y patrones lingüísticos complejos.

**No deterministas**: el mismo prompt puede dar respuestas distintas en cada ejecución — dos usuarios preguntando lo mismo pueden recibir redacciones distintas (aunque coherentes) porque el modelo no repite siempre la misma secuencia de tokens.

**Cómo generan texto**: en cada paso, el modelo predice la siguiente palabra (token) en función del contexto, asignando una probabilidad a cada candidato posible y eligiendo entre los más probables — no "sabe" la respuesta de antemano, la va construyendo token a token.

### Inferencia en el borde (Edge) vs. en la nube

| | SLM en el dispositivo (Edge) | LLM en servidor remoto |
|---|---|---|
| Conexión | Puede funcionar sin Internet | Necesita conexión |
| Latencia | Baja | Más alta |
| Potencia de cómputo | Limitada (tamaño de modelo pequeño) | Alta, permite modelos grandes |
| Cuándo usarlo | Detección rápida en tiempo real (ej. cámara de seguridad en una fábrica con Raspberry Pi) | Predicciones más complejas (ej. reconocimiento facial avanzado) donde se puede tolerar más latencia |

### GANs (Generative Adversarial Networks)

Arquitectura formada por dos redes neuronales que compiten entre sí durante el entrenamiento:

- **Generador**: toma ruido aleatorio como entrada y genera una imagen falsa, intentando imitar el estilo de las imágenes reales de entrenamiento.
- **Discriminador**: recibe imágenes reales y generadas, y aprende a distinguir cuáles son falsas.

El generador mejora tratando de engañar al discriminador, y el discriminador mejora detectando los fallos del generador — ambos se retroalimentan durante el entrenamiento.

### Modelos de difusión

Modelo generativo (muy usado para generar imágenes de alta calidad) basado en simular cómo se dispersan partículas en el espacio:

- **Difusión directa**: convierte progresivamente una imagen real en ruido — se usa para entrenar el modelo.
- **Difusión inversa**: parte de ruido aleatorio y lo transforma paso a paso en una imagen coherente — es lo que se usa para generar contenido nuevo.

### Modelos multimodales

Combinan información de varias modalidades a la vez (texto, imagen, audio, vídeo) para generar salidas más completas y contextuales, en vez de trabajar con un solo tipo de dato. Aplicaciones: análisis conjunto de imagen y texto, reconocimiento de voz y texto, realidad aumentada/virtual.

### Hiperparámetros

Configuraciones externas al modelo (no se aprenden durante el entrenamiento, las fija quien entrena el modelo) que afectan a cómo aprende. Ajustarlos bien ayuda a evitar el overfitting y mejora precisión/eficiencia.

Los más relevantes:

- **Tasa de aprendizaje (Learning Rate)**: si es muy alta, el modelo "aprende demasiado rápido" y puede sacar conclusiones equivocadas; si es muy baja, aprende muy lento.
- **Tamaño del lote (Batch Size)**: lotes grandes necesitan más memoria pero dan resultados más estables; lotes pequeños entrenan más rápido pero de forma menos consistente.
- **Número de épocas (Epochs)**: cuántas veces el modelo ve el dataset completo — hay que calibrarlo para que aprenda sin sobreajustarse.

## Comandos clave

Módulo teórico, sin comandos ni CLI.

## Notas y gotchas

- La diferencia clave entre GenAI y ML "clásico" no es la tecnología sino el objetivo: ML clásico predice/clasifica sobre datos existentes, GenAI produce contenido nuevo que no existía antes.
- Que un LLM sea no determinista no es un bug: es consecuencia directa de cómo genera texto (muestreo probabilístico token a token), y es intencional para que las respuestas no sean siempre idénticas.
- La elección entre inferencia en el borde y en la nube es, en el fondo, un trade-off latencia/conectividad vs. potencia/precisión — no hay una opción "mejor" en general, depende del caso de uso (alarma en tiempo real vs. análisis complejo).
- Difusión directa e inversa es fácil confundirlas: directa = imagen→ruido (para entrenar), inversa = ruido→imagen (para generar).

## Recursos

-
