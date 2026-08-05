# Módulo 02 — Infraestructura global del Cloud de AWS

## Resumen

Elementos de la infraestructura global de AWS: **Regiones, Zonas de Disponibilidad (AZ), Centros de datos, Edge Locations y Local Zones**.

### Regiones
- Ubicación geográfica específica para alojar tus recursos. AWS está presente en 40+ países.
- Ventajas: alta disponibilidad, baja latencia, cumplimiento de regulaciones locales.
- **Cómo elegir una región**:
  - Cumplimiento legal / gobernanza de datos (los datos nunca salen de una región sin permiso explícito).
  - Proximidad a los clientes (menor latencia).
  - Disponibilidad de servicios (no todos los servicios/features están en todas las regiones).
  - Precio (varía por región).

### Zonas de Disponibilidad (AZ)
- Cada AZ = uno o más centros de datos discretos, con alimentación, red y conectividad **redundantes e independientes**.
- Una región tiene normalmente **mínimo 3, máximo 6 AZs** (ej. `eu-west-2a`, `eu-west-2b`, `eu-west-2c`).
- Las AZs de una región están conectadas por enlaces de baja latencia, pero están físicamente separadas para estar aisladas de catástrofes.

### Edge Locations
- Puntos de distribución de contenido lo más cerca posible del usuario final, para reducir la latencia (usadas por servicios como CloudFront).

### AWS Local Zones
- Extensión de una región AWS que sitúa cómputo/almacenamiento/BBDD cerca de grandes núcleos de población, para latencia aún más baja que la región "madre".
- Ejemplo: región `us-east-1` (N. Virginia) con Local Zones en Boston, Chicago, Dallas, Miami, Houston.
- Casos de uso: apps de baja latencia en el borde, migraciones híbridas, residencia de datos.

### Progresión del razonamiento "por qué existen las AZs y regiones" (ejemplo del curso)
1. App en **una sola AZ** en Londres → falla si cae esa AZ (baja disponibilidad) y es lenta para usuarios lejanos.
2. App en **varias AZ** de Londres → resiliente a fallo de una AZ, pero sigue siendo lenta para usuarios lejanos y cae si falla toda la región.
3. App en **varias regiones** (ej. Londres + N. Virginia) → resuelve latencia global y resiliencia ante caída de una región entera.

### Servicios globales vs. regionales
| Ámbito regional | Ámbito global |
|---|---|
| Amazon EC2 (IaaS) | AWS IAM (gestión de usuarios) |
| AWS Elastic Beanstalk (PaaS) | Amazon Route 53 (DNS) |
| AWS Lambda (FaaS) | AWS CloudFront (CDN) |
| Amazon Rekognition (SaaS) | AWS WAF (firewall de apps web) |

### Creación de cuentas AWS
- Estructura típica: **cuenta de gestión** (root, MFA, presupuesto) con usuarios IAM (ej. `IAMADMIN`) que administran cuentas hijas separadas por entorno (ej. `GENERAL`/gestión, `PRODUCTION`).
- Truco práctico para crear varias cuentas AWS con un solo Gmail: usar alias `+` en el email — `usuario+AWSAccount1@gmail.com`, `usuario+AWSAccount2@gmail.com`, etc. No hace falta crear los alias de antemano, Gmail los redirige automáticamente al buzón principal.

## Comandos clave

*(No aplica — módulo conceptual.)*

## Notas y gotchas

- El truco del `+` en Gmail para multi-cuenta es muy útil para practicar AWS Organizations sin tener que crear correos nuevos — ver [[modulo-04-organizacion-cloud]].
- Elegir bien la región desde el principio importa: no todos los servicios/features nuevos llegan a todas las regiones a la vez.

## Recursos

- https://aws.amazon.com/es/about-aws/global-infrastructure/
- https://aws.amazon.com/es/about-aws/global-infrastructure/localzones/locations/
- https://aws.amazon.com/about-aws/global-infrastructure/regional-product-services/
