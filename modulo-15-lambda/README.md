# Módulo 15 — AWS Lambda

## Resumen

### De arquitectura monolítica a serverless
- Una app monolítica encadena capas (subida → procesamiento → almacenamiento): si una capa falla o se satura, afecta a todas las demás, y escala en bloque aunque solo una parte lo necesite.
- Primera evolución: meter una **cola** entre capas (ej. SQS) — desacopla el productor del consumidor y permite escalar el procesamiento de 0 a N según la longitud de la cola.
- Evolución completa: **arquitectura orientada a eventos** — un productor emite un evento ("se subió un video"), un enrutador de eventos (ej. EventBridge) lo filtra/clasifica, y los consumidores interesados reaccionan sin que el productor sepa quién los escucha.

### ¿Qué es serverless?
- Paradigma en el que **no gestionas servidores** — no significa que no existan, significa que no los aprovisionas ni los ves.
- Solo despliegas código/funciones, que escalan automáticamente según el uso.
- Empezó siendo sinónimo de FaaS (Function as a Service); hoy incluye BBDD, mensajería, almacenamiento, notificaciones, etc.
- Pagas por lo que usas, no por tener el servidor encendido.
- Ejemplos en AWS: Lambda, Fargate, App Runner, EventBridge (cómputo); DynamoDB, S3 (almacenamiento); API Gateway, SNS, SQS (API/mensajería); Rekognition, Textract, Comprehend (IA/ML).

### Introducción a AWS Lambda
- Ejecuta código sin administrar servidores, pensado para tareas cortas y específicas.
- Responde a eventos (S3, SNS, API Gateway...) o se invoca directamente.
- Corre sobre un **runtime** (Python, Node.js, Java, etc.); la CPU asignada depende de la memoria configurada.
- Solo se paga por la duración real de ejecución.
- Runtimes administrados por AWS (actualizaciones de seguridad automáticas) vs. **Custom Runtime** (empaquetas tu propio entorno como imagen Docker para lenguajes no soportados, ej. Rust, Elixir, PHP).

### Límites clave
- Tiempo máximo de ejecución: **900 segundos (15 min)**.
- Memoria: 128 MB – 10.240 MB (10 GB), en incrementos de 1 MB.
- Paquete de despliegue: 50 MB comprimido (.zip) / 250 MB descomprimido.
- Espacio `/tmp`: hasta 10 GB.
- Concurrencia por defecto: 1.000 ejecuciones por región y cuenta.
- Variables de entorno: 4 KB.

### Lambda vs. otros servicios de cómputo
| Servicio | Cuándo usarlo |
|---|---|
| **Lambda** | Tareas/procesos por eventos, corta duración, minimizar coste por inactividad |
| **Fargate** | Contenedores con más control de configuración, microservicios de larga duración con escalado horizontal |
| **Elastic Beanstalk** | PaaS para apps web, despliegue rápido con auto scaling predefinido |
| **EC2** | Necesitas acceso a nivel de SO, software personalizado, cargas permanentes |

### Casos de uso típicos
- **App serverless**: API Gateway expone un endpoint, Lambda procesa la petición.
- **Procesamiento de archivos**: subida a un bucket S3 dispara Lambda (validar, convertir, mover a otro bucket).
- **Triggers de base de datos**: un stream de DynamoDB dispara Lambda al detectar un cambio.
- **CRON serverless**: EventBridge dispara Lambda en un horario fijo (reportes, correos).
- **Procesamiento en tiempo real**: Kinesis captura datos (ej. IoT) y Lambda los analiza al vuelo.

### Invocaciones de Lambda
- **Síncrona**: respuesta inmediata (CLI, SDK, API Gateway, ALB). Quien invoca es responsable de manejar errores y reintentos.
- **Asíncrona**: quien invoca no espera respuesta (ej. S3 al generar un evento). Lambda reintenta automáticamente 0-2 veces (configurable), si sigue fallando, el evento puede acabar en una **DLQ** (SQS/SNS). La función debe ser **idempotente** (al reintentar debe dar el mismo resultado).
- **Event Source Mapping**: un conector que lee eventos de un stream/cola (Kinesis, SQS, DynamoDB Streams) y se los pasa a Lambda por lotes, usando los permisos del rol de ejecución. Si falla el procesamiento de un lote y los reintentos se agotan, el lote va a una DLQ (cola de errores).

