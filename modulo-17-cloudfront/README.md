# Módulo 17 — CloudFront y AWS Global Accelerator

## Resumen

### Visión general de CloudFront
- CDN (red de entrega de contenidos) de AWS: cachea contenido en ubicaciones **edge** cercanas al usuario para mejorar la velocidad de lectura y la experiencia.
- Más de 700 puntos de presencia globales.
- Incluye protección DDoS e integración con Shield y WAF.
- Términos clave: **origen** (fuente del contenido: S3 o un origen HTTP personalizado), **distribución** (la unidad de configuración de CloudFront), **edge location** (caché local), **regional edge cache** (una capa de caché intermedia, mayor que una edge location).

### Orígenes de CloudFront
- **Bucket S3**: para distribuir y cachear archivos en el borde; puede usarse también como entrada para subir archivos a S3. Acceso protegido con **OAC** (Origin Access Control, sustituye al antiguo OAI) combinado con la política del bucket.
- **Origen HTTP personalizado**: ALB, instancia EC2, sitio web estático de S3 (hay que habilitarlo antes como tal) o cualquier backend HTTP.
- Con ALB/EC2 como origen hace falta abrir el grupo de seguridad a las IPs públicas de las edge locations (hay una lista pública de IPs de CloudFront para esto); si el origen es un ALB, basta con permitir el propio grupo de seguridad del ALB desde CloudFront y restringir el resto del tráfico público a las instancias.

### CloudFront vs. S3 Cross-Region Replication
| | CloudFront | S3 CRR |
|---|---|---|
| Alcance | Red global de edge locations | Debes configurar cada región destino |
| Frescura del contenido | Cacheado según TTL (puede tardar en refrescar) | Casi en tiempo real |
| Acceso | Lectura y escritura (vía el origen) | Solo lectura |
| Mejor para | Contenido estático, disponible en todo el mundo | Contenido dinámico, baja latencia en pocas regiones concretas |

### Caché en CloudFront
- Cada objeto en caché se identifica por una **clave de caché**: por defecto, hostname + ruta del recurso; se pueden añadir cabeceras, cookies o query strings a esa clave mediante políticas de caché.
- TTL configurable de 0 segundos hasta 1 año; se define desde el origen con las cabeceras `Cache-Control` y `Expires`.
- **Invalidación**: si actualizas el origen, CloudFront no se entera hasta que expire el TTL — para forzar el refresco (total o de una ruta concreta) se usa la API `CreateInvalidation`.

### Comportamiento de caché según cabeceras, cookies y query strings
Los tres funcionan bajo la misma lógica de tres niveles:
| Nivel | Efecto en caché | Cuándo usarlo |
|---|---|---|
| **Ninguno / solo por defecto** | Máximo rendimiento — misma respuesta cacheada para todos | Contenido que no depende del cliente |
| **Lista blanca (whitelist)** | Cachea una versión distinta por cada valor permitido | Necesitas variantes controladas (idioma, un parámetro de búsqueda) |
| **Reenviar todo (forward all)** | Cada combinación genera una copia distinta — casi anula la caché | Contenido totalmente dinámico/personalizado por usuario |

- Caso particular: **sesiones sticky con ALB** — hay que incluir la cookie `AWSALB` en la whitelist y poner el TTL de caché por debajo del tiempo de expiración de esa cookie, para no romper el enrutamiento pegajoso.

### Restricción geográfica
- Lista de permitidos (solo países aprobados) o lista de bloqueo (países prohibidos), usando una base de datos Geo-IP de terceros.

### Informes y logs
- **Informes**: Cache Statistics, Popular Objects, Top Referrers, Usage Reports, Viewers Report — se generan a partir de los Access Logs.
- **Logs**: cada solicitud queda registrada (IP, fecha/hora, URL, navegador, código de respuesta, tiempo de respuesta...) en un bucket S3 dedicado.

