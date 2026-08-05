# Módulo 10 — Elastic Load Balancing (ELB)

## Resumen

### Escalabilidad, elasticidad y alta disponibilidad
- **Escalabilidad**: capacidad de acomodar más carga — vertical (más hardware) u horizontal (más nodos).
- **Elasticidad**: una vez el sistema es escalable, se autoescala según la carga real.
- **Alta disponibilidad**: ejecutar la app en **al menos 2 AZ**, para sobrevivir a la pérdida de un centro de datos entero. El escalado horizontal necesita infraestructura extra: ASG + Load Balancer, ambos multi-AZ.

### ¿Qué es el Load Balancing?
Servidor que reparte el tráfico entrante entre varias instancias EC2 (u otros destinos), dirigiendo tráfico solo a las que están sanas. Motivos para usarlo:
- Distribuir carga entre varias instancias.
- Exponer **un único punto de acceso** (DNS) de la app.
- Tolerar fallos de instancias sin interrumpir el servicio.
- Health checks periódicos.
- Terminación SSL/HTTPS centralizada.
- Alta disponibilidad entre zonas.

### ELB = Load Balancer gestionado por AWS
AWS se encarga del funcionamiento, actualizaciones, mantenimiento y alta disponibilidad — tú solo configuras. Tipos:

| Tipo | Capa OSI | Protocolo | Para qué |
|---|---|---|---|
| **Application LB (ALB)** | 7 | HTTP/HTTPS | Enrutamiento HTTP avanzado, microservicios, contenedores |
| **Network LB (NLB)** | 4 | TCP/UDP | Alto rendimiento (millones de req/s), baja latencia (~100ms vs ~400ms del ALB), IP estática |
| **Gateway LB (GWLB)** | 3 | GENEVE (puerto 6081) | Insertar firewalls/appliances de terceros en el flujo de tráfico de la VPC |
| ~~Classic LB (CLB)~~ | 4 y 7 | — | Retirado en 2023 |

### Application Load Balancer (ALB) — enrutamiento
Puede dirigir tráfico según:
- **Ruta de la URL** (`misitio.com/clientes` vs `/articulos`).
- **Nombre de host / subdominio** (`ventas.misitio.com` vs `soporte.misitio.com`).
- **Query string o cabeceras HTTP** (ej. `?categoria=libros`, o el `Platform` header para separar tráfico móvil/desktop).
- Cada regla apunta a un **Target Group** distinto (ej. Target Group EC2 en AWS vs. Target Group on-premises).

### Network Load Balancer (NLB) — Target Groups
Puede apuntar a: instancias EC2, direcciones IP privadas, o incluso otro ALB. Health checks soportan TCP, HTTP y HTTPS. **No está en la capa gratuita.**

### Gateway Load Balancer (GWLB)
Redirige el tráfico transparentemente a dispositivos de seguridad de terceros (firewalls, IDS/IPS) sin tener que reconfigurar rutas de red — centraliza la inspección de tráfico.

### Sticky Sessions (sesiones persistentes)
- Solo en **ALB**. Redirige siempre al mismo cliente a la misma instancia backend, usando una cookie `AWSALB` (caduca a los 7 días).
- ⚠️ Activarlo puede provocar **desequilibrio de carga** entre instancias.

### Load Balancer de zona cruzada (cross-zone)
Cada nodo del LB reparte tráfico entre **todas** las instancias registradas en **todas** las AZ (no solo las de su propia AZ).

| Tipo | Por defecto | Coste por cruzar AZ |
|---|---|---|
| ALB | Activado | Gratis |
| NLB / GWLB | Desactivado | Se paga si lo activas |
| CLB | Desactivado | Gratis |

### SSL/TLS y ACM
- **SSL/TLS** cifra el tráfico en tránsito entre cliente y Load Balancer. TLS es la versión moderna (la gente sigue diciendo "SSL" por costumbre).
- **AWS Certificate Manager (ACM)**: aprovisiona, gestiona y **renueva automáticamente** certificados TLS. Certificados públicos **gratis**. Se integra con ELB, API Gateway, CloudFront.
- El Load Balancer usa un certificado X.509; puedes gestionar múltiples certificados/dominios en un mismo listener HTTPS con **SNI (Server Name Indication)** — el cliente indica el hostname en el handshake TLS para que el servidor sirva el certificado correcto. Solo funciona en ALB, NLB y CloudFront.

### Health Checks
| Estado del target | Significado |
|---|---|
| `Initial` | Registrándose |
| `Healthy` | Responde bien a los checks |
| `Unhealthy` | Falló varios checks → no recibe tráfico |
| `Unused` | No registrado en el grupo |
| `Draining` | Se está retirando (deja de recibir tráfico gradualmente) |
| `Unavailable` | No se están ejecutando checks |

Parámetros por defecto: protocolo `HTTP`, puerto `80`, ruta `/`, timeout `5s`, intervalo `30s`, `HealthyThresholdCount=3`, `UnhealthyThresholdCount=5`, matcher código `200`.

⚠️ **Si todas las instancias están "unhealthy", el LB sigue enviándoles tráfico igualmente** (no tiene alternativa) — hay que actuar rápido para recuperarlas.

### Monitorización
Métricas automáticas a CloudWatch (sin configurar nada): códigos HTTP 2XX-5XX, `HealthyHostCount`/`UnHealthyHostCount`, latencia, nº de solicitudes.

### Troubleshooting típico
| Error | Causa / solución |
|---|---|
| **400 Bad Request** | Solicitud del cliente mal formada |
| **503 Service Unavailable** | No hay instancias disponibles → revisa `HealthyHostCount`, asegúrate de tener instancias sanas en todas las AZ |
| **504 Gateway Timeout** | La EC2 tardó demasiado en responder → revisa keep-alive y que el timeout del LB sea mayor que el del servidor |

## Comandos clave

*(No aplica — configuración vía consola: creación de ALB/NLB, target groups, listeners y health checks.)*

## Notas y gotchas

- ALB = capa 7 (HTTP), NLB = capa 4 (TCP/UDP), GWLB = capa 3 — clásico de examen, no confundir capas.
- Sticky sessions solo existen en ALB, no en NLB.
- Cross-zone load balancing: ALB lo trae gratis por defecto; en NLB/GWLB hay que activarlo explícitamente y **se paga** el tráfico entre AZ.
- Recuerda: los Security Groups del Load Balancer y de las EC2 son capas separadas — ver [[modulo-06-ec2-basico]]. El SG de las instancias debe permitir tráfico **solo desde el SG del Load Balancer**, no desde internet directamente.

## Recursos

- https://docs.aws.amazon.com/es_es/elasticloadbalancing/latest/application/target-group-health-checks.html
- https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-cloudwatch-metrics.html
- https://docs.aws.amazon.com/es_es/elasticloadbalancing/latest/classic/ts-elb-error-message.html