### Seguridad en Lambda
- **Resource Policy** (política de recursos): controla **quién puede invocar** la función — útil para permisos cross-account o para que otro servicio AWS (S3, SNS...) invoque la función. No hace falta si quien invoca es una identidad de la misma cuenta (confianza implícita).
- **Execution Role** (rol de ejecución): un rol IAM que define **qué puede ejecutar** la función en tu nombre (leer de DynamoDB, escribir en S3...).
- Acceso entre cuentas distintas: requiere política de identidad (salida, en la cuenta que invoca) **y** política de recurso (entrada, en la función Lambda) — ninguna cuenta confía en la otra por defecto.
- Ej: S3 no puede asumir roles IAM, así que necesita permiso explícito en la resource policy de Lambda para invocarla, restringido por `Condition.ArnLike.AWS:SourceArn` al bucket concreto.

### Variables de entorno
- Pares clave-valor leídos en tiempo de ejecución, sin tocar el código (ej. `ENV=dev`).
- Se pueden cifrar con KMS para datos sensibles.

### Redes: Lambda y VPC
- Por defecto, una función sin VPC configurada corre en una **VPC administrada por AWS** — no puede acceder a recursos privados (RDS, ElastiCache, ALB/NLB internos).
- **Lambda "pública"** (sin VPC propia): puede salir a internet y acceder a servicios públicos de AWS, pero no a recursos dentro de una VPC privada.
- **Lambda en una VPC** (subred privada): puede acceder a recursos privados, pero no sale a internet salvo que haya un NAT Gateway; puede acceder a servicios públicos de AWS vía VPC Endpoints sin salir a internet.

### Ciclo de vida de una ejecución
- **INIT**: se carga el código y las dependencias.
- **INVOKE**: se ejecuta el handler — si es la primera vez, es un **cold start**.
- **Invocaciones siguientes (warm start)**: si el entorno sigue activo, se reutiliza (más rápido).
- **SHUTDOWN**: tras un tiempo sin invocaciones, AWS libera el entorno.
- Ej: inicializar clientes SDK (ej. `boto3.client('s3')`) **fuera** del handler, así se crean una vez y se reutilizan entre invocaciones warm — evita latencia y saturación de conexiones.

### Lambda Layers y almacenamiento
- **Layers**: paquetes de dependencias compartidas entre funciones, para no duplicarlas en cada paquete de despliegue (hasta 5 capas por función, 250 MB en total).
- **`/tmp`**: almacenamiento efímero local, hasta 10 GB — útil para archivos intermedios, se pierde al destruirse el entorno.
- **EFS**: si la función corre en una VPC, puede montar un sistema de archivos EFS — persistente y compartido entre invocaciones, a diferencia de `/tmp`.

### Versiones y alias
- Trabajar en una función = trabajar en **`$LATEST`** (mutable).
- Publicar una función crea una **versión** inmutable con número creciente y su propio ARN (`...:funcion:1`, `...:funcion:2`...).
- Un **alias** (ej. `prod`, `dev`) es un nombre mutable que apunta a una versión concreta, con su propio ARN — permite desplegar sin cambiar quién invoca.
- Los alias permiten **despliegues canary** (repartir tráfico entre dos versiones, ej. 90%/10%).
- Un alias no puede apuntar a otro alias.

### Concurrencia y throttling
- Concurrencia = nº de ejecuciones simultáneas que soporta la función. Límite por defecto: 1.000 por región/cuenta.
- Al superarlo se produce **throttling**: en invocación síncrona devuelve error 429; en asíncrona se reintenta y puede acabar en DLQ.
- Fórmula para estimarla: `solicitudes/segundo × duración media de cada solicitud (segundos)`.
- **Concurrencia reservada**: garantiza un cupo fijo para una función (el resto de funciones comparte lo que quede).
- **Concurrencia aprovisionada**: mantiene entornos ya inicializados y listos, evitando cold starts (tiene coste adicional).

