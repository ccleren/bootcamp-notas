# Módulo 32 — Analíticas en AWS

## Resumen

Este módulo recorre el ecosistema de servicios de analítica de AWS: almacén de datos (Redshift), preparación de datos (Glue), streaming (Kinesis, MSK), consultas ad-hoc (Athena), big data (EMR), visualización (QuickSight), integración SaaS (AppFlow), orquestación (MWAA) y búsqueda (OpenSearch).

### Amazon Redshift
- Basado en PostgreSQL, pero pensado para **OLAP** (análisis), no OLTP — carga datos por lotes (ej. cada hora), no transacción a transacción.
- Almacenamiento **por columnas**, ejecución de consultas en paralelo masivo (MPP), escala a PBs, interfaz SQL.
- Se integra con herramientas de BI (QuickSight, Tableau...).
- **Consulta federada**: puede consultar datos en vivo en Aurora/RDS sin moverlos. **Consulta a S3** (Redshift Spectrum): consulta directamente un data lake en S3.
- Casos de uso: modernizar el almacén de datos, análisis de impresiones/clics publicitarios, ventas globales, tendencias sociales, datos históricos de bolsa.

### AWS Glue
- Servicio **serverless** de integración/ETL de datos: prepara y carga datos para análisis.
- Compatible con Redshift, RDS, DynamoDB, S3 y la mayoría de BBDD SQL; trabajos activados por eventos, programación o bajo demanda.
- Casos de uso: consultar un data lake en S3 sin moverlo, transformar logs, pipelines ETL disparados por eventos (Lambda ejecuta un job de Glue), catálogo unificado de metadatos (**Glue Data Catalog**).
- **Glue Studio**: editor visual de flujos ETL (DAGs), fuentes S3/Kinesis/Kafka/JDBC, destino a S3 o al Data Catalog.
- **Glue DataBrew**: limpieza/preparación de datos con interfaz visual (+250 transformaciones), recetas reutilizables, reglas de calidad de datos.
  - Manejo de **PII**: sustitución, mezcla, cifrado determinístico/probabilístico, descifrado, anulación, enmascaramiento, hashing — todo aplicable como transformación dentro de un job de DataBrew.

### Amazon Kinesis (familia)
| Servicio | Para qué |
|---|---|
| **Kinesis Data Streams** | Captura, procesa y almacena streams de datos (tú gestionas productores/consumidores) |
| **Managed Apache Flink** (antes Kinesis Data Analytics) | Analiza streams con SQL o Flink |
| **Amazon Data Firehose** | Carga streams en destinos (S3, Redshift, OpenSearch, terceros) de forma gestionada |
| **Kinesis Video Streams** | Captura y procesa streams de vídeo |

#### Kinesis Data Streams
- Datos organizados en **fragmentos (shards)**; retención 1-365 días, con capacidad de reproducir (replay) — inmutable una vez insertado.
- Los registros con la misma clave de partición van al mismo fragmento (garantiza orden dentro de esa clave).
- **Modo aprovisionado**: eliges nº de shards; cada shard = 1 MB/s (1000 registros/s) de entrada, 2 MB/s de salida; pagas por shard-hora.
- **Modo bajo demanda**: sin gestión de capacidad, 4 MB/s / 4000 registros/s por defecto, escala solo según el pico de los últimos 30 días.
- **Registro**: nº de secuencia + clave de partición + hasta 1 MB de datos. Usa `PutRecords` (batch) para eficiencia.
- `ReadProvisionedThroughputExceeded`: throttling por exceder capacidad — mitigar con clave de partición bien distribuida, backoff exponencial, o más shards.
- **Dividir shards** (split): para un shard "caliente"; aumenta capacidad/coste; máx. 2 shards por operación.
- **Fusionar shards** (merge): para shards "fríos" con poco tráfico; reduce capacidad/coste; máx. 2 shards por operación.
- Consumidores: fan-out clásico (2 MB/s compartidos entre todos los consumidores de un shard) vs. fan-out mejorado (2 MB/s por consumidor y shard, vía `SubscribeToShard`).
- Con Lambda: lee por lotes configurables, reintenta hasta éxito o caducidad, hasta 10 lotes en paralelo por shard.

