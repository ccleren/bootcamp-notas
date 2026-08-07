# Módulo 12 — Monitorización, resolución de problemas y auditoría de AWS

## Resumen

**CloudWatch** (monitorización), **X-Ray** (trazabilidad de peticiones) y **CloudTrail** (auditoría de quién hizo qué).

### Por qué monitorizar
- A los usuarios no les importa cómo desplegaste la app, solo que funcione (latencia, caídas).
- Supervisión interna: prevenir problemas antes de que ocurran, controlar coste/rendimiento, detectar tendencias.

### Amazon CloudWatch
- Monitoreo en tiempo real de recursos y apps AWS. Cuatro piezas: **Métricas**, **Logs**, **Eventos**, **Alarmas**.
- Métricas destacables: `CPUUtilization`, `NetworkIn/NetworkOut`, `DiskReadOps/DiskWriteOps`, `Latency`.
- Monitorización estándar: cada 5 min. **Detallada** (de pago, 10 métricas gratis): cada 1 min — útil para que el ASG escale rápido.
- ⚠️ La RAM **no** se envía por defecto — hace falta el agente (ver [Módulo 07](../modulo-07-ec2-avanzado/README.md)).

### CloudWatch Logs
- Centraliza logs de apps/servicios en **Log Groups** → **Log Streams** → eventos con timestamp.
- Política de expiración configurable (nunca / 30 días / etc.).
- Fuentes: agente CloudWatch, Elastic Beanstalk, ECS, Lambda, VPC Flow Logs, API Gateway, CloudTrail, Route 53.
- Destinos de exportación: S3, Kinesis Data Streams, Kinesis Data Firehose, Lambda.
- **Exportación a S3**: tarda hasta 12h en estar disponible, vía `CreateExportTask` — no es tiempo real (usa suscripciones).
- **Suscripciones a Logs**: casi en tiempo real, vía Lambda/Kinesis Data Firehose.
- **CloudWatch Logs Insights**: consultar logs con lenguaje de query y volcar resultados a dashboards.
- Por defecto, **los logs de una EC2 no llegan solos a CloudWatch** — hace falta el agente + permisos IAM.

### Agente unificado de CloudWatch
- Versión moderna del agente (el antiguo "agente de logs" solo mandaba logs) manda **métricas de sistema + logs**.
- Métricas que recoge: CPU (activa/inactiva/sistema/usuario), disco (espacio + IOPS), RAM, netstat (conexiones TCP/UDP), procesos, swap.

### CloudWatch Alarms
- Vigila una métrica, dispara una acción al cruzar un umbral. Estados: `OK`, `INSUFFICIENT_DATA`, `ALARM`.
- Acciones posibles: parar/terminar/reiniciar/recuperar una EC2, activar Auto Scaling, notificar a SNS (email, Lambda, etc.).
- Ej: alarma de facturación sobre la métrica de billing de CloudWatch.

### Amazon EventBridge
- Conecta eventos de AWS y de SaaS externos (Zendesk, Datadog, Auth0...) con automatizaciones.
- Tres tipos de bus: **default** (eventos de servicios AWS), **de socios** (SaaS externos), **personalizado** (tus propias apps).
- **Registro de esquemas**: infiere y versiona la estructura de los eventos, permite generar código a partir de ella.
- **Política basada en recursos**: permite agregar eventos de otras cuentas/regiones a un bus centralizado (patrón multi-cuenta).
- Ej: EC2/RDS mandan alertas de estado → EventBridge dispara Lambda (autoescalar, reiniciar) → notifica a Slack/Teams.

### AWS X-Ray
- Resuelve el problema de depurar sistemas distribuidos: sin X-Ray, cada servicio tiene sus propios logs con formatos distintos y no hay vista conjunta de la arquitectura.
- Construye un **mapa de servicios** a partir de trazas — visual, para que perfiles no técnicos puedan ayudar a diagnosticar.
- Para qué sirve: encontrar cuellos de botella, entender dependencias entre microservicios, localizar errores/excepciones, cumplir SLA, identificar usuarios afectados.
- Compatible con: Lambda, Elastic Beanstalk, API Gateway, ECS, ELB, EC2.
- **Trazas (traces)**: siguen una petición de principio a fin, compuestas de **segmentos** (+ subsegmentos), con anotaciones opcionales. Se puede trazar el 100% o solo una muestra (%, o tasa/minuto).
- Seguridad: IAM para autorización, KMS para cifrado en reposo.
- **Cómo activarlo**: 1) Importar el SDK de X-Ray en el código (Java, Python, Go, Node.js, .NET) — captura automáticamente llamadas a AWS, peticiones HTTP, BBDD, colas SQS, 2) Instalar el **demonio X-Ray** (intercepta paquetes UDP) o usar la integración nativa (Lambda ya lo trae). Necesita permisos IAM para escribir en X-Ray.

