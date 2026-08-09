# Módulo 13 — Amazon S3 (Simple Storage Service)

## Resumen

- S3 = almacenamiento de objetos, uno de los servicios base de AWS. Capacidad prácticamente infinita, cualquier tipo de dato (vídeo, audio, fotos, texto, datasets grandes).
- **Cada bucket se crea en una región concreta**.
- Dos componentes: **Buckets** (contenedores) y **Objetos** (los archivos en sí).

### Buckets
- Nombre **único a nivel global** — entre todas las cuentas y regiones de AWS, no solo la tuya.
- Viven en una región concreta aunque S3 se sienta "global".

### Objetos
- Cada objeto tiene: **key** (identificador), **ID de versión**, **valor** (los datos), **metadatos**, **subrecursos**, información de control de acceso.
- Tamaño: de 0 bytes a **5 TB**. Para subir más de 5GB de una vez, hace falta **carga multiparte**.
- La **key es la ruta completa** (ej. s3://mi-bucket/`mi-carpeta/archivo.txt`) — **no existen carpetas de verdad** dentro de un bucket, no existe el concepto de directorios.

### Casos de uso típicos
Backup y almacenamiento, recuperación ante desastres, datos de IoT, almacenamiento híbrido, hosting de apps/medios, big data analytics, entrega de software, **sitios web estáticos**.

### Políticas de bucket
Documento JSON (igual que las políticas IAM) con: `Resource` (bucket/objeto/access point afectado), `Effect` (Allow/Deny), `Actions`, `Principal` (quién), `Condition` (opcional).

### Bloqueo de acceso público
- **Activado por defecto** en buckets, access points y objetos nuevos — para evitar filtraciones accidentales.
- Se puede desactivar explícitamente vía política de bucket / de access point / permisos de objeto (hay que hacerlo a propósito).

### Patrones de acceso a S3
| Patrón | Cómo se controla |
|---|---|
| Acceso público a un bucket | Política de bucket que permite el acceso público |
| EC2 → S3 | Rol IAM en la instancia |
| Usuario IAM → S3 | Política IAM adjunta al usuario |
| Entre cuentas AWS | Política de bucket que permite la otra cuenta |

### Sitios web estáticos
- S3 puede servir un sitio web estático directamente por HTTP.
- La URL varía según la región del bucket.
- ⚠️ Error 403 al acceder → la política de bucket no permite lecturas públicas (causa nº1 de este error).

### Versionado
- Se activa **a nivel de bucket**, no por archivo.
- Protege contra borrados accidentales — puedes volver a una versión anterior.
- Los archivos que ya existían antes de activar versionado obtienen la versión **"null"**.
- **Suspender** el versionado no borra las versiones ya creadas.

### Replicación entre buckets
- Requiere **versionado activado en origen y destino** — sin eso no funciona.
- Puede ser entre cuentas de AWS distintas. La copia es **asíncrona**.
- Solo replica objetos **nuevos** desde que se activa (no hace backfill de lo que ya había).
- Dos tipos:
  - **CRR** (entre regiones): cumplimiento normativo, menor latencia de acceso, entre cuentas.
  - **SRR** (misma región): agregación de logs, replicar entre cuenta de producción y de test.
- **No se encadena**: si el bucket 1 replica al 2, y el 2 replica al 3, lo que se crea en el 1 **no** llega al 3.

### Clases de almacenamiento
| Clase | Para qué | Precio aprox. (N. Virginia) |
|---|---|---|
| **S3 Standard** | Datos activos, acceso frecuente, milisegundos, ≥3 AZ | $0.023/GB |
| **S3 Express One Zone** | Acceso ultrarrápido, 1 sola AZ, cargas intensivas | $0.16/GB |
| **S3 Standard-IA** | Acceso infrecuente pero rápido cuando se necesita, ≥3 AZ, mín. 30 días | $0.0125/GB |
| **S3 One Zone-IA** | Como IA pero 1 sola AZ, datos recreables, mín. 30 días | $0.01/GB |
| **S3 Glacier Instant Retrieval** | Archivo frío, recuperación en milisegundos, mín. 90 días | $0.004/GB |
| **S3 Glacier Flexible Retrieval** | Recuperación asíncrona (minutos-horas), mín. 90 días | $0.0036/GB |
| **S3 Glacier Deep Archive** | Archivo a largo plazo, mín. 180 días | $0.00099/GB |
| **S3 Intelligent-Tiering** | Mueve automáticamente entre niveles según el patrón de acceso real | $0.0025/1000 objetos + coste del nivel usado |

- Todas las clases IA/Glacier cobran **tarifa de recuperación por GB** además del almacenamiento.
- Se puede cambiar de clase manualmente o con **reglas de ciclo de vida** (lifecycle rules).
- **Intelligent-Tiering**: sin coste de recuperación, pequeño cargo de monitorización mensual, ideal cuando el patrón de acceso es impredecible. Objetos menores de 128KB no son elegibles (se facturan siempre al nivel frecuente).

## Comandos clave

```bash
# Crear un bucket (fuera de us-east-1 hace falta indicar la región del bucket)
aws s3api create-bucket \
  --bucket <bucket-name> --region <region> \
  --create-bucket-configuration LocationConstraint=<region>

# Subir/bajar archivos y sincronizar carpetas
aws s3 cp <archivo-local> s3://<bucket-name>/<key>
aws s3 sync <carpeta-local> s3://<bucket-name>/<prefijo>

# Activar versionado
aws s3api put-bucket-versioning \
  --bucket <bucket-name> --versioning-configuration Status=Enabled

# Bloquear/permitir acceso público explícitamente
aws s3api put-public-access-block \
  --bucket <bucket-name> \
  --public-access-block-configuration "BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true"

# Aplicar una política de bucket (JSON local)
aws s3api put-bucket-policy --bucket <bucket-name> --policy file://bucket-policy.json

# Activar hosting de sitio web estático
aws s3api put-bucket-website \
  --bucket <bucket-name> --website-configuration file://website-config.json

# Configurar replicación entre buckets (origen y destino ya versionados)
aws s3api put-bucket-replication \
  --bucket <bucket-name> --replication-configuration file://replication-config.json

# Reglas de ciclo de vida (ej. mover a IA a los 30 días, a Glacier a los 90)
aws s3api put-bucket-lifecycle-configuration \
  --bucket <bucket-name> --lifecycle-configuration file://lifecycle-config.json
```

## Notas y gotchas

- Un nombre de bucket es único **en todo AWS**, no solo en tu cuenta — si el nombre "obvio" que quieres ya existe (de otra cuenta), hay que buscar otro.
- No hay carpetas reales en S3: son solo claves con `/` — importante para no asumir operaciones tipo "renombrar carpeta" (en realidad hay que copiar+borrar cada objeto).
- El error 403 sirviendo un sitio web estático casi siempre es la política de bucket, no la configuración de hosting en sí.
- La replicación no rellena hacia atrás (solo objetos nuevos) ni se encadena entre más de un salto — hay que tenerlo en cuenta al diseñar una arquitectura multi-bucket.

## Recursos

- https://docs.aws.amazon.com/AmazonS3/latest/userguide/Welcome.html
- https://docs.aws.amazon.com/AmazonS3/latest/userguide/storage-class-intro.html
- https://docs.aws.amazon.com/AmazonS3/latest/userguide/Versioning.html
- https://docs.aws.amazon.com/AmazonS3/latest/userguide/replication.html
- https://docs.aws.amazon.com/AmazonS3/latest/userguide/access-control-block-public-access.html
