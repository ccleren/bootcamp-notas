# Módulo 16 — Amazon Route 53

## Resumen

### Qué resuelve el DNS
- El DNS (sistema de nombres de dominio) traduce nombres legibles (`website.com`) a direcciones IP (`172.17.91.32`) — memorizar IPs para cada web/servicio no es viable, y menos aún si esas IPs cambian con el tiempo.
- Resolución típica de una petición: navegador → resolver DNS local (asignado por tu ISP/empresa) → servidor raíz → servidor TLD (`.com`) → servidor del dominio (SLD) → IP final. Cada salto puede cachear la respuesta según su TTL.

### Qué es Route 53
- Servicio DNS de AWS: dirige usuarios a aplicaciones de forma fiable, con servidores DNS distribuidos globalmente y escalado automático.
- El nombre viene del puerto tradicional de DNS (53).
- Además de DNS, funciona como **registrador de dominios**.
- Soporta zonas de alojamiento **públicas** y **privadas**.

### Zonas de alojamiento (hosted zones)
| Tipo | Accesible desde | Caso de uso |
|---|---|---|
| **Pública** | Internet + VPCs de AWS, vía 4 servidores de nombres (NS) dedicados | Dominios de cara al público; también puedes apuntar un dominio comprado fuera de AWS hacia aquí |
| **Privada** | Solo las VPCs asociadas explícitamente | Entornos de dev/test que simulan producción sin exponer nada a internet |

### Registros DNS
- Cada registro tiene: **nombre** de dominio/subdominio, **tipo**, **valor**, **política de enrutamiento** y **TTL** (obligatorio salvo en registros Alias).
- Tipos básicos: `A` (IPv4), `AAAA` (IPv6), `CNAME` (alias de un dominio hacia otro dominio), `NS`, `MX` (servidores de correo, con prioridad).
- Tipos avanzados: `CAA`, `DS`, `NAPTR`, `PTR`, `SOA`, `TXT`, `SPF`, `SRV`.
- **TTL**: tiempo que un resolver cachea la respuesta antes de volver a preguntar. TTL bajo = cambios se propagan rápido pero más tráfico a Route 53, TTL alto = menos tráfico pero cambios más lentos en propagarse.

### Registros Alias
- Extensión de Route 53 (no un tipo DNS estándar) para apuntar un nombre a un **recurso de AWS** (ELB, CloudFront, API Gateway, Elastic Beanstalk, sitio web S3, App Runner, AppSync, Global Accelerator, VPC Interface Endpoints, u otro registro de la misma hosted zone).
- Siempre es de tipo `A`/`AAAA`, reconoce automáticamente si cambia la IP del recurso destino, y **no admite TTL configurable** (lo gestiona Route 53).
- ⚠️ No se puede crear un Alias apuntando directamente al nombre DNS de una instancia EC2.

### Políticas de enrutamiento
| Política | Comportamiento |
|---|---|
| **Simple** | Devuelve un único recurso (o varios valores, y el cliente elige uno al azar); no admite health checks |
| **Ponderado** | Reparte tráfico entre varios recursos según un peso relativo sobre el total; si un registro no está sano se descarta y se reelige |
| **Latencia** | Devuelve el recurso con menor latencia estimada hacia el usuario, entre los que estén sanos |
| **Conmutación por error (failover)** | Dirige al recurso primario; si su health check falla, pasa al secundario |
| **Geolocalización** | Responde según continente/país/estado del usuario; requiere un registro "por defecto" para el resto de casos |
| **Geoproximidad** | Ajusta cuánto tráfico recibe cada recurso según su ubicación geográfica y un valor de "tendencia" (bias) que amplía o reduce esa zona |
| **Basado en IP** | Enruta según rangos CIDR del cliente, útil para optimizar rendimiento o coste de red |
| **Varios valores** | Devuelve hasta 8 registros sanos (de más que haya, elige 8 al azar) — mejora disponibilidad, pero no sustituye a un balanceador |

### Health checks (comprobaciones de salud)
- Protocolos soportados: HTTP, HTTPS, TCP. Solo válidos para **recursos públicos**.
- ~15 verificadores globales comprueban el endpoint cada 30s (10s con coste extra); se considera sano si más del 18% de los verificadores lo reportan como tal.
- Pasan solo con respuestas 2xx/3xx; se puede además exigir cierto texto en los primeros 5120 bytes de la respuesta.
- **Health checks calculados**: combinan hasta 256 checks "hijo" en uno "padre", definiendo cuántos deben pasar para que el padre pase — útil para mantenimiento parcial sin tumbar todo el servicio.
- Para recursos **privados** (dentro de una VPC): los verificadores de Route 53 no pueden alcanzarlos directamente — se resuelve creando una alarma de CloudWatch sobre el recurso y un health check que vigila esa alarma.

### Importar registros DNS externos
- Si compraste el dominio en otro registrador (ej. Wordpress) pero quieres gestionar el DNS desde Route 53, puedes importar esos registros — es una operación soportada, no un caso especial.

## Comandos clave

```bash
# Crear una hosted zone pública
aws route53 create-hosted-zone \
  --name <dominio> --caller-reference <id-unico-para-la-peticion>

# Listar las hosted zones de la cuenta
aws route53 list-hosted-zones

# Listar los registros de una hosted zone
aws route53 list-resource-record-sets --hosted-zone-id <hosted-zone-id>

# Crear/modificar/borrar registros (vía un archivo JSON de cambios)
aws route53 change-resource-record-sets \
  --hosted-zone-id <hosted-zone-id> --change-batch file://<ruta-cambios.json>

# Crear un health check HTTP sobre un endpoint público
aws route53 create-health-check \
  --caller-reference <id-unico-para-la-peticion> \
  --health-check-config Type=HTTP,ResourcePath=<ruta-health>,FullyQualifiedDomainName=<dominio>,Port=80,RequestInterval=30,FailureThreshold=3

# Consultar el estado de un health check
aws route53 get-health-check-status --health-check-id <health-check-id>
```

Ejemplo de `change-batch` para crear un registro `A`:
```json
{
  "Changes": [
    {
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "<subdominio.dominio>",
        "Type": "A",
        "TTL": 300,
        "ResourceRecords": [{ "Value": "<ip>" }]
      }
    }
  ]
}
```

## Notas y gotchas

- No confundir zona de alojamiento **privada** con "DNS privado que también resuelve desde fuera" — es literalmente inaccesible fuera de las VPCs asociadas, no hay excepción.
- Un registro Alias no lleva TTL propio (lo gestiona Route 53 internamente) — si ves un Alias sin TTL en la consola no es un error, es el comportamiento esperado.
- Un registro Alias **no puede apuntar al nombre DNS de una instancia EC2** directamente — solo a los recursos AWS soportados (ELB, CloudFront, S3, etc.) o a otro registro de la misma zona.
- La política **Ponderada** con peso `0` en un registro significa que nunca se devuelve, salvo que **todos** los registros tengan peso `0` — entonces se consideran todos.
- Los health checks de Route 53 corren desde fuera de tu VPC — para monitorizar algo privado hace falta el rodeo vía alarma de CloudWatch, no hay forma directa.
- La política de **varios valores** mejora disponibilidad pero no es un sustituto de un Load Balancer real (no hace nada de reparto de carga inteligente, solo devuelve varias IPs sanas).

## Recursos

- https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/Welcome.html
- https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/ResourceRecordTypes.html
- https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/routing-policy.html
- https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/dns-failover.html
- https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/hosted-zones-private.html