### AWS CloudTrail
- **Activado por defecto**. Da gobernanza, normativa y auditoría: historial de llamadas a la API hechas por consola, SDK, CLI o servicios AWS.
- Un trail puede cubrir todas las regiones (por defecto) o solo una.
- Los logs se pueden enviar a CloudWatch Logs o a S3.
- **Retención**: 90 días en CloudTrail. Para más tiempo, exportar a S3 y consultar con Athena.
- ⚠️ Regla de oro: **si algo desaparece en AWS sin que sepas por qué, revisa CloudTrail primero.**

### CloudTrail vs. CloudWatch vs. X-Ray (para no confundirlos)
| Servicio | Responde a | Ejemplo |
|---|---|---|
| **CloudTrail** | ¿Quién hizo qué llamada a la API y cuándo? | Detectar quién borró un bucket S3 |
| **CloudWatch** | ¿Cómo se comportan mis recursos en el tiempo? | Métricas de CPU, logs de app, alarmas |
| **X-Ray** | ¿Por dónde pasa una petición y dónde se atasca? | Latencia,errores y fallos entre microservicios |

## Comandos clave

```bash
# Crear primero el tema SNS al que avisará la alarma (email, Lambda, etc.)
aws sns create-topic --name <sns-topic-name>

# CloudWatch: crear una alarma sobre CPU de una instancia
aws cloudwatch put-metric-alarm \
  --alarm-name <alarm-name> --metric-name CPUUtilization --namespace AWS/EC2 \
  --statistic Average --period 300 --threshold 80 \
  --comparison-operator GreaterThanThreshold --evaluation-periods 2 \
  --dimensions "Name=InstanceId,Value=<instance-id>" \
  --alarm-actions "arn:aws:sns:<region>:<account-id>:<sns-topic-name>"

# Forzar el estado de la alarma manualmente, para probar la acción sin
# esperar a que la métrica real cruce el umbral
aws cloudwatch set-alarm-state \
  --alarm-name <alarm-name> --state-value ALARM \
  --state-reason "Prueba manual de la alarma"

aws cloudwatch describe-alarms

# CloudWatch Logs
aws logs create-log-group --log-group-name <log-group-name>
aws logs put-retention-policy --log-group-name <log-group-name> --retention-in-days 30
aws logs create-export-task \
  --log-group-name <log-group-name> --from 0 --to 9999999999999 \
  --destination <s3-bucket-name>

# CloudTrail
aws cloudtrail create-trail \
  --name <trail-name> --s3-bucket-name <s3-bucket-name> --is-multi-region-trail
aws cloudtrail lookup-events --max-results 10

# EventBridge: regla + destino
aws events put-rule --name <rule-name> --event-pattern '{"source":["aws.ec2"]}'
aws events put-targets \
  --rule <rule-name> \
  --targets "Id=1,Arn=arn:aws:lambda:<region>:<account-id>:function:<function-name>"

# CloudWatch Logs Insights: lanzar una query y leer el resultado
aws logs start-query \
  --log-group-name <log-group-name> --start-time 0 --end-time 9999999999 \
  --query-string "fields @timestamp, @message | sort @timestamp desc | limit 20"
aws logs get-query-results --query-id <query-id>
```

*(X-Ray se activa sobre todo integrando el SDK en el código de la app, no tanto por CLI — no hay un comando "de arranque" equivalente.)*

### CloudWatch Logs Insights — sintaxis de consulta
Lenguaje de query propio (no SQL) para buscar/agregar sobre los logs de un Log Group. Se ejecuta desde consola o vía `start-query`/`get-query-results`.

```sql
-- Ver los últimos 20 eventos, más recientes primero
fields @timestamp, @message
| sort @timestamp desc
| limit 20

-- Contar eventos por timestamp exacto (uno por evento, no agrupa)
stats count(*) by @timestamp

-- Contar eventos agrupados en bloques de 30 segundos
stats count(*) by bin(30s)
```
- `fields`: qué campos devolver (`@timestamp` y `@message` son campos por defecto de todo log de CloudWatch).
- `sort`: ordena el resultado; `desc` = más reciente primero.
- `limit`: número máximo de filas devueltas.
- `stats ... by`: agrega resultados. `by @timestamp` agrupa por el timestamp exacto de cada evento (equivale a un conteo por evento); `by bin(30s)` agrupa en franjas de tiempo — mucho más útil para ver picos de actividad a lo largo del tiempo.

## Notas y gotchas

- Ante cualquier "¿quién borró/cambió esto?", el primer sitio donde mirar es **CloudTrail**, no CloudWatch.
- La RAM de una EC2 nunca llega sola a CloudWatch — patrón que se repite en todo el curso, ver [Módulo 07](../modulo-07-ec2-avanzado/README.md).
- CloudTrail solo guarda 90 días por defecto — si necesitas histórico más largo, hay que exportar a S3 activamente, no viene solo.

## Recursos

- https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/WhatIsCloudWatch.html
- https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/WhatIsCloudWatchLogs.html
- https://docs.aws.amazon.com/xray/latest/devguide/aws-xray.html
- https://docs.aws.amazon.com/awscloudtrail/latest/userguide/cloudtrail-user-guide.html
- https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-what-is.html
