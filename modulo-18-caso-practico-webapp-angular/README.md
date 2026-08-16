# Módulo 18 — Caso práctico: WebApp con Angular en AWS

## Resumen

Caso práctico corto que combina varios servicios ya vistos (Route 53, CloudFront, S3, ACM) en una arquitectura real de despliegue de una SPA (Single Page Application) — en este caso una app Angular, pero el patrón sirve para cualquier frontend estático (React, Vue...).

Implementación real de este caso práctico: [angular-aws-static-hosting](https://github.com/ccleren/angular-aws-static-hosting).

### Arquitectura
1. El usuario resuelve el dominio vía **Route 53** (búsqueda DNS).
2. La petición llega a **CloudFront**, que sirve el contenido cacheado desde el edge más cercano.
3. CloudFront usa un certificado de **AWS Certificate Manager (ACM)** para servir el sitio por HTTPS.
4. El origen de CloudFront es un bucket de **Amazon S3** que aloja los ficheros estáticos ya compilados de la app Angular (el `build`/`dist`).

### Por qué esta combinación y no servir directamente desde S3
- S3 solo sirve HTTP en el hosting de sitios web estático; para HTTPS con dominio propio hace falta CloudFront + ACM delante.
- CloudFront añade caché global (mejor rendimiento) y reduce peticiones directas al bucket.
- Route 53 con un registro Alias apuntando a la distribución de CloudFront evita depender de la URL genérica `*.cloudfront.net`.
- Es el mismo patrón "separar contenido estático" visto en el [Módulo 17](../modulo-17-cloudfront/README.md), aplicado a una SPA completa en vez de solo a assets sueltos.

### Dónde encaja cada pieza ya estudiada
| Servicio | Rol en este caso práctico | Módulo relacionado |
|---|---|---|
| S3 | Almacena los ficheros estáticos compilados de Angular | [Módulo 13](../modulo-13-s3/README.md) |
| CloudFront | CDN + terminación HTTPS delante de S3 | [Módulo 17](../modulo-17-cloudfront/README.md) |
| ACM | Certificado TLS para el dominio propio | — |
| Route 53 | DNS del dominio, registro Alias hacia CloudFront | [Módulo 16](../modulo-16-route53/README.md) |

## Comandos clave

```bash
# Subir el build de la app (contenido estático) al bucket de origen
aws s3 sync <ruta-build-local> s3://<bucket-name> --delete

# Solicitar un certificado público para el dominio (debe pedirse en us-east-1 si el
# certificado lo va a usar CloudFront, independientemente de en qué región esté todo lo demás)
aws acm request-certificate \
  --domain-name <dominio> --validation-method DNS --region us-east-1

# Consultar el estado/validación de un certificado
aws acm describe-certificate --certificate-arn <certificate-arn> --region us-east-1

# Invalidar la caché de CloudFront tras cada despliegue nuevo
aws cloudfront create-invalidation --distribution-id <distribution-id> --paths "/*"
```

## Notas y gotchas

- Un certificado de ACM que va a usar una distribución de CloudFront **debe solicitarse en `us-east-1`**, sin importar en qué región esté el resto de la infraestructura — es una limitación específica de CloudFront, no de ACM en general.
- Tras cada despliegue de una nueva versión de la app hay que invalidar la caché de CloudFront (o versionar los assets con hash en el nombre) — si no, los usuarios pueden seguir viendo la versión antigua hasta que expire el TTL.
- Este patrón (S3 + CloudFront + ACM + Route 53) es el estándar para SPAs estáticas en AWS — vale la pena tenerlo interiorizado como plantilla, se repite mucho en arquitecturas reales.

## Recursos

- https://docs.aws.amazon.com/acm/latest/userguide/gs.html
- https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/GettingStarted.html
- https://docs.aws.amazon.com/AmazonS3/latest/userguide/WebsiteHosting.html
