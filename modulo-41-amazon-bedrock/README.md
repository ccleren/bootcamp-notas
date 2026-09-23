# Módulo 41 — Amazon Bedrock

## Resumen

Amazon Bedrock es el servicio totalmente administrado de AWS para construir aplicaciones de IA Generativa: da acceso a una selección de modelos fundacionales (propios de Amazon y de terceros) vía API, sin tener que entrenarlos ni gestionar la infraestructura por debajo. Se pueden usar tal cual o personalizarlos con datos propios.

### Visión general

- Antes de usar un modelo hay que **solicitar acceso**: los modelos Amazon (Titan) se aprueban al instante, los de terceros pueden pedir información adicional — la aprobación tarda solo unos minutos.
- El coste de los modelos de terceros se factura a través de AWS, a las tarifas de ese proveedor.
- No hace falta entrenar nada para empezar a usarlos; personalizarlos con datos propios es opcional.

### Modelos disponibles (comparativa)

| Modelo | Ventana de contexto | Especialidad | Precio (por 1K tokens) |
|---|---|---|---|
| Amazon Titan Text Express | 8K tokens | Texto multiidioma (+100 idiomas), alto rendimiento | Entrada $0.0008 / Salida $0.0016 |
| Llama 2 70b-chat | 4K tokens | Diálogo y tareas a gran escala, principalmente en inglés | Entrada $0.0019 / Salida $0.0025 |
| Claude 2.1 | 200K tokens | Análisis, pronóstico, comparación de documentos | Entrada $0.008 / Salida $0.024 |
| Stable Diffusion (SDXL 1.0) | 77 tokens/prompt | Generación de imágenes realistas | $0.04–$0.08 / imagen |

Una ventana de contexto grande permite tener en cuenta más información a la vez y da respuestas más coherentes, a cambio de más memoria y capacidad de proceso. Referencia de otros modelos: Gemini 1.0 Pro (32K), GPT-4 Turbo (128K), Gemini 1.5 Pro (1M).

### Personalización de modelos

| Técnica | Qué hace | Cuándo usarla |
|---|---|---|
| **Fine-Tuning** | Ajusta un modelo ya entrenado con datos propios etiquetados (pares prompt/completion) para una tarea concreta | Adaptar el modelo a resumen, Q&A, clasificación... — más económico que reentrenar desde cero |
| **Continuous Pre-training** | Sigue entrenando el modelo con datos sin etiquetar de un dominio concreto (aplicable a Titan Text Express y Lite) | Dar conocimiento profundo de un dominio (medicina, finanzas, legal) sin necesidad de etiquetar datos |
| **Transfer Learning** | Concepto general de ML: reutilizar un modelo entrenado en una tarea para aplicarlo a otra relacionada. Fine-Tuning es un caso concreto de Transfer Learning | — |

Formato de datos de entrenamiento (JSON Lines):

```json
// Fine-tuning: text-to-text
{"prompt": "what is AWS", "completion": "it's Amazon Web Services"}

// Continued Pre-training: text-to-text
{"input": "AWS stands for Amazon Web Services"}
```

Reentrenar un modelo fundacional completo requiere mucha más inversión, datos y cómputo que un Fine-Tuning, que solo ajusta el modelo para una tarea concreta con menos datos y recursos.

### Bases de conocimiento y RAG

Las **bases de conocimiento** permiten dar a un modelo fundacional contexto de fuentes de datos privadas de la empresa (S3, SharePoint, web crawler...) para que sus respuestas sean más precisas y personalizadas.

**RAG (Retrieval-Augmented Generation)**: técnica que recupera información relevante de esas fuentes de datos y la añade a la consulta antes de enviarla al modelo, en vez de depender solo de lo que el modelo aprendió en su entrenamiento. Flujo: consulta del usuario → búsqueda en la base de conocimiento → información relevante recuperada → consulta + contexto se envían al modelo fundacional → respuesta generada.

**Bases de datos vectoriales para RAG**: almacenan los documentos como *embeddings* (vectores numéricos) para poder buscar por similitud semántica en vez de por coincidencia exacta de texto.

| Servicio AWS | Tipo | Nota |
|---|---|---|
| Amazon OpenSearch Service | Búsqueda y análisis | Consultas de similitud en tiempo real, millones de embeddings |
| Amazon Aurora | Relacional | Con la escalabilidad de la infraestructura de AWS |
| Amazon Neptune | Grafos | Para relaciones complejas entre datos |
| Amazon DocumentDB | NoSQL (compatible MongoDB) | Consultas de similitud en tiempo real |
| Amazon RDS para PostgreSQL | Relacional open source | Compatible con extensiones vectoriales de PostgreSQL |

### Guardrails (barreras de protección)

Mecanismos de Bedrock para un uso seguro, ético y responsable de los modelos: permiten definir **temas denegados**, **filtros de contenido** dañino (en entrada y salida), umbrales contra jailbreaks e inyección de prompts, y eliminación de PII/datos sensibles.

### Agentes de Amazon Bedrock

Un agente es un "asistente virtual" configurable que interactúa con el usuario y ejecuta tareas conectándose a bases de datos, APIs externas, funciones Lambda o bases de conocimiento — automatiza flujos de trabajo en lugar de solo responder texto.