#### Amazon Data Firehose
- Totalmente gestionado, sin servidor, escalado automático — carga streams "casi en tiempo real" en S3/Redshift/OpenSearch/terceros/HTTP personalizado.
- Buffer configurable: 0-900s de intervalo, mínimo 1 MB de tamaño. Soporta transformación con Lambda, y puede mandar datos fallidos (o todos) a un bucket S3 de respaldo.

#### Kinesis Data Streams vs. Data Firehose
| | Data Streams | Data Firehose |
|---|---|---|
| Gestión | Escribes productor/consumidor propio | Totalmente gestionado |
| Latencia | Tiempo real (~200ms) | Casi en tiempo real |
| Almacenamiento | 1-365 días, con replay | No almacena |
| Escalado | Manual (split/merge) | Automático |

#### Managed Apache Flink (aplicaciones SQL)
- Analiza en tiempo real datos de Kinesis Data Streams/Firehose con SQL, pudiendo enriquecerlos con datos de referencia en S3.
- Sin servidor que aprovisionar, escalado automático, pago por consumo real.
- Salida: puede crear un nuevo stream de Kinesis, o enviar resultados vía Firehose a otro destino.
- Casos de uso: análisis de series temporales, dashboards y métricas en tiempo real.

### Amazon Athena
- Consultas SQL **interactivas y serverless** directamente sobre datos en S3 — sin cargarlos, sin servidores que gestionar.
- Modelo "esquema al leer": las tablas se definen de antemano en un catálogo, y los datos se proyectan a esa estructura al consultarlos — el dato original en S3 nunca se modifica.
- Pago por datos escaneados — usar formatos columnares (ORC, Parquet) ahorra mucho coste frente a CSV/JSON.
- Soporta formatos estructurados, semiestructurados y no estructurados; lee directamente logs de CloudTrail, ELB, VPC Flow Logs.
- Casos de uso: análisis de logs, ETL ligero, consultas sobre data lakes.

### Amazon EMR
- EMR = Elastic MapReduce: clústeres gestionados de **Hadoop** (y Spark, HBase, Presto, Flink) para Big Data, con cientos de instancias EC2 si hace falta.
- AWS gestiona aprovisionamiento y configuración; soporta auto scaling e instancias Spot.
- Casos de uso: procesamiento masivo de datos, ML, indexación web.

### Amazon QuickSight
- Servicio de BI serverless: dashboards y visualizaciones interactivas, análisis ad-hoc, alertas de anomalías.
- Fuentes nativas: Redshift, Aurora, RDS, Athena, OpenSearch, S3, IoT Analytics, y cualquier fuente JDBC/ODBC (incluye SaaS como Salesforce).
- Casos de uso: KPIs de ventas/marketing, análisis financiero, RRHH, logística, monitoreo de TI/seguridad.

### Amazon AppFlow
- Transferencia automática y segura de datos entre servicios AWS y apps SaaS, sin desarrollar integraciones a medida.
- Incluye transformación de datos en el propio flujo (limpiar/filtrar/enriquecer), cifrado en tránsito, cumplimiento (HIPAA, GDPR).
- Frecuencia: programada, por evento o bajo demanda.

### Amazon MWAA (Managed Workflows for Apache Airflow)
- Airflow gestionado: crea, programa y monitoriza flujos de trabajo complejos definidos como código Python (DAGs).
- AWS gestiona la infraestructura subyacente.
- Casos de uso: coordinar pipelines ETL, preparar datos para ML.

### Amazon MSK (Managed Streaming for Apache Kafka)
- Kafka gestionado: aprovisiona, configura y mantiene el clúster; apps/herramientas Kafka existentes funcionan sin cambios de código.
- Alta disponibilidad: reemplaza nodos no sanos automáticamente, replica datos entre brokers en varias AZ.
- Configuración: nº de AZ (recomendado 3), VPC/subredes, tipo de instancia del broker, nº de brokers por AZ, tamaño de volúmenes EBS (1GB-16TB).
- **Seguridad**: cifrado opcional en tránsito (TLS broker-broker y cliente-broker), en reposo (KMS sobre EBS); autenticación/autorización vía mTLS+ACLs, SASL/SCRAM+ACLs, o IAM.
- **Monitoreo**: CloudWatch Metrics o Prometheus (JMX Exporter, Node Exporter), niveles básico/mejorado/por-tema, entrega de logs de broker a S3/Kinesis/CloudWatch.