### Otras integraciones
- **Function URLs**: URL HTTPS directa a una función (`https://<id>.lambda-url.<region>.on.aws/`), sin necesidad de API Gateway ni ALB — ideal para prototipos/webhooks, pero sin throttling avanzado ni WAF nativos.
- **ALB + Lambda**: un ALB puede enrutar peticiones HTTP/HTTPS directamente a una función como target.
- **EventBridge + Lambda**: reglas CRON/rate para tareas programadas, o reglas sobre cambios de estado de otros servicios (ej. CodePipeline).

### Monitorización
- **CloudWatch**: métricas automáticas (invocaciones, errores, duración, ejecuciones concurrentes, errores en destinos asíncronos/DLQ).
- **CloudWatch Logs**: cada función genera un log group `/aws/lambda/<nombre-función>`, siempre que el rol de ejecución tenga permisos de `logs:CreateLogGroup`/`CreateLogStream`/`PutLogEvents`.
- **X-Ray**: traza el recorrido de una petición a través de la función y otros servicios, útil para ver dónde se pierde tiempo.

## Comandos clave

```bash
# Crear una función a partir de un paquete .zip local
aws lambda create-function \
  --function-name <function-name> --runtime <runtime> \
  --role <execution-role-arn> --handler <archivo>.<handler> \
  --zip-file fileb://<ruta-paquete.zip>

# Invocar una función directamente (síncrono)
aws lambda invoke \
  --function-name <function-name> --payload '<json-payload>' --region <region-name> <archivo-salida.json>

# Configurar variables de entorno
aws lambda update-function-configuration \
  --function-name <function-name> \
  --environment "Variables={<CLAVE1>=<valor1>,<CLAVE2>=<valor2>}"

# Dar permiso a un servicio (ej. S3) para invocar la función (resource policy)
aws lambda add-permission \
  --function-name <function-name> --statement-id <statement-id> \
  --action lambda:InvokeFunction --principal s3.amazonaws.com \
  --source-arn <bucket-arn> --source-account <account-id>

# Publicar una versión inmutable a partir de $LATEST
aws lambda publish-version --function-name <function-name>

# Crear un alias que apunte a una versión concreta
aws lambda create-alias \
  --function-name <function-name> --name <alias-name> --function-version <version-number>

# Reservar concurrencia para una función
aws lambda put-function-concurrency \
  --function-name <function-name> --reserved-concurrent-executions <n>

# Conectar una fuente de eventos (ej. una cola SQS) a la función
aws lambda create-event-source-mapping \
  --function-name <function-name> --event-source-arn <source-arn> --batch-size <n>

# Habilitar una URL HTTPS directa para la función
aws lambda create-function-url-config \
  --function-name <function-name> --auth-type NONE

# Borrar una función
aws lambda delete-function --function-name <function-name>
```

## Notas y gotchas

- No confundir **Resource Policy** (quién puede invocar la función, entrada) con **Execution Role** (qué puede hacer la función, salida) — son permisos en direcciones opuestas y se necesitan ambos para invocaciones cross-account.
- Una Lambda dentro de una VPC pierde el acceso a internet por defecto: hace falta NAT Gateway (para salir a internet) o VPC Endpoints (para llegar a servicios AWS sin salir de la red privada). Es fácil asumir que "meterla en la VPC" solo añade acceso, cuando también puede quitarlo.
- `$LATEST` es la única versión mutable — cualquier prueba de código en curso vive ahí; las versiones publicadas ya no cambian nunca, por eso son seguras para producción vía alias.
- Un alias no puede apuntar a otro alias — solo a una versión.
- El throttling se comporta distinto según el tipo de invocación: en síncrona el cliente ve el fallo (429) al momento, en asíncrona el sistema reintenta solo — importante al diseñar qué pasa si se supera el límite de concurrencia.
- Inicializar el cliente SDK fuera del handler es la optimización más repetida en las buenas prácticas del curso — mismo patrón que "reutilizar conexiones" en otros contextos.

## Recursos

- https://docs.aws.amazon.com/lambda/latest/dg/welcome.html
- https://docs.aws.amazon.com/lambda/latest/dg/lambda-invocation.html
- https://docs.aws.amazon.com/lambda/latest/dg/configuration-concurrency.html
- https://docs.aws.amazon.com/lambda/latest/dg/configuration-versions.html
- https://docs.aws.amazon.com/lambda/latest/dg/urls-configuration.html
