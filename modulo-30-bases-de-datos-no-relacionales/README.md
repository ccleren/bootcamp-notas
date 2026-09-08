# Módulo 30 — Bases de datos no relacionales (DynamoDB, ElastiCache, MemoryDB)

## Resumen

### NoSQL vs. SQL, en la práctica
- Las apps NoSQL (MongoDB, DynamoDB, Cassandra, Redis...) no soportan `JOIN` ni agregaciones tipo `SUM`/`AVG` — todos los datos que necesita una consulta suelen estar en una misma fila/elemento.
- Escalan horizontalmente por diseño. No hay "mejor o peor" frente a SQL — hay que modelar los datos y pensar las consultas de forma distinta desde el principio.

## Amazon DynamoDB

### Visión general
- NoSQL totalmente gestionada, alta disponibilidad con replicación Multi-AZ, escala a millones de solicitudes/segundo y cientos de TB.
- Integrada con IAM para seguridad. Clases de tabla: acceso estándar y acceso poco frecuente (IA).

### Tablas globales
- Replicación **activa-activa** entre regiones: lectura y escritura en cualquier región donde exista la tabla global, con baja latencia local.

### Conceptos básicos
- Una tabla tiene elementos (filas), cada uno con atributos (pueden ser nulos). Tamaño máximo de un elemento: **400 KB**.
- Toda tabla necesita una **clave primaria**, definida al crearla, que garantiza la unicidad de cada elemento.

### Claves primarias
- **Clave de partición (HASH)**: única por elemento, debe ser "diversa" (alta cardinalidad) para que los datos se repartan bien entre particiones. Ej.: `movie_id` es mejor candidato que `movie_language` (poca variedad, sesgado).
- **Clave de partición + clave de ordenación (HASH+RANGE)**: la combinación debe ser única; los elementos con la misma clave de partición se agrupan y ordenan por la clave de rango.

### Modos de capacidad
| Modo | Cómo funciona | Cuándo usarlo |
|---|---|---|
| **Aprovisionado** (por defecto) | Defines RCU/WCU por adelantado; admite auto scaling de esas unidades; capacidad de ráfaga (burst) para picos puntuales | Carga predecible |
| **Bajo demanda** | Escala solo, sin planificación; ~2,5x más caro | Carga desconocida/impredecible, picos bruscos |

### Unidades de capacidad — fórmulas (importantes)
- **WCU**: 1 WCU = 1 escritura/segundo de un elemento de hasta 1 KB (redondeando el tamaño al KB superior).
  - Ej.: 20 elementos/s de 3 KB → `20 × (3/1) = 60 WCU`.
  - Ej.: 6 elementos/s de 6,5 KB (redondea a 7 KB) → `6 × (7/1) = 42 WCU`.
- **RCU**: 1 RCU = 1 lectura fuertemente consistente/segundo, o 2 eventualmente consistentes/segundo, de un elemento de hasta 4 KB (redondeando al múltiplo de 4 KB superior).
  - Ej.: 16 lecturas eventualmente consistentes/s de 12 KB → `(16/2) × (12/4) = 24 RCU`.
  - Ej.: 10 lecturas fuertemente consistentes/s de 6 KB (redondea a 8 KB) → `10 × (8/4) = 20 RCU`.
- **Lectura eventualmente consistente** (por defecto): tras escribir, una lectura inmediata puede devolver un dato desactualizado.
- **Lectura fuertemente consistente**: `ConsistentRead=True` en `GetItem`/`Query`/`Scan`; consume el doble de RCU.

### Particiones
- DynamoDB aplica una función hash sobre la clave de partición para decidir en qué partición va cada elemento — no hay orden garantizado entre particiones.
- `#particiones = ceil(max(RCUtotal/3000 + WCUtotal/1000, TamañoTotal/10GB))`.

### Throttling
- `ProvisionedThroughputExceededException` al superar RCU/WCU aprovisionadas — causas típicas: claves "calientes" (elemento muy popular), particiones calientes, elementos muy grandes.
- Mitigación: retroceso exponencial (backoff, disponible en el SDK), distribuir mejor las claves de partición, o usar **DAX** si el cuello de botella es de lectura.

