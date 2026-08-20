# Módulo 02 — Infraestructura global del Cloud de AWS

## Resumen

Jerarquía geográfica de AWS: **Regiones → Zonas de Disponibilidad (AZ) → Edge Locations / Local Zones**.

### Regiones
- Área geográfica de AWS (Irlanda, Frankfurt, N. Virginia...). Presencia en 40+ países.
- Cómo elegir región: dónde debe quedarse legalmente el dato, cercanía a usuarios (latencia), disponibilidad del servicio que necesitas (no todo llega a todas las regiones a la vez), precio.

### Zonas de Disponibilidad (AZ)
- Cada región tiene 3-6 AZ (ej. `eu-west-2a`, `eu-west-2b`, `eu-west-2c`).
- Cada AZ = uno o más centros de datos con electricidad/refrigeración/red propias, físicamente separados de las demás.
- Conectadas entre sí por enlaces de muy baja latencia → replicar entre AZ apenas penaliza rendimiento.

### Por qué no basta una sola AZ (ni una sola región)
1. Una sola AZ → si cae, cae todo.
2. Varias AZ de la misma región → sobrevive a fallo de una AZ, pero sigue lenta para usuarios lejanos y cae si falla la región entera.
3. Varias regiones → resuelve latencia global y tolerancia a fallo de región completa.

### Edge Locations
Puntos de presencia mucho más numerosos que las regiones, para servir contenido cerca del usuario final (los usa CloudFront).

### AWS Local Zones
- "Sucursal" de una región concreta, cerca de una gran ciudad, para menos latencia aún que la región madre.
- Ej: `us-east-1` (N. Virginia) con Local Zones en Miami, Chicago, Dallas.
- Casos de uso: apps muy sensibles a latencia, migraciones híbridas, residencia de datos.

### Servicios regionales vs. globales
| Regionales (eliges dónde) | Globales (uno para toda la cuenta) |
|---|---|
| EC2 | IAM |
| Elastic Beanstalk | Route 53 |
| Lambda | CloudFront |
| Rekognition | WAF |

### Organizar varias cuentas AWS
- Patrón típico: cuenta de gestión (root + MFA + presupuesto) → usuario admin gestiona cuentas separadas por entorno.
- Truco Gmail: `tucorreo+cuenta1@gmail.com` y `tucorreo+cuenta2@gmail.com` llegan al mismo buzón pero AWS los trata como direcciones distintas — útil para dar de alta cuentas de prueba sin crear correos nuevos.

## Comandos clave

```bash
# Listar todas las regiones disponibles para tu cuenta
aws ec2 describe-regions --output table

# Listar las AZ de la región configurada actualmente
aws ec2 describe-availability-zones --output table

# Ver qué región/perfil está usando la CLI ahora mismo
aws configure list
```

## Notas y gotchas

- El truco del `+` en Gmail es muy útil para el [Módulo 04 — Caso práctico real: Organización en el Cloud](../modulo-04-caso-practico-organizacion-cloud/README.md), donde se practica con varias cuentas.
- No asumas que un servicio nuevo está disponible en tu región habitual — conviene comprobarlo antes de diseñar sobre él.

## Recursos

- https://aws.amazon.com/es/about-aws/global-infrastructure/
- https://aws.amazon.com/es/about-aws/global-infrastructure/localzones/locations/
- https://aws.amazon.com/about-aws/global-infrastructure/regional-product-services/
