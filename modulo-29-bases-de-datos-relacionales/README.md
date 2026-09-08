# Módulo 29 — Bases de datos relacionales (RDS y Aurora)

## Resumen

### Amazon RDS
- Servicio de bases de datos relacionales **gestionado**, con SQL como lenguaje de consulta.
- Motores soportados: MariaDB, Microsoft SQL Server, MySQL, Oracle, PostgreSQL.

### RDS vs. base de datos en EC2
- RDS gestiona por ti: aprovisionamiento, parcheo del SO, backups continuos con restauración punto-en-el-tiempo, dashboards de monitorización, réplicas de lectura, Multi-AZ para disaster recovery, escalado (vertical/horizontal), almacenamiento sobre EBS.
- A cambio, **no hay acceso SSH** a la instancia de base de datos.

### Autoescalado de almacenamiento
- RDS puede aumentar el almacenamiento automáticamente al detectar que se está agotando, hasta un umbral máximo que tú defines — evita tener que escalar manualmente, útil con cargas de trabajo imprevisibles. Soportado por todos los motores RDS.

### Réplicas de lectura
- Copia de una base de datos principal, sincronizada continuamente, usada para escalar lecturas y mejorar disponibilidad.
- Separan lecturas de escrituras: la app sigue escribiendo en la instancia principal y puede repartir lecturas entre varias réplicas.
- Requiere actualizar la cadena de conexión de la app para aprovechar las réplicas — no es automático desde el punto de vista de la aplicación.

### RDS Multi-AZ — instancia en espera (standby)
- Replicación **síncrona** hacia una instancia standby en otra AZ de la misma región.
- La standby **no puede usarse** para lecturas ni escrituras — solo existe para failover.
- Failover: 60-120 segundos. Solo se permite una standby. Tiene coste adicional (no es capa gratuita).
- Cubre: fallo de AZ, fallo de la instancia principal, failover manual, cambio de tipo de instancia, parches.

### RDS Multi-AZ — dos instancias legibles (multi-AZ DB cluster)
- Un escritor + dos lectores en distintas AZ, con replicación **asíncrona** vía registros de transacción (más eficiente).
- Los lectores sí se pueden usar para lecturas — cierta escalabilidad de lectura además de alta disponibilidad.
- Failover más rápido: ~35 segundos.

### Coste de red entre réplicas
- Réplicas dentro de la **misma región**: sin coste de red.
- Réplicas en **regiones distintas**: se paga tarifa de transferencia entre regiones.

### RDS Custom
| | RDS estándar | RDS Custom |
|---|---|---|
| Gestión de BD y SO | Totalmente por AWS | Compartida — acceso administrativo al SO y la BD |
| Acceso | Sin SSH | SSH o SSM Session Manager a la instancia EC2 subyacente |
| Motores | Todos los soportados | Oracle y SQL Server |
| Caso de uso | Uso estándar | Apps que necesitan personalizar el SO/BD subyacente |

### RDS Proxy
- Proxy de base de datos gestionado: agrupa y comparte conexiones entre la app y la BD.
- Reduce estrés de recursos (CPU/RAM) y conexiones abiertas; reduce el tiempo de failover de RDS/Aurora hasta un 66%.
- No requiere cambios de código en la mayoría de apps; aplica permisos IAM; **nunca es accesible públicamente** — solo desde dentro de la VPC.
- Soporta RDS (MySQL, PostgreSQL, MariaDB) y Aurora (MySQL, PostgreSQL).

### Bloqueos (locks)
- Las bases relacionales bloquean filas/tablas para evitar escrituras concurrentes conflictivas.
- **Bloqueo compartido (S LOCK)**: varios usuarios pueden leer, nadie puede escribir mientras esté activo (`FOR SHARE`).
- **Bloqueo exclusivo (X LOCK)**: solo una transacción puede tenerlo; bloquea lectura y escritura de otros (`FOR UPDATE`).
```sql
-- Bloquear una tabla completa para escritura
LOCK TABLES employees WRITE;

--Desbloquear tablas
UNLOCK TABLES;

-- Bloqueo compartido (permite lecturas concurrentes)
SELECT * FROM employees WHERE department = 'Finance' FOR SHARE;

-- Bloqueo exclusivo
SELECT * FROM employees WHERE employee_id = 123 FOR UPDATE;
```
- ⚠️ Transacciones/bloqueos mal cerrados pueden derivar en un **deadlock**.

### Buenas prácticas operativas
- **Monitoreo**: CloudWatch para CPU, memoria, almacenamiento, latencia de réplica.
- **Backups**: programarlos en horas de baja escritura.
- **Multi-AZ/failover**: TTL de DNS bajo (≤30s) en la app, y probar el failover antes de necesitarlo de verdad.
- **Rendimiento**: I/O insuficiente ralentiza la recuperación tras un fallo — migrar a más IOPS si hace falta.
- **Protección**: limitar la tasa de peticiones (ej. en API Gateway) para no saturar la BD.

### Optimización de consultas
- Usar índices para acelerar `SELECT` (apoyarse en `EXPLAIN` para saber cuáles hacen falta).
- `ANALYZE TABLE` periódicamente, evitar escaneos completos de tabla, simplificar cláusulas `WHERE`.

## Amazon Aurora

### Qué es
- Motor propietario de AWS (no open source), compatible con drivers de **MySQL** y **PostgreSQL**.
- Rendimiento reclamado: ~5x MySQL en RDS, ~3x PostgreSQL en RDS.
- Almacenamiento autoescalable en incrementos de 10 GB hasta 128 TB.
- Réplicas con retardo <10ms; failover prácticamente instantáneo.
- Cuesta ~20% más que RDS equivalente, a cambio de más eficiencia.