### Operaciones básicas
| Operación | Qué hace |
|---|---|
| `PutItem` | Crea o sustituye un elemento (misma clave primaria) |
| `UpdateItem` | Modifica atributos o crea si no existe; útil para contadores atómicos |
| `GetItem` | Lectura por clave primaria; eventualmente consistente por defecto |
| `Query` | Filtra por `KeyConditionExpression` (partición obligatoria con `=`, ordenación opcional con más operadores) + `FilterExpression` sobre atributos no clave |
| `Scan` | Recorre toda la tabla y filtra después — ineficiente; se puede paralelizar con varios workers |
| `DeleteItem` / `DeleteTable` | Borra un elemento (con condición opcional) o la tabla completa |

### Operaciones por lotes
- `BatchWriteItem`: hasta 25 `PutItem`/`DeleteItem` por llamada, hasta 16 MB, no permite `UpdateItem`; los fallos vuelven en `UnprocessedItems`.
- `BatchGetItem`: hasta 100 elementos / 16 MB, lecturas en paralelo; los fallos vuelven en `UnprocessedKeys`.

### PartiQL
- Sintaxis tipo SQL para consultar DynamoDB (datos estructurados, semiestructurados o anidados) desde consola, NoSQL Workbench, API, CLI o SDK.

### Escrituras condicionales
- Solo proceden si se cumple una condición sobre los atributos (`attribute_exists`, `attribute_not_exists`, `begins_with`, `contains`, comparaciones...).
- Distinto de `FilterExpression` (que filtra resultados de **lectura**): las expresiones de condición son para **escritura**.

### Índices secundarios
| Tipo | Clave de partición | Ámbito de la consulta | Límite |
|---|---|---|---|
| **LSI** (local) | Misma que la tabla base | Una partición de la tabla base | Hasta 5 por tabla; se define al crear la tabla; usa las RCU/WCU de la tabla |
| **GSI** (global) | Puede ser distinta | Toda la tabla | Necesita su propia capacidad RCU/WCU |

- ⚠️ Si se estrangula un GSI por escrituras, **también se estrangula la tabla principal**, aunque la tabla tenga WCU de sobra — elegir bien la clave de partición del GSI es crítico.

### DynamoDB Accelerator (DAX)
- Caché en memoria totalmente gestionada, mejora la latencia de milisegundos a microsegundos (~10x).
- Solo funciona con DynamoDB (a diferencia de ElastiCache, que sirve para otras bases de datos también).

### Bloqueo optimista
- Cada elemento lleva un atributo de versión; una actualización solo procede si la versión coincide con la leída — evita que dos clientes se pisen las escrituras entre sí sin necesidad de bloqueos reales.

### DynamoDB Streams
- Captura en orden cronológico los cambios (insert/update/delete) de una tabla, disponibles 24h.
- Casos de uso: reaccionar en tiempo real (email de bienvenida), sincronizar tablas entre regiones, disparar Lambda, auditoría.
- Con Lambda: se define un **event source mapping**; Lambda se invoca **síncronamente** por cada lote de cambios, necesita permisos (`dynamodb:DescribeStream`, `GetRecords`, `GetShardIterator`, `ListStreams`).

### TTL (Time To Live)
- Borra elementos automáticamente pasada una fecha (atributo tipo `Number`, timestamp Unix Epoch). Sin coste de WCU.
- El borrado real ocurre hasta 48h después de expirar — mientras tanto, el elemento expirado sigue apareciendo en lecturas/consultas si no se filtra explícitamente.

### Transacciones
- Operaciones "todo o nada" con propiedades ACID sobre varios elementos/tablas.
- Consumen **el doble** de RCU/WCU (DynamoDB hace 2 operaciones por elemento: preparar y confirmar).
- Ej.: 5 escrituras transaccionales/s de 4 KB → `5 × (4/1) × 2 = 40 WCU`.
- Ej.: 10 lecturas transaccionales/s de 9 KB (redondea a 12 KB) → `10 × (12/4) × 2 = 60 RCU`.