### Estrategia: separar contenido estático y dinámico
- No todo el tráfico de una app es igual — separar estático (imágenes, CSS, JS → S3) de dinámico (APIs, sesiones → ALB/Lightsail) detrás de la misma distribución mejora el ratio de cache hits, reduce carga al backend y baja el coste. Patrón típico: Route 53 → CloudFront → (S3 para estático / ALB para dinámico).

### Precios
- Se cobra por transferencia de datos saliente hacia internet (varía según región) y por invocación (~0,10 USD por millón de invocaciones).
- **Clases de precio**: Todas las regiones (mejor rendimiento), Clase 200 (excluye las regiones más caras), Clase 100 (solo las regiones más baratas) — reducir edge locations usadas reduce coste.

### AWS Global Accelerator
- Acerca la red troncal de AWS al cliente: el tráfico entra por el edge más cercano usando **IPs anycast** (la misma IP asignada a múltiples ubicaciones) y viaja por la red privada de AWS hasta el destino, con menos saltos por internet público.
- Asigna 2 IPs anycast fijas como punto de entrada único.
- A diferencia de CloudFront (enfocado en HTTP/S y con caché), Global Accelerator funciona también con **TCP/UDP no-HTTP** y **no cachea nada** — es puro enrutamiento de red, no una CDN.

## Comandos clave

```bash
# Invalidar toda la caché de una distribución (o una ruta concreta con --paths "/imagenes/*")
aws cloudfront create-invalidation \
  --distribution-id <distribution-id> --paths "/*"

# Listar las distribuciones de la cuenta
aws cloudfront list-distributions

# Ver el detalle/estado de una distribución
aws cloudfront get-distribution --id <distribution-id>

# Crear un accelerator de Global Accelerator
aws globalaccelerator create-accelerator \
  --name <accelerator-name> --idempotency-token <token-unico> --region us-west-2

# Añadir un listener (puerto/protocolo de entrada) al accelerator
aws globalaccelerator create-listener \
  --accelerator-arn <accelerator-arn> --protocol TCP \
  --port-ranges FromPort=<puerto>,ToPort=<puerto> \
  --idempotency-token <token-unico> --region us-west-2

# Añadir un grupo de endpoints (destinos) a un listener, en una región concreta
aws globalaccelerator create-endpoint-group \
  --listener-arn <listener-arn> --endpoint-group-region <region-destino> \
  --idempotency-token <token-unico> --region us-west-2
```

*(`globalaccelerator` es un servicio global gestionado siempre desde `us-west-2`, aunque los endpoints estén en otras regiones — por eso el `--region us-west-2` en todos los comandos anteriores.)*

## Notas y gotchas

- CloudFront con **"forward all headers"** desactiva la caché por completo (obliga TTL=0) — es fácil activarlo sin querer pensando que solo añade información, y en realidad anula el propósito de usar CloudFront.
- Lo mismo aplica a cookies y query strings: cuantos más elementos entran en la clave de caché, más variantes se generan y peor ratio de cache hit — la opción por defecto (ignorar) suele ser la más rentable salvo que necesites personalización real.
- La invalidación de caché (`CreateInvalidation`) tiene coste según el número de rutas invalidadas — no es gratis usarla como sustituto de un TTL bien ajustado.
- Global Accelerator no es una alternativa a CloudFront, es complementaria: CloudFront cachea HTTP/S, Global Accelerator enruta cualquier tráfico TCP/UDP por la red privada de AWS sin cachear nada — la pregunta clave para elegir es "¿necesito caché?" más que "¿cuál es mejor?".
- Con ALB/EC2 como origen, olvidar abrir el grupo de seguridad a las IPs públicas de las edge locations es un error típico que hace parecer que CloudFront "no llega" al origen.

## Recursos

- https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/Introduction.html
- https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/Invalidation.html
- https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/RequestAndResponseBehaviorS3Origin.html
- https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/Expiration.html
- https://docs.aws.amazon.com/global-accelerator/latest/dg/what-is-global-accelerator.html