Orquestación: el agente recibe una tarea → construye un prompt (con historial, instrucciones, acciones y bases de conocimiento disponibles) → el modelo genera una **cadena de pensamiento** dividida en pasos → en cada paso puede llamar a una API, buscar en una base de conocimiento o ejecutar una acción → combina los resultados → devuelve la respuesta final.

### Evaluación de modelos

Permite comparar varios modelos fundacionales entre sí para elegir el más adecuado, con dos enfoques:

- **Evaluación automática**: se envían preguntas de referencia al modelo, sus respuestas se comparan contra respuestas de referencia con un algoritmo de evaluación (métricas como precisión, robustez, toxicidad), usando datasets integrados (ej. BoolQ) o propios.
- **Evaluación humana**: para criterios subjetivos que no se pueden automatizar — empleados propios o un equipo gestionado por AWS puntúan las respuestas (pulgar arriba/abajo, escala 1-5, elegir entre respuestas).

Solo se puede elegir un `taskType` (tarea a evaluar, ej. resumen de texto) por cada trabajo de evaluación.

**Métricas de calidad de texto:**

| Métrica | Qué mide | Ejemplo |
|---|---|---|
| **ROUGE** | Calidad de resúmenes/traducciones. ROUGE-N: n-gramas coincidentes; ROUGE-L: subsecuencia común más larga | Compara "El gato persigue al ratón..." vs. el texto de referencia |
| **BLEU** | Calidad de traducciones, comparando n-gramas de distintos tamaños (no solo palabras sueltas, también secuencias) | Traducción generada vs. traducción humana de referencia |
| **BERTScore** | Similitud **semántica** entre texto generado y de referencia, aunque usen palabras distintas | "El automóvil aceleró rápidamente" ≈ "El coche se movió velozmente" |

### Tokenización, ventana de contexto y embeddings

- **Token**: unidad mínima de significado que usa el modelo (palabra, subpalabra o símbolo). Los modelos no trabajan con texto plano, sino con tokens convertidos a valores numéricos.
- Puede tokenizarse por **palabra completa** (`"El", "gato", "está", "durmiendo"`) o por **subpalabras** (`"increíblemente"` → `"incre", "íble", "mente"`), lo que permite procesar palabras que no están en el vocabulario del modelo.
- **Embeddings**: representación vectorial numérica de un token/palabra/frase. Palabras con significado relacionado (ej. "luz" y "lámpara") generan vectores cercanos en el espacio vectorial — es la base de la búsqueda semántica y del RAG.

### PartyRock

Playground de Amazon Bedrock para crear aplicaciones de IA Generativa con una interfaz web, sin necesidad de cuenta de AWS ni pasar por la consola.

### Bedrock + CloudWatch

Cada invocación de un modelo (texto, imágenes, embeddings) puede registrarse: los logs se envían a CloudWatch Logs y también se guardan en S3, y se pueden crear alarmas de CloudWatch sobre comportamientos inusuales.

### Precios

- **Bajo demanda**: se cobra por tokens procesados (texto) o por imagen generada.
- **Por lotes**: varias predicciones a la vez, la salida se guarda en S3, con hasta un 50% de descuento.
- **Rendimiento aprovisionado**: cobro por hora con compromiso (1, 6 meses...), obligatorio para modelos personalizados.
- **Importación de modelos**: la importación en sí es gratis, se cobra por la inferencia bajo demanda.
- **Personalización**: se cobra por tokens durante el entrenamiento + almacenamiento mensual del modelo resultante.
- **Evaluación**: automática = solo coste de inferencia; humana = coste adicional por tarea.

Coste según técnica (de menor a mayor): **Prompt Engineering** (no requiere entrenamiento) < **RAG** (el FM no cambia, no hay cómputo de entrenamiento) < **Fine-Tuning basado en instrucciones** (el FM se ajusta con instrucciones específicas) < **Fine-Tuning de adaptación de dominio** (alta computación para adaptar el FM por completo).

⚠️ Los precios cambian con el tiempo — revisar siempre la [página oficial de precios](https://aws.amazon.com/es/bedrock/pricing/) antes de dar por buena una cifra.

## Comandos clave

Módulo teórico y de consola — no se usaron comandos CLI específicos.

## Notas y gotchas

- El orden de coste (Prompt Engineering → RAG → Fine-Tuning por instrucciones → Fine-Tuning de dominio) es también, en gran medida, el orden en que conviene probar soluciones: empezar por ajustar el prompt antes de plantearse entrenar nada.
- RAG no modifica el modelo fundacional en ningún momento — solo enriquece la consulta con contexto recuperado. Fine-Tuning sí ajusta los pesos del modelo. Son técnicas complementarias, no alternativas excluyentes.
- Continuous Pre-training solo está disponible para Titan Text Express y Lite, no para cualquier modelo de Bedrock — es una limitación a tener en cuenta al elegir modelo si se planea hacer preentrenamiento continuo.
- ROUGE/BLEU comparan textos por coincidencia de palabras/n-gramas; BERTScore compara por significado — dos frases pueden tener BLEU bajo y BERTScore alto si dicen lo mismo con palabras distintas.

## Recursos

- [Preparar datos para personalizar un modelo — AWS Docs](https://docs.aws.amazon.com/es_es/bedrock/latest/userguide/model-customization-prepare.html)
- [Tareas de evaluación de modelos — AWS Docs](https://docs.aws.amazon.com/bedrock/latest/userguide/model-evaluation-tasks.html)
- [Precios de Amazon Bedrock](https://aws.amazon.com/es/bedrock/pricing/)