### DynamoDB vs. otros almacenamientos
| Comparación | Diferencia clave |
|---|---|
| vs. **ElastiCache** | ElastiCache es en memoria; DynamoDB es serverless. Ambos son clave-valor |
| vs. **EFS** | EFS necesita montarse como unidad de red en instancias EC2 |
| vs. **EBS/Instance Store** | Solo sirven como caché local, no compartida |
| vs. **S3** | S3 tiene más latencia y no está pensado para objetos pequeños |

### Patrones de diseño frecuentes
- **Fragmentación de escritura (write sharding)**: si una clave de partición tiene pocos valores posibles (ej. `Candidate_ID` con solo 2 candidatos), se generan particiones calientes — se mitiga añadiendo un sufijo a la clave (`Candidate_A-98`) para repartir mejor.
- **Objetos grandes**: el archivo pesado va a S3, y DynamoDB solo guarda los metadatos (incluida la URL del objeto) — nunca el binario en sí.
- **Indexación de metadatos S3**: un trigger Lambda sobre eventos de S3 mantiene sincronizados los metadatos en DynamoDB automáticamente.

### Seguridad y resiliencia
- Endpoints VPC (sin salir a internet), control de acceso granular vía IAM (incluso a nivel de atributo).
- Cifrado en reposo (KMS) y en tránsito (TLS).
- Backup/restauración y **PITR** (recuperación punto-en-el-tiempo), igual que RDS.
- Autenticación de usuarios finales vía proveedores externos (Cognito, Google, Facebook, SAML...) que obtienen un rol IAM temporal para operar sobre DynamoDB.

## Amazon ElastiCache

### Visión general
- Lo que RDS es para bases relacionales gestionadas, ElastiCache lo es para **Redis o Memcached** gestionados.
- Reduce carga de lectura sobre la BD principal, ayuda a que la app no tenga estado propio, reduce datos transferidos entre cliente y servidor.
- ⚠️ Usar ElastiCache normalmente implica cambios en el código de la aplicación (no es transparente).

### Patrones de uso
- **Caché de base de datos**: la app consulta ElastiCache primero; si falla (cache miss), lee de RDS y escribe el resultado en la caché.
- **Almacén de sesiones**: cualquier instancia de la app puede leer la sesión de un usuario desde ElastiCache, sin depender de a qué instancia concreta se conectó antes.

### Redis vs. Memcached
| | Redis | Memcached |
|---|---|---|
| Alta disponibilidad | Sí (Multi-AZ, failover automático) | No |
| Réplicas de lectura | Sí | No (solo sharding entre nodos) |
| Persistencia/backup | Sí | No |
| Arquitectura | — | Multihilo |

### Estrategias de caché
| Estrategia | Cómo funciona | Ventaja | Desventaja |
|---|---|---|---|
| **Lazy loading** | Solo se cachea lo que se pide, al pedirse | La caché no se llena de datos sin usar; un fallo de nodo no es fatal | Penalización del primer acceso (3 saltos); datos pueden quedar obsoletos |
| **Write-through** | Cada escritura en BD también escribe en caché | La caché nunca está desactualizada | Penalización en cada escritura; datos ausentes hasta la primera escritura (mitigar combinando con lazy loading); puede cachear datos que nunca se leen |

### Invalidación y TTL
- Tres formas de invalidar: borrado explícito, expulsión por memoria llena (LRU), o TTL.
- Si hay demasiadas invalidaciones por falta de memoria, hay que escalar la caché.

### Amazon MemoryDB (para Redis)
- Base de datos en memoria **duradera**, compatible con Redis — más de 160M de solicitudes/segundo.
- Registro transaccional Multi-AZ para durabilidad y recuperación rápida (no es solo caché, puede ser la BD principal).
- Casos de uso: apps de microservicios, juegos online, streaming de medios.

## Comandos clave