### Amazon OpenSearch
- Fork de Elasticsearch (iniciado por AWS), gestionado, para búsqueda/visualización/análisis en tiempo real de grandes volúmenes de datos.
- Incluye **OpenSearch Dashboards** para visualización.
- Casos de uso: búsqueda de texto rápida, monitoreo de aplicaciones, SIEM (gestión de seguridad e incidentes).

## Comandos clave

```bash
# Crear un clúster Redshift de un solo nodo
aws redshift create-cluster \
  --cluster-identifier <cluster-id> --node-type <tipo-nodo> \
  --master-username <usuario> --master-user-password <password> --cluster-type single-node

# Crear un stream de Kinesis Data Streams
aws kinesis create-stream --stream-name <stream-name> --shard-count <n>

# Publicar un registro en el stream
aws kinesis put-record --stream-name <stream-name> --data "<datos>" --partition-key <clave-particion>

# Crear un delivery stream de Firehose hacia S3
aws firehose create-delivery-stream \
  --delivery-stream-name <delivery-stream-name> \
  --s3-destination-configuration RoleARN=<role-arn>,BucketARN=<bucket-arn>

# Lanzar una consulta en Athena
aws athena start-query-execution \
  --query-string "<consulta-sql>" \
  --result-configuration OutputLocation=s3://<bucket-name>/<prefijo>/

# Crear un clúster EMR con Spark
aws emr create-cluster \
  --name "<cluster-name>" --release-label <version-emr> \
  --applications Name=Spark --instance-type <tipo-instancia> --instance-count <n> --use-default-roles

# Lanzar la ejecución de un job de Glue
aws glue start-job-run --job-name <job-name>

# Crear un clúster MSK (a partir de un JSON con la config de brokers)
aws kafka create-cluster \
  --cluster-name <cluster-name> --broker-node-group-info file://<ruta-broker-info.json> \
  --kafka-version "<version>" --number-of-broker-nodes <n>
```

## Notas y gotchas

- Redshift es OLAP, no OLTP — es un error típico intentar usarlo como sustituto de RDS/Aurora para tráfico transaccional; está pensado para cargas por lotes y consultas analíticas masivas, no para escrituras frecuentes de pocas filas.
- La diferencia clave entre Kinesis Data Streams y Data Firehose no es "cuál es mejor" sino "¿necesito control fino y replay, o solo cargar datos en un destino sin gestionar nada?" — Streams da control, Firehose da simplicidad.
- Athena cobra por **datos escaneados**, no por tiempo de cómputo — pasar de CSV/JSON a Parquet/ORC columnar puede reducir el coste drásticamente en la misma consulta.
- Glue DataBrew con las técnicas de PII no sustituye una estrategia de gobierno de datos — son herramientas para aplicarla, pero hay que decidir explícitamente qué campos son PII y qué técnica usar en cada uno.
- MSK da acceso a Kafka "de verdad" (compatible con herramientas existentes) a cambio de gestionar tú mismo tópicos, particiones y ACLs — es la opción cuando ya hay inversión en el ecosistema Kafka, no un sustituto directo de Kinesis para todo.

## Recursos

- https://docs.aws.amazon.com/redshift/latest/mgmt/welcome.html
- https://docs.aws.amazon.com/glue/latest/dg/what-is-glue.html
- https://docs.aws.amazon.com/streams/latest/dev/introduction.html
- https://docs.aws.amazon.com/firehose/latest/dev/what-is-this-service.html
- https://docs.aws.amazon.com/athena/latest/ug/what-is.html
- https://docs.aws.amazon.com/emr/latest/ManagementGuide/emr-what-is-emr.html
- https://docs.aws.amazon.com/msk/latest/developerguide/what-is-msk.html
- https://docs.aws.amazon.com/opensearch-service/latest/developerguide/what-is.html
