# Módulo 10 — Elastic Load Balancing (ELB)

## Resumen

### Escalabilidad, elasticidad, alta disponibilidad — no son lo mismo
- **Escalabilidad**: aguantar más carga (vertical = más hardware, horizontal = más nodos).
- **Elasticidad**: el sistema se autoescala solo según demanda.
- **Alta disponibilidad**: sobrevivir a la pérdida de un centro de datos entero → app repartida en 2+ AZ. Escalado horizontal necesita ASG + Load Balancer, ambos multi-AZ.

### Para qué sirve un Load Balancer
Puerta de entrada única que reparte tráfico entre instancias backend, solo a las sanas.
- Reparte carga entre varias instancias.
- Un único punto de acceso (DNS).
- Tolera fallos de instancias sin caer el servicio.
- Health checks continuos.
- Cifrado HTTPS centralizado.
- Alta disponibilidad entre zonas.

### ELB = balanceador gestionado por AWS
| Tipo | Capa OSI | Protocolo | Para qué |
|---|---|---|---|
| **ALB** | 7 | HTTP/HTTPS | Enrutamiento avanzado, microservicios, contenedores |
| **NLB** | 4 | TCP/UDP | Máximo rendimiento, mínima latencia, IP fija |
| **GWLB** | 3 | GENEVE | Insertar firewalls/appliances de terceros en el tráfico |
| ~~CLB~~ | 4 y 7 | — | Retirado en 2023 |

### ALB — enrutamiento
Dirige tráfico según: ruta de la URL, subdominio, query string o cabeceras HTTP. Cada regla apunta a un **Target Group** distinto (AWS o on-prem).

### NLB — Target Groups
Puede apuntar a: instancias EC2, IPs privadas, u otro ALB. Health checks por TCP/HTTP/HTTPS. No está en la capa gratuita.

### GWLB
Inserta dispositivos de seguridad de terceros (firewalls, IDS/IPS) en el flujo de tráfico de la VPC sin reconfigurar rutas — centraliza inspección.

### Sticky Sessions
- Solo en **ALB**. Cookie `AWSALB` (7 días) fija al cliente a la misma instancia.
- ⚠️ Puede desequilibrar la carga entre instancias.

### Balanceo cruzado (cross-zone)
Reparte tráfico entre todas las instancias de todas las AZ, no solo las de su propia AZ.

| Tipo | Por defecto | Coste tráfico entre AZ |
|---|---|---|
| ALB | Activado | Gratis |
| NLB / GWLB | Desactivado | Se paga si lo activas |

### SSL/TLS y ACM
- Cifra el tráfico cliente↔Load Balancer. TLS es la versión actual (se sigue diciendo "SSL" por costumbre).
- **ACM**: emite/gestiona/renueva certificados automáticamente. Públicos gratis. Se integra con ELB, API Gateway, CloudFront.
- **SNI**: sirve varios dominios/certificados en el mismo listener HTTPS — el cliente indica el hostname en el handshake. Solo ALB, NLB, CloudFront.

### Health Checks
| Estado | Significado |
|---|---|
| `Initial` | Registrándose |
| `Healthy` | Responde bien |
| `Unhealthy` | Falló varios checks, sin tráfico |
| `Unused` | No registrado |
| `Draining` | Retirándose progresivamente |
| `Unavailable` | Sin checks ejecutándose |

Valores por defecto: HTTP, puerto 80, ruta `/`, timeout 5s, intervalo 30s, 3 aciertos para sano / 5 fallos para no sano, código 200 esperado.

⚠️ Si **todas** las instancias están unhealthy, el balanceador sigue enviándoles tráfico igualmente (no tiene alternativa) — hay que reaccionar rápido.

### Monitorización
CloudWatch recibe automático: códigos 2XX-5XX, `HealthyHostCount`/`UnHealthyHostCount`, latencia, nº de solicitudes.

### Troubleshooting
| Error | Causa |
|---|---|
| 400 Bad Request | Petición del cliente mal formada |
| 503 Service Unavailable | Sin instancias disponibles → revisar `HealthyHostCount` y AZs |
| 504 Gateway Timeout | Backend tardó demasiado → revisar keep-alive y timeout del LB vs servidor |

## Comandos clave

```bash
# Crear un Application Load Balancer
aws elbv2 create-load-balancer \
  --name mi-alb --subnets <subnet-id-a> <subnet-id-b> \
  --security-groups <security-group-id> --type application

# Crear un target group con health check
aws elbv2 create-target-group \
  --name mi-target-group --protocol HTTP --port 80 --vpc-id <vpc-id> \
  --health-check-path /health

# Registrar instancias en el target group
aws elbv2 register-targets \
  --target-group-arn arn:aws:elasticloadbalancing:<region>:<account-id>:targetgroup/mi-target-group/<id> \
  --targets Id=<instance-id>

# Crear el listener (regla de entrada del LB)
aws elbv2 create-listener \
  --load-balancer-arn arn:aws:elasticloadbalancing:<region>:<account-id>:loadbalancer/app/mi-alb/<id> \
  --protocol HTTP --port 80 \
  --default-actions Type=forward,TargetGroupArn=arn:aws:elasticloadbalancing:<region>:<account-id>:targetgroup/mi-target-group/<id>

# Ver el estado de salud de los targets
aws elbv2 describe-target-health \
  --target-group-arn arn:aws:elasticloadbalancing:<region>:<account-id>:targetgroup/mi-target-group/<id>
```

## Notas y gotchas

- Examen: ALB = capa 7, NLB = capa 4, GWLB = capa 3.
- Sticky sessions solo en ALB, no en NLB.
- Cross-zone: gratis y activado por defecto en ALB; en NLB/GWLB hay que activarlo y se paga.
- Ver [[modulo-06-ec2-basico]]: el SG de las instancias debería permitir tráfico solo desde el SG del balanceador, nunca abrirlo directo a internet.

## Recursos

- https://docs.aws.amazon.com/es_es/elasticloadbalancing/latest/application/target-group-health-checks.html
- https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-cloudwatch-metrics.html
- https://docs.aws.amazon.com/es_es/elasticloadbalancing/latest/classic/ts-elb-error-message.html
