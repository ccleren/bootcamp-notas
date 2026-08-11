# Módulo 14 — Amazon S3 Avanzado

## Resumen

### Reglas de ciclo de vida (lifecycle)
- Mueven objetos entre clases de almacenamiento automáticamente, o los borran tras X tiempo.
- Modelo en **cascada**: Standard → Standard-IA → Glacier, según antigüedad.
- Se pueden borrar versiones antiguas de objetos (si hay versionado).
- Se pueden crear reglas por **prefijo** (`s3://bucket/fotos/*`) o por **etiquetas** (`Fase: Test`).
- Patrón típico: datos con acceso poco frecuente → Standard IA → para retención a largo plazo → Glacier .

### Buckets de pago por solicitante (Requester Pays)
- Normalmente el **propietario** del bucket paga almacenamiento + transferencia.
- Con Requester Pays, **quien descarga** paga el coste de red (el propietario sigue pagando el almacenamiento).
- Útil para compartir datasets grandes con otras cuentas sin asumir el coste de descarga.
- ⚠️ Al activarlo, **se pierde el acceso anónimo** al bucket.

### CORS (Cross-Origin Resource Sharing)
- Mecanismo del navegador que permite que una web en un origen pida recursos a otro origen (ej. `mi-bucket-page` pidiendo un vídeo de `mi-bucket-videos`).
- Sin las cabeceras CORS correctas en el bucket destino, el navegador bloquea la petición.
- Se puede permitir un origen específico o todos (`*`).

### MFA Delete
- Exige un código MFA para: **borrar permanentemente una versión de un objeto** o **suspender el versionado**.
- Requiere que el **versionado esté activado**.
- Solo el **root** de la cuenta propietaria puede activar/desactivar MFA Delete (ni siquiera un admin IAM).

### Logs de acceso de S3
- Registra todas las peticiones (autorizadas o denegadas) a un bucket, en **otro bucket** de logs.
- El bucket de logs debe estar en la **misma región**.
- ⚠️ **Nunca uses el mismo bucket como monitorizado y como destino de logs** — se crea un bucle infinito de logs generando logs.

### URLs prefirmadas (presigned URLs)
- Dan acceso temporal a un objeto **sin cambiar la política del bucket**.
- Se generan desde consola, CLI o SDK.
- Caducidad: consola 1 min–12h; CLI/SDK hasta 7 días.
- Casos de uso: descarga temporal de contenido premium, subida temporal a una ruta concreta.

### S3 Glacier Vault Lock
- Modelo **WORM** (Write Once, Read Many): bloquea una política de bóveda para que no se pueda editar ni borrar nunca más.
- Los objetos en esa bóveda Glacier no se pueden eliminar.
- Caso de uso: cumplimiento normativo, retención obligatoria de datos.

### S3 Object Lock
- Bloquea el borrado de una **versión concreta** de un objeto durante un tiempo determinado.
- El permiso `s3:PutObjectLegalHold` permite un bloqueo **indefinido** (sin periodo fijo, hasta que se retira el permiso).

### Puntos de acceso (Access Points) de S3
- Dan acceso granular a un bucket sin tener que meter todo en una única política de bucket gigante.
- Los usuarios pueden acceder a través de un ARN de punto de acceso o un alias de punto de acceso.
- Cada Access Point tiene su propia política + su propio DNS único.
- Se puede restringir por prefijo, etiqueta o acción.
- Pueden configurarse para solo aceptar tráfico desde una **VPC concreta** (vía VPC Endpoint) — refuerza que el acceso solo venga de tu red privada.

### Notificaciones de eventos S3
- Eventos: `ObjectCreated`, `ObjectRemoved`, `ObjectRestore`, `Replication`...
- Destinos: **SNS**, **SQS**, **Lambda**.
- Se puede filtrar por patrón de nombre (ej. `*.jpg`).
- Se pueden crear tantos eventos S3 como quieras.
- Cada destino necesita una política de recursos que permita a `s3.amazonaws.com` invocarlo/publicar, con `Condition` restringiendo el `aws:SourceArn` al bucket concreto (evita que cualquier bucket ajeno dispare tu función).
- **Con EventBridge** (en vez de notificación directa): filtrado avanzado con reglas JSON, más de 18 destinos posibles (Step Functions, Kinesis...), reintentos y entrega fiable.

### Rendimiento de S3
- **Carga multiparte**: recomendada >100MB, **obligatoria** >5GB — divide el archivo y sube las partes en paralelo.
- **Transfer Acceleration**: usa la red global de AWS (edge locations) para acelerar subidas de larga distancia — el archivo entra por el edge más cercano y viaja por la red privada de AWS hasta el bucket.
- Se beneficia de la red de entrega de contenido *(CDN)*, utiliza protocolos seguros y reduce costos.

### S3 Select / Glacier Select
- Permite usar **SQL simple** para filtrar filas/columnas de un objeto y traer solo el subconjunto que necesitas, en vez de descargar el objeto entero.
- Reduce transferencia de red y coste de CPU en el cliente.
- Glacier Select hace lo mismo pero directamente sobre datos archivados en Glacier.

