# Módulo 38 — IA preentrenada de AWS

## Resumen

Catálogo de los servicios de IA/ML de AWS que vienen **ya entrenados** para tareas concretas (visión, voz, lenguaje, recomendaciones, fraude...). No hay que entrenar un modelo desde cero: se usa el servicio directamente o se ajusta con datos propios.

### Entrenamiento desde cero vs. modelo preentrenado

| | Entrenar desde cero | Usar preentrenado |
|---|---|---|
| Flujo | Datos en crudo → limpieza/preprocesamiento → entrenamiento → evaluación y ajustes | Modelo preentrenado disponible → ajustes opcionales con datos propios → integración en la app |
| Ventajas | Control total sobre el modelo | Rapidez de implementación, costes reducidos, menos riesgo, fácil de integrar, aun así personalizable |

### Catálogo de servicios

| Servicio | Qué hace | Casos de uso |
|---|---|---|
| **Amazon Translate** | Traducción automática neuronal, +70 idiomas | Integración con S3/Lambda para traducir contenido en pipelines |
| ↳ *Terminología personalizada* | Permite definir un diccionario propio (por ejemplo, nombres de marca o términos técnicos) para que Translate no los traduzca y los deje tal cual, subiendo un CSV con las columnas del idioma de origen y destino | Mantener nombres de producto/empresa sin traducir en el texto de salida |

Ejemplo de `terminology.csv` (columnas `es`,`en`):

```csv
es, en
"AWS", "Amazon Web Services"
```
| **Amazon Comprehend** | NLP: detecta idioma, extrae frases/entidades clave, analiza sentimiento (positivo/negativo/neutro) | Análisis de opiniones, clasificación de texto |
| **Amazon Transcribe** | Voz → texto (ASR), streaming o por lotes (ej. desde S3) | Elimina PII automáticamente, detecta idioma en audio multilingüe |
| **Amazon Comprehend Medical** | NLP sobre texto clínico no estructurado (notas médicas, resultados de pruebas) | Documentación clínica, análisis de estudios, cumplimiento normativo — resultados en S3 |
| **Amazon Transcribe Medical** | ASR especializado en sector salud, integrado con Comprehend Medical | Dictado y transcripción de consultas médicas, con puntuación y timestamps por palabra |
| **Amazon Polly** | Texto → voz (síntesis realista, varios idiomas y voces) | Lectura de artículos, noticias como audio |
| **Amazon Rekognition** | Reconocimiento de imágenes/vídeo | Ver desglose de funcionalidades abajo |
| **Amazon Textract** | Extrae texto y datos de documentos escaneados (PDFs, imágenes, escritura a mano) | Facturas, reportes financieros, formularios, documentos de identidad |
| **Amazon Kendra** | Motor de búsqueda inteligente sobre repositorios de contenido (PDFs, HTML, Word, FAQs) | Responder preguntas en lenguaje natural, centralizar documentación interna |
| **Amazon Lex** | ASR + NLU: convierte voz en texto y reconoce la intención detrás | Chatbots y asistentes de voz |
| **Amazon Connect** | Centro de contacto en la nube (llamadas, chat, gestión de agentes) | Se combina con Lex para IVR conversacional |
| **Amazon Personalize** | Recomendaciones personalizadas en tiempo real (usado internamente por amazon.com) | Recomendación de productos, contenido, integrable en web/app/email/SMS |
| **Amazon Fraud Detector** | Detección de fraude con modelos de ML | E-commerce, cuentas falsas, préstamos, pagos |
| **Amazon CodeGuru** | Mejora de calidad y rendimiento de código con IA | **Reviewer**: recomendaciones de código; **Profiler**: detecta cuellos de botella |
| **Amazon Mechanical Turk (MTurk)** | Marketplace de tareas humanas simples ("Human Intelligence Tasks") | Clasificación de imágenes, transcripción, etiquetado de datos |
| **Amazon Augmented AI (A2I)** | Añade revisión humana a predicciones automatizadas de baja confianza | Moderación de contenido, control de calidad |
| **AWS DeepRacer** | Coche autónomo a escala 1/18 para aprender aprendizaje por refuerzo | Entrenar modelos en simulador, competir en la DeepRacer League |

### Funcionalidades de Amazon Rekognition

| Funcionalidad | Qué hace |
|---|---|
| Detección de etiquetas | Identifica objetos, escenas y actividades en una imagen |
| Propiedades de la imagen | Extrae atributos como colores dominantes, brillo, nitidez |
| Moderación de imágenes | Detecta contenido inapropiado, no deseado u ofensivo |
| Análisis facial | Detecta rasgos faciales (edad aproximada, emociones, si lleva gafas, etc.) |
| Comparación de rostros | Compara dos caras y da una puntuación de similitud (verificación de identidad) |
| Face Liveness | Comprueba que hay una persona real delante de la cámara, no una foto o vídeo, para evitar suplantación |
| Reconocimiento de famosos | Identifica personas públicas/famosas en imágenes y vídeos |
| Texto en la imagen | Detecta y extrae texto presente en una imagen (similar a OCR) |
| Detección de EPI | Detecta si las personas llevan equipo de protección individual (cascos, guantes, mascarillas...) |
| Análisis de vídeos almacenados | Procesa vídeos ya guardados (ej. en S3) de forma asíncrona |
| Transmisión de eventos de vídeo | Analiza streaming de vídeo en tiempo real |

### Rekognition + revisión humana (A2I)

Flujo típico de moderación de contenido con control de calidad: imágenes etiquetadas de entrenamiento → Amazon Rekognition analiza → si hay dudas, pasa por revisión humana opcional vía Amazon Augmented AI → aprobación o rechazo final. Rekognition permite adaptadores de moderación personalizados entrenados con imágenes etiquetadas propias para mejorar la precisión en un caso de uso concreto.

### Amazon Lex + Connect — flujo de ejemplo

Caso típico (reprogramar una cita): el usuario llama → **Amazon Connect** gestiona la llamada y usa **Amazon Lex** para interactuar por voz → al reconocer la intención, se activa una función **Lambda** que consulta la base de datos de programación → Connect confirma la nueva cita al usuario por SMS.

## Comandos clave

Módulo teórico, sin comandos ni CLI — es un catálogo de servicios gestionados que se consumen vía SDK/consola.

## Notas y gotchas

- Casi todos estos servicios siguen el mismo patrón: subes datos (texto, imagen, audio) a S3 o los envías directamente, AWS los procesa con un modelo ya entrenado y te devuelve el resultado — no hay que gestionar infraestructura de ML.
- Transcribe Medical y Comprehend Medical son servicios separados (voz→texto y NLP respectivamente) pero pensados para encadenarse: primero transcribes la consulta médica, luego extraes información clínica del texto resultante.
- A2I no es un servicio de ML en sí — es la pieza que añade un humano en el bucle cuando la confianza del modelo es baja, útil para cumplimiento o control de calidad antes de tomar una decisión automática.
- DeepRacer es la vía "hands-on" para aprender aprendizaje por refuerzo sin montar infraestructura propia: entrenas en simulador y despliegas el mismo modelo en el coche físico.

## Recursos

-
