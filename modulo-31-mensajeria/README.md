# Módulo 31 — Mensajería en AWS (SQS y SNS)

## Resumen

### Por qué desacoplar aplicaciones
- Dos patrones de comunicación entre apps: **síncrono** (app llama directamente a app) o **asíncrono/basado en eventos** (app → cola → app).
- La comunicación síncrona sufre con picos de tráfico (ej. pasar de 50 a 5000 vídeos a codificar de golpe). Desacoplar con colas/streams permite que cada parte escale de forma independiente.
- Tres modelos: **SQS** (cola), **SNS** (pub/sub), **Kinesis** (streaming en tiempo real).

## Amazon SQS (Simple Queue Service)

### Qué es
- Servicio de colas gestionado para desacoplar componentes: un productor envía mensajes, uno o varios consumidores los reciben y procesan.
- **Cola estándar**: rendimiento y nº de mensajes ilimitados, retención 4 días por defecto (máx. 14), latencia <10ms, hasta 256 KB por mensaje. Puede haber **duplicados** (al menos una entrega) y **desorden** (orden de mejor esfuerzo).

### Producir y consumir mensajes
- Producir: `SendMessage`, con atributos propios (ID de vídeo, ID de curso...) hasta 256 KB.
- Consumir: sondear la cola (hasta 10 mensajes por `ReceiveMessage`), procesar, y borrar explícitamente con `DeleteMessage`.
- Se puede escalar horizontalmente el número de consumidores (varias EC2) para procesar en paralelo.

### SQS + Auto Scaling Group
- La métrica de CloudWatch `ApproximateNumberOfMessages` (longitud de la cola) dispara una alarma que escala el ASG de consumidores hacia arriba o abajo según la demanda.

### Patrón de desacople de capas
- Front-end → `SendMessage` → cola SQS (infinitamente escalable) → back-end hace `ReceiveMessage` y procesa → guarda resultado en S3.
- El front-end y el back-end escalan de forma completamente independiente entre sí.

### Seguridad
- Cifrado en tránsito (HTTPS), en reposo (KMS) o del lado del cliente.
- Políticas de acceso a la cola (similares a las de bucket S3): permiten acceso cross-account o que otros servicios (SNS, S3...) escriban en la cola, restringiendo por `aws:SourceArn`/`aws:SourceAccount`.

### Tiempo de espera de visibilidad (Visibility Timeout)
- Al sondear un mensaje, este se vuelve invisible para otros consumidores durante un tiempo (30s por defecto) — tiempo que tiene el consumidor para procesarlo y borrarlo.
- Si no se procesa a tiempo, el mensaje vuelve a estar visible y **puede procesarse dos veces**.
- `ChangeMessageVisibility` permite pedir más tiempo si el procesamiento va a tardar.
- Timeout muy alto + consumidor bloqueado = reprocesamiento lento; timeout muy bajo = más duplicados.

### Dead Letter Queue (DLQ)
- Si un mensaje se recibe sin borrarse más veces que `MaximumReceives`, pasa a una DLQ — útil para depurar mensajes problemáticos sin bloquear el flujo normal.
- La DLQ de una cola FIFO debe ser FIFO; la de una estándar, estándar.
- Conviene dar a la DLQ una retención larga (14 días) y procesarla antes de que expire.
- **Redirigir al origen**: una vez arreglado el problema, se pueden reenviar los mensajes de la DLQ de vuelta a la cola original (o a otra) por lotes, sin código personalizado, mediante una política de redirección.

### Cola con retraso (Delay Queue)
- Retrasa la visibilidad de un mensaje hasta 15 minutos — configurable por cola (`DelaySeconds` por defecto) o por mensaje al enviarlo.

### Sondeo largo (Long Polling)
- El consumidor puede "esperar" a que lleguen mensajes en vez de recibir una respuesta vacía inmediata — reduce el nº de llamadas a la API y la latencia.
- `WaitTimeSeconds`: 1-20 segundos (se recomienda 20). Preferible siempre al sondeo corto.

### Cliente extendido de SQS
- Para mensajes de más de 256 KB (ej. 1 GB): la librería cliente extendido (Java) guarda el payload grande en S3 y envía solo un mensaje pequeño con los metadatos/referencia por SQS.

### Colas FIFO
- Garantizan orden y **exactamente una entrega** (deduplicando), a cambio de rendimiento limitado (300 msg/s sin batching, 3000 msg/s con batching).
- **Deduplicación** (ventana de 5 min): por contenido (hash SHA-256 del cuerpo) o por un `MessageDeduplicationId` explícito.
- **Agrupación de mensajes** (`MessageGroupID`): los mensajes de un mismo grupo se procesan en orden por un único consumidor a la vez; distintos grupos se procesan en paralelo (sin orden garantizado entre grupos).

## Amazon SNS (Simple Notification Service)

### Qué es
- Modelo **pub/sub**: un productor publica en un **topic** SNS, y cualquier número de suscriptores recibe cada mensaje — hasta 12.5M de suscripciones por topic, 100.000 topics.
- Suscriptores posibles: SQS, Lambda, Kinesis Data Firehose, email, SMS, HTTP(S), notificaciones push móviles.
- Muchos servicios AWS publican directamente a SNS: CloudWatch Alarms, AWS Budgets, Auto Scaling Group, eventos de S3, DynamoDB, CloudFormation, RDS Events, AWS DMS.