### Cifrado en S3
| Método | Quién gestiona la clave | Notas |
|---|---|---|
| **SSE-S3** | AWS (propiedad de S3) | Activado por defecto, AES-256, cabecera `x-amz-server-side-encryption: AES256` |
| **SSE-KMS** | AWS KMS | Auditable vía CloudTrail, más control granular, cabecera `aws:kms` |
| **SSE-C** | El cliente (clave no se guarda en S3) | Obliga a HTTPS, hay que mandar la clave en cada petición |
| **Cifrado del lado del cliente** | El cliente, completamente | Cifras antes de subir y descifras después de bajar — S3 nunca ve el dato en claro |

- **Cifrado en tránsito**: SSL/TLS. S3 expone endpoint HTTP (sin cifrar) y HTTPS (cifrado) — usa siempre HTTPS. Obligatorio para SSE-C.
- Se puede **forzar HTTPS** con una política de bucket que deniegue `s3:GetObject` si `aws:SecureTransport` es `false`.

### S3 Object Lambda
- Ejecuta una función Lambda que **transforma el objeto al vuelo**, antes de devolverlo a quien lo pidió — sin duplicar el dato.
- Casos de uso: convertir formatos (XML→JSON), extraer columnas de un CSV, ocultar datos sensibles, redimensionar/marcar imágenes.
- Se monta sobre un **Access Point de S3 Object Lambda**, que intercepta la petición y llama a la Lambda.

### Puntos de acceso multi-región
- Un único endpoint que enruta a **varios buckets en distintas regiones**.
- **Redirección inteligente**: manda la petición al bucket geográficamente más cercano.
- **Replicación bidireccional** entre los buckets.
- Soporta failover: modo **Activo-Activo** (ambas regiones sirven) o **Activo-Pasivo** (una es respaldo).

### S3 VPC Endpoints
- Permiten que instancias de una VPC accedan a S3 **sin salir a internet**.
- Alternativa a que una instancia pública use el Internet Gateway para llegar a S3.
- Se puede restringir el acceso al bucket solo desde ese VPC Endpoint (`aws:SourceVpce`) o desde cualquier endpoint de una VPC concreta (`aws:SourceVpc`).
- Ventajas: más seguridad, más control, ideal para subredes privadas sin salida a internet.

## Comandos clave

```bash
# Requester Pays: el que descarga paga la transferencia
aws s3api put-bucket-request-payment \
  --bucket <bucket-name> --request-payment-configuration Payer=Requester

# CORS
aws s3api put-bucket-cors \
  --bucket <bucket-name> --cors-configuration file://cors-config.json

# Activar MFA Delete (requiere versionado + el serial y código del dispositivo MFA)
aws s3api put-bucket-versioning \
  --bucket <bucket-name> \
  --versioning-configuration Status=Enabled,MFADelete=Enabled \
  --mfa "<mfa-device-serial> <mfa-code>"

# Logs de acceso hacia otro bucket
aws s3api put-bucket-logging \
  --bucket <bucket-name> --bucket-logging-status file://logging-config.json

# Generar una URL prefirmada (caduca en 1h)
aws s3 presign s3://<bucket-name>/<key> --expires-in 3600

# Object Lock (WORM a nivel de objeto)
aws s3api put-object-lock-configuration \
  --bucket <bucket-name> --object-lock-configuration file://object-lock-config.json

# Crear un Access Point
aws s3control create-access-point \
  --account-id <account-id> --name <access-point-name> --bucket <bucket-name>

# Notificaciones de eventos (a SNS/SQS/Lambda)
aws s3api put-bucket-notification-configuration \
  --bucket <bucket-name> --notification-configuration file://notification-config.json

# Cifrado por defecto del bucket (SSE-S3 o SSE-KMS)
aws s3api put-bucket-encryption \
  --bucket <bucket-name> --server-side-encryption-configuration file://encryption-config.json

# Crear un VPC Endpoint de tipo Gateway para S3 (acceso privado sin salir a internet)
aws ec2 create-vpc-endpoint \
  --vpc-id <vpc-id> --service-name com.amazonaws.<region>.s3 \
  --route-table-ids <route-table-id>
```

## Notas y gotchas

- El bucle de logs (bucket de logs = bucket monitorizado) es un error clásico que hace crecer el bucket exponencialmente — revisa siempre que sean buckets distintos.
- Requester Pays desactiva el acceso anónimo automáticamente — no es opcional, es una consecuencia directa de activarlo.
- Solo el usuario **root** puede tocar MFA Delete, ni un admin de IAM con permisos totales puede hacerlo — es una protección deliberadamente fuera del alcance de IAM.
- Para exámenes de certificación: hay que saber distinguir SSE-S3 / SSE-KMS / SSE-C / cifrado de cliente de memoria — hay preguntas típicas sobre "quién gestiona la clave" en cada uno.
- Object Lock (a nivel de objeto/versión) y Glacier Vault Lock (a nivel de bóveda completa) resuelven un problema parecido (WORM) pero a escalas distintas — no confundirlos.

## Recursos

- https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lifecycle-mgmt.html
- https://docs.aws.amazon.com/AmazonS3/latest/userguide/ShareObjectPreSignedURL.html
- https://docs.aws.amazon.com/AmazonS3/latest/userguide/UsingKMSEncryption.html
- https://docs.aws.amazon.com/AmazonS3/latest/userguide/access-points.html
- https://docs.aws.amazon.com/AmazonS3/latest/userguide/NotificationHowTo.html