### Alta disponibilidad y almacenamiento
- 6 copias de los datos en 3 AZ (4 de 6 necesarias para escribir, 3 de 6 para leer) — autorreparación con replicación entre pares, almacenamiento dividido en 100 volúmenes.
- Un volumen de almacenamiento **compartido** entre la instancia principal (writer) y las réplicas (readers) — no se duplica el dato por instancia como en RDS Multi-AZ clásico.
- Recuperación automática del writer en menos de 30 segundos.

### Endpoints de un clúster Aurora
- **Endpoint del writer**: apunta siempre a la instancia principal.
- **Endpoint del reader**: balancea automáticamente entre las réplicas de lectura activas.
- **Endpoints personalizados**: permiten dirigir cierto tráfico (ej. consultas analíticas) a un subconjunto concreto de instancias, con distinto tamaño/tipo si hace falta.

### Auto Scaling de réplicas
- Aurora añade/quita réplicas de lectura automáticamente según la carga, y reparte las lecturas entre las activas.

### Aurora Serverless
- Ajusta la capacidad de cómputo automáticamente según la demanda, sin gestionar instancias — pagas solo por lo usado, puede arrancar/parar solo.

### Bases de datos globales de Aurora
- Un clúster **principal** (lectura/escritura) en una región, y clústeres **secundarios** de solo lectura en otras regiones.
- Replicación entre regiones típicamente <1 segundo.
- Promover una región secundaria en caso de desastre: RTO < 1 minuto.
- Útil para reducir latencia de lectura global y como estrategia de disaster recovery multi-región.

### Aurora Machine Learning
- Permite invocar modelos de ML (SageMaker, Comprehend, Bedrock) directamente desde una consulta SQL, sin mover los datos ni aprender herramientas de ML nuevas.
- Casos de uso: recomendaciones en tiempo real, detección de fraude.

### Copias de seguridad y clonación
- Backups automáticos sin impacto en rendimiento, con restauración punto-en-el-tiempo hasta 35 días (solo se almacenan los cambios de bloque desde el último backup).
- **Clonación**: crea un nuevo clúster a partir de uno existente usando copy-on-write — inicialmente comparte el mismo volumen (rápido, sin copiar nada), y solo se separan los datos que cambian. Ideal para levantar un entorno de pruebas desde producción sin impactarla.
- **Restauración desde S3**: subir un backup local (mysqldump para RDS, Percona XtraBackup para Aurora) a S3 y restaurarlo como una instancia/clúster nuevo.

### Seguridad en RDS y Aurora
- **Cifrado en reposo**: vía KMS, debe definirse al crear la instancia — si la principal no está cifrada, sus réplicas tampoco pueden estarlo (hay que pasar por un snapshot y restaurar cifrado para cambiarlo después).
- **Cifrado en tránsito**: TLS por defecto.
- **Autenticación IAM**: conectarse con roles IAM en vez de usuario/contraseña.
- **Grupos de seguridad**: controlan el acceso de red.
- Sin SSH disponible (salvo RDS Custom); logs de auditoría opcionales hacia CloudWatch Logs.

## Comandos clave

```bash
# Crear una instancia RDS
aws rds create-db-instance \
  --db-instance-identifier <db-instance-id> --db-instance-class <clase> \
  --engine <motor> --master-username <usuario> --master-user-password <password> \
  --allocated-storage <gb>

# Crear una réplica de lectura
aws rds create-db-instance-read-replica \
  --db-instance-identifier <replica-id> --source-db-instance-identifier <db-instance-id>

# Convertir una instancia existente a Multi-AZ
aws rds modify-db-instance --db-instance-identifier <db-instance-id> --multi-az

# Crear un clúster Aurora
aws rds create-db-cluster \
  --db-cluster-identifier <cluster-id> --engine aurora-mysql \
  --master-username <usuario> --master-user-password <password>

# Crear un endpoint personalizado en un clúster Aurora
aws rds create-db-cluster-endpoint \
  --db-cluster-identifier <cluster-id> --db-cluster-endpoint-identifier <endpoint-id> \
  --endpoint-type READER

# Crear un RDS Proxy
aws rds create-db-proxy \
  --db-proxy-name <proxy-name> --engine-family MYSQL \
  --auth '[{"AuthScheme":"SECRETS","SecretArn":"<secret-arn>"}]' \
  --role-arn <role-arn> --vpc-subnet-ids <subnet-id-1> <subnet-id-2>
```

## Notas y gotchas

- La instancia standby de un RDS Multi-AZ clásico **no sirve para escalar lecturas** — es fácil asumirlo por analogía con las réplicas de lectura, pero su único propósito es el failover.
- RDS Proxy nunca es accesible desde fuera de la VPC — si algo necesita conectarse al proxy desde internet, la arquitectura está mal planteada, no es un límite a "sortear".
- Para cifrar una base de datos RDS/Aurora que se creó sin cifrar, no hay un botón directo — el camino es snapshot → restaurar como cifrada, no una modificación in situ.
- La clonación de Aurora (copy-on-write) es mucho más barata y rápida que un snapshot+restore tradicional para levantar un entorno de pruebas — vale la pena tenerlo presente frente al hábito de restaurar desde un backup.
- Aurora Global Database resuelve un problema distinto al de las réplicas de lectura normales: no es solo "leer más rápido", es tener un plan de recuperación ante desastres con RTO menor a un minuto en otra región.

## Recursos

- https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Welcome.html
- https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Concepts.MultiAZ.html
- https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Aurora.Overview.html
- https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/rds-proxy.html
- https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-global-database.html