### Cómo publicar
- **Publicación de topic** (vía SDK): crear topic → crear suscripción(es) → publicar mensajes.
- **Publicación directa** (apps móviles): crear una plataforma → un endpoint de plataforma → publicar en ese endpoint (integra con Google GCM, Apple APNS...).

### Seguridad
- Igual que SQS: cifrado en tránsito/reposo/cliente, políticas de acceso tipo bucket-policy para acceso cross-account o para que otros servicios publiquen en el topic.

### Fan-out (SNS + SQS)
- Publicar **una vez** en SNS y que llegue a **todas** las colas SQS suscritas — totalmente desacoplado, sin pérdida de datos, con las ventajas de SQS (persistencia, procesamiento diferido, reintentos) en cada rama.
- Se pueden añadir más suscriptores SQS con el tiempo sin tocar el productor.
- La política de acceso de cada cola SQS debe permitir explícitamente que SNS le escriba.

### Casos de uso de fan-out
- **Eventos S3 a varias colas**: S3 solo permite una regla de evento por combinación tipo+prefijo — para mandar el mismo evento a varias colas, se pasa por un topic SNS que hace fan-out.
- **SNS → Kinesis Data Firehose → S3**: para persistir/archivar en S3 (u otro destino compatible con Firehose) todo lo publicado en un topic.

### SNS FIFO
- Igual que SQS FIFO pero a nivel de topic: orden por `MessageGroupID`, deduplicación por ID o por contenido.
- Solo admite **colas SQS FIFO** como suscriptores; mismo límite de rendimiento que SQS FIFO.
- **Fan-out con orden y deduplicación**: SNS FIFO + varias colas SQS FIFO suscritas.

### Filtrado de mensajes
- Una **política de filtro** (JSON) por suscripción decide qué mensajes le llegan a esa suscripción concreta, según sus atributos (ej. `estado: Cancelado`).
- Una suscripción sin política de filtro recibe **todos** los mensajes del topic.

## Comandos clave

```bash
# Crear una cola estándar
aws sqs create-queue --queue-name <queue-name> \
  --attributes "DelaySeconds=0,MessageRetentionPeriod=345600"

# Crear una cola FIFO
aws sqs create-queue --queue-name <queue-name>.fifo \
  --attributes "FifoQueue=true,ContentBasedDeduplication=true"

# Enviar un mensaje (con retraso opcional)
aws sqs send-message --queue-url <queue-url> --message-body "<mensaje>" --delay-seconds <segundos>

# Recibir mensajes (sondeo largo)
aws sqs receive-message --queue-url <queue-url> --max-number-of-messages 10 --wait-time-seconds 20

# Borrar un mensaje ya procesado
aws sqs delete-message --queue-url <queue-url> --receipt-handle <receipt-handle>

# Extender el tiempo de visibilidad de un mensaje en proceso
aws sqs change-message-visibility --queue-url <queue-url> --receipt-handle <receipt-handle> --visibility-timeout <segundos>

# Vaciar una cola por completo
aws sqs purge-queue --queue-url <queue-url>

# Configurar una DLQ (redrive policy) sobre una cola existente
aws sqs set-queue-attributes --queue-url <queue-url> \
  --attributes '{"RedrivePolicy":"{\"deadLetterTargetArn\":\"<dlq-arn>\",\"maxReceiveCount\":\"5\"}"}'

# Crear un topic SNS
aws sns create-topic --name <topic-name>

# Suscribir una cola SQS a un topic
aws sns subscribe --topic-arn <topic-arn> --protocol sqs --notification-endpoint <queue-arn>

# Publicar un mensaje en un topic
aws sns publish --topic-arn <topic-arn> --message "<mensaje>"

# Aplicar una política de filtrado a una suscripción
aws sns set-subscription-attributes \
  --subscription-arn <subscription-arn> --attribute-name FilterPolicy \
  --attribute-value '{"<atributo>":["<valor>"]}'
```

## Notas y gotchas

- Un tiempo de visibilidad mal ajustado es la causa más común de mensajes duplicados o de reprocesamiento lento en SQS — ni "cuanto más alto mejor" ni "cuanto más bajo mejor", depende de cuánto tarda realmente el consumidor.
- Fan-out (SNS + varias SQS) es la respuesta por defecto cuando "un evento tiene que llegar a varios sistemas distintos" — evita acoplar el productor a la lista de consumidores, que pueden crecer con el tiempo sin tocar nada del origen.
- Una cola/topic FIFO no es "la versión mejor" de la estándar — tiene un límite de rendimiento mucho menor; se elige FIFO solo cuando el orden y la deduplicación exacta son requisitos reales del negocio.
- Sin `MessageGroupID` distintos, una cola FIFO solo permite un consumidor activo a la vez — es fácil perder paralelismo por no pensar en los grupos desde el diseño.
- Una suscripción SNS sin política de filtro es indistinguible de "recibe todo" — si algo recibe mensajes que no debería, lo primero a revisar es si le falta el filtro, no el productor.

## Recursos

- https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/welcome.html
- https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-dead-letter-queues.html
- https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/FIFO-queues.html
- https://docs.aws.amazon.com/sns/latest/dg/welcome.html
- https://docs.aws.amazon.com/sns/latest/dg/sns-message-filtering.html
- https://docs.aws.amazon.com/sns/latest/dg/sns-sqs-as-subscriber.html
