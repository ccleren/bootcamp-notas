# Módulo 28 — Introducción a las bases de datos de AWS

## Resumen

### Relacionales (SQL) vs. no relacionales (NoSQL)
- **Relacionales**: almacenan datos en formato tabular (filas y columnas) — las columnas son atributos, las filas son valores concretos.
- **No relacionales**: usan modelos de datos distintos según el caso (clave-valor, documento, columna ancha, grafo...); optimizadas para grandes volúmenes, baja latencia y esquemas flexibles.

### Servicios de AWS por tipo
| Relacionales (SQL) | No relacionales (NoSQL) |
|---|---|
| Amazon RDS | Amazon DynamoDB (clave-valor) |
| Amazon Aurora | Amazon DocumentDB (documento) |
| | Amazon Timestream (series temporales) |
| | Amazon Keyspaces (columna ancha, compatible Cassandra) |
| | Amazon Neptune (grafo) |
| | Amazon ElastiCache (caché en memoria) |

### Bases de datos relacionales (SQL)
- Los datos se organizan en tablas relacionadas entre sí mediante **claves**.
- **Clave primaria**: identifica de forma única cada fila de una tabla (ej. `P_ID` en una tabla de personas).
- **Clave compuesta**: combina varias columnas para relacionar dos tablas entre sí (ej. una tabla que cruza `P_ID` con `M_ID` para relacionar personas con sus mascotas).
- Ideal cuando los datos tienen una estructura fija y relaciones claras entre entidades.

### Bases de datos no relacionales (NoSQL) — modelos de datos
| Modelo | Cómo almacena | Ideal para |
|---|---|---|
| **Clave-valor** | Un valor asociado a una clave, sin esquema ni estructura fija | Máxima velocidad y escalabilidad (ej. contar eventos por franja horaria) |
| **Columna ancha** | Filas identificadas por una clave de partición, con columnas variables por fila | Grandes volúmenes con acceso flexible por clave |
| **Documento** | Cada registro es un documento (JSON-like) autocontenido | Datos con esquema flexible, ej. perfil de usuario o publicaciones en una red social |
| **Grafo** | Nodos (entidades) y relaciones explícitas entre ellos | Modelar relaciones complejas, ej. "persona trabaja para empresa, empresa ubicada en ciudad" |

### Almacenamiento por filas vs. por columnas
- **Almacén de filas** (ej. MySQL): cada fila completa se guarda junta — ideal para operaciones típicas de añadir/actualizar/borrar una fila.
- **Almacén de columnas** (ej. Redshift): los valores de una misma columna se guardan juntos — ideal cuando se necesita analizar todos los valores de un atributo concreto (ej. todos los precios), típico de analítica/agregaciones.

## Notas y gotchas

- La elección entre SQL y NoSQL no es "cuál es mejor" sino "qué forma tienen mis datos y cómo los voy a consultar" — datos muy relacionados y con esquema fijo piden SQL; datos de alto volumen, esquema variable o acceso simple por clave piden NoSQL.
- Cada modelo NoSQL (clave-valor, documento, columna ancha, grafo) resuelve un problema distinto — no son intercambiables entre sí solo porque todos sean "NoSQL"; el modelo de grafo, por ejemplo, no tiene sentido para lo que resuelve mejor un almacén clave-valor.
- Almacén por filas vs. por columnas es la misma distinción que aparece en el mundo del Data Warehousing: transaccional (OLTP, por filas) vs. analítico (OLAP, por columnas) — Redshift usando columnas no es casualidad, está pensado para analítica.

## Recursos

- https://aws.amazon.com/es/compare/the-difference-between-relational-and-non-relational-databases/
- https://docs.aws.amazon.com/whitepapers/latest/choosing-an-aws-nosql-database/choosing-an-aws-nosql-database.html
- https://docs.aws.amazon.com/neptune/latest/userguide/intro.html