```bash
# Crear una tabla DynamoDB en modo bajo demanda
aws dynamodb create-table \
  --table-name <table-name> \
  --attribute-definitions AttributeName=<clave-particion>,AttributeType=S \
  --key-schema AttributeName=<clave-particion>,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST

# Insertar/reemplazar un elemento
aws dynamodb put-item \
  --table-name <table-name> --item '{"<clave-particion>":{"S":"<valor>"},"<atributo>":{"S":"<valor>"}}'

# Leer un elemento por clave primaria
aws dynamodb get-item --table-name <table-name> --key '{"<clave-particion>":{"S":"<valor>"}}'

# Consultar por clave de partición
aws dynamodb query \
  --table-name <table-name> \
  --key-condition-expression "<clave-particion> = :valor" \
  --expression-attribute-values '{":valor":{"S":"<valor>"}}'

# Escanear la tabla completa (usar con cuidado)
aws dynamodb scan --table-name <table-name>

# Actualizar atributos de un elemento
aws dynamodb update-item \
  --table-name <table-name> --key '{"<clave-particion>":{"S":"<valor>"}}' \
  --update-expression "SET <atributo> = :valor" \
  --expression-attribute-values '{":valor":{"S":"<nuevo-valor>"}}'

# Borrar un elemento
aws dynamodb delete-item --table-name <table-name> --key '{"<clave-particion>":{"S":"<valor>"}}'

# Habilitar TTL sobre un atributo
aws dynamodb update-time-to-live \
  --table-name <table-name> \
  --time-to-live-specification "Enabled=true,AttributeName=<atributo-ttl>"

# Crear un clúster Redis replicado en ElastiCache
aws elasticache create-replication-group \
  --replication-group-id <replication-group-id> \
  --replication-group-description "<descripcion>" \
  --engine redis --cache-node-type <tipo-nodo> --num-cache-clusters <n>
```

### `scan` desde la CLI: proyección, filtro y paginación

```bash
# Traer solo determinados atributos (projection-expression)
aws dynamodb scan --table-name <table-name> --projection-expression "<atributo1>, <atributo2>"

# Filtrar los resultados tras el escaneo (filter-expression)
aws dynamodb scan --table-name <table-name> \
  --filter-expression "<atributo> = :valor" \
  --expression-attribute-values '{ ":valor": {"S":"<valor>"}}'

# Sin --page-size: una sola llamada a la API si caben todos los elementos
aws dynamodb scan --table-name <table-name>

# Con --page-size 1: una llamada a la API por elemento (paginación más granular)
aws dynamodb scan --table-name <table-name> --page-size 1

# Limitar cuántos elementos se muestran en esta ejecución (max-items)
aws dynamodb scan --table-name <table-name> --max-items 1

# Continuar la paginación con el NextToken devuelto por la llamada anterior
aws dynamodb scan --table-name <table-name> --max-items 1 --starting-token <next-token>
```
- `--page-size` controla cuántos elementos pide la CLI a la API **por llamada** (afecta al nº de llamadas, no a lo que ves en pantalla); `--max-items` controla cuántos elementos muestra la CLI en total antes de parar y devolver un `NextToken`.
- `--starting-token` retoma la paginación exactamente donde la dejó el `NextToken` de la respuesta anterior — sin él, cada ejecución vuelve a empezar desde el principio.

## Notas y gotchas

- Las fórmulas de RCU/WCU (y sus versiones x2 en transacciones) son justo el tipo de cálculo que se repite en el examen — merece la pena tenerlas interiorizadas, no solo reconocerlas.
- Un GSI mal diseñado puede tumbar por throttling la tabla principal aunque esta tenga capacidad de sobra — no es un componente aislado, comparte destino con la tabla base.
- El patrón de "S3 para el objeto, DynamoDB para los metadatos" es la respuesta por defecto siempre que aparece un archivo grande en un diseño con DynamoDB — nunca meter el binario directamente en un elemento (límite de 400 KB, además de ser mala práctica).
- ElastiCache no es un "añadido gratis" — implica decisiones de diseño (qué estrategia de caché, qué TTL, cómo invalidar) y casi siempre cambios de código, no es solo activarlo.
- MemoryDB no es "ElastiCache pero mejor" — es un producto distinto pensado para ser la base de datos primaria (con durabilidad real), no solo una caché delante de otra BD.

## Recursos

- https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Introduction.html
- https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/capacity-mode.html
- https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/GSI.html
- https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Streams.html
- https://docs.aws.amazon.com/AmazonElastiCache/latest/dg/WhatIs.html
- https://docs.aws.amazon.com/memorydb/latest/devguide/what-is-memorydb.html
