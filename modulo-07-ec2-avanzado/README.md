# Módulo 07 — EC2 Avanzado (Spot, IPs, Placement Groups, CloudWatch)

## Resumen

### Instancias Spot
- Hasta 90% descuento vs. bajo demanda; a cambio, AWS puede reclamarla con 2 min de aviso si el precio sube por encima de tu máximo.
- Solo para cargas tolerantes a fallos (batch, análisis de datos) — nunca BBDD o cargas críticas.
- **Bloqueo de Spot**: reserva sin interrupciones 1-6h.
- **Flota Spot**: varios pools (tipo/SO/AZ) hasta alcanzar capacidad objetivo dentro de un límite de precio.
- Cancelar la solicitud **no** apaga las instancias ya lanzadas — son dos pasos independientes.

### IP pública, privada, Elastic IP, ENI
- IP pública: identifica en todo internet, cambia al parar/arrancar.
- IP privada: solo dentro de tu VPC.
- **Elastic IP**: IP pública fija hasta que la liberas (5 por cuenta por defecto). Mejor evitarlas cuando se pueda — Load Balancer + DNS suele ser mejor solución.
- **ENI**: tarjeta de red virtual de la VPC, varias IPs/SG/MAC propia, se puede mover a otra instancia de la misma AZ (failover manual).

### Hibernación
- Guarda la RAM en el volumen EBS raíz (cifrado obligatorio) → arranque mucho más rápido.
- Límites: RAM <150GB, no bare metal, máx. 60 días hibernada, vale para bajo demanda/reservada/Spot.

### Placement Groups
| Estrategia | Cómo agrupa | Para qué | Punto débil |
|---|---|---|---|
| Cluster | Muy juntas, misma AZ, baja latencia | HPC, cálculo científico | Si cae la AZ, caen todas |
| Spread | Hardware físico distinto | Apps críticas | Máx. 7 por AZ |
| Partition | Particiones sin compartir rack | Big Data (Hadoop, Kafka) | Hasta 7 particiones por AZ |

### Shutdown behavior
- `Stop` (se conserva) vs `Terminate` (se destruye).
- ⚠️ Examen: si `shutdown behavior = terminate`, la protección contra terminación **no aplica** al hacer `shutdown` desde el SO.

### Troubleshooting al lanzar EC2
| Síntoma | Causa típica |
|---|---|
| Error de permisos | Falta `ec2:RunInstances` / `iam:PassRole` |
| Nombre de dispositivo inválido | Volumen duplicado o reservado (`/dev/xvda` = root) |
| Límite de instancias | Tope de vCPU por región → pedir aumento de cuota o cambiar región |
| Capacidad insuficiente | Probar otro tipo/AZ, esperar, dividir la solicitud |
| Se termina justo al lanzar | Snapshot dañado, root cifrado sin permisos KMS, AMI incompleta, límite de volúmenes EBS |

### Instancias Burstable (familia T)
- T2/T3/T3a/T4g: acumulan créditos de CPU con uso bajo, los gastan en picos.
- Créditos agotados → vuelve a rendimiento base. Si pasa constantemente, cambiar de tipo.
- Casos de uso: web con tráfico irregular, microservicios, BBDD pequeñas.
- **Modo Unlimited**: mantiene rendimiento alto sin créditos, con coste extra — ⚠️ vigilar factura.

### Métricas de CloudWatch para EC2
- Automáticas: CPU, red, disco local, estado — cada 5 min (1 min con monitorización detallada, de pago).
- Personalizadas (las mandas tú): RAM, datos de app, etc.
- ⚠️ **La RAM nunca se reporta por defecto** — de ahí el CloudWatch Agent.

### Agente unificado de CloudWatch
- Se instala en la instancia (o on-prem) para mandar métricas/logs extra.
- Requiere rol IAM; configuración centralizable vía SSM Parameter Store — ver [[modulo-08-aws-systems-manager]].
- Plugin `procstat`: monitoriza procesos concretos (CPU, memoria, hilos) por PID/exe/pattern.

### Health Checks — tema de examen
Cada minuto, automático, no desactivable. Si falla alguno → `Impaired`.

| Tipo | Detecta | Ejemplo |
|---|---|---|
| Instancia | Problema dentro de tu máquina | Red mal configurada, memoria agotada, SO colgado |
| Sistema | Problema del host físico de AWS | Fallo de hardware/energía/red del host |
| EBS adjunto | Accesibilidad/I/O de volúmenes | Volumen inaccesible |

- Falla check de **sistema** → AWS migra automáticamente a otro host.
- Falla check de **instancia** → recuperación manual.

## Comandos clave

### CloudWatch Agent (Amazon Linux 2)
```bash
# Rol IAM: CloudWatchAgentServerPolicy (+ Admin si guardas config en SSM)
sudo su
yum update -y
yum install -y httpd
echo "Bienvenido al servidor Apache con CloudWatch Agent" > /var/www/html/index.html
systemctl start httpd
systemctl enable httpd

yum install -y amazon-cloudwatch-agent

# Evitar errores si el wizard activa CollectD
sudo mkdir -p /usr/share/collectd
sudo touch /usr/share/collectd/types.db

# Asistente de configuración
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-config-wizard
# Linux, EC2 sí, resolución 60s, logs de access_log/error_log, guardar en SSM: sí

# Arrancar con config desde SSM
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
  -a fetch-config -m ec2 -c ssm:AmazonCloudWatch-linux -s

# ...o desde archivo local
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
  -a fetch-config -m ec2 -c file:/opt/aws/amazon-cloudwatch-agent/bin/config.json -s
```
Verificar: CloudWatch → Log groups (`access_log`, `error_log`) y Metrics → `CWAgent`.

### Mini ETL en instancia Spot (Python + boto3)
```bash
pip3 install boto3
```
```python
import boto3

S3_BUCKET = "demoetlspotec2instance"
ARCHIVO_ORIGEN = "input_data.txt"
ARCHIVO_DESTINO = "output_data.txt"

s3 = boto3.client('s3')

with open(ARCHIVO_ORIGEN, 'r') as archivo:
    datos = archivo.readlines()

datos_transformados = [linea.upper() for linea in datos]
with open(ARCHIVO_DESTINO, 'w') as archivo:
    archivo.writelines(datos_transformados)

s3.upload_file(ARCHIVO_DESTINO, S3_BUCKET, ARCHIVO_DESTINO)
```
Patrón: batch job corto y reanudable, ideal para correr sobre una Spot Fleet.

## Notas y gotchas

- Antes de usar una Elastic IP, piénsalo dos veces — casi siempre hay mejor alternativa (LB + DNS).
- Monitorización estándar (5 min, gratis) vs detallada (1 min, de pago) — la detallada importa para que el ASG reaccione rápido.
- La RAM nunca se reporta por defecto a CloudWatch — de ahí que el agente sea casi obligatorio si necesitas vigilar memoria.

## Recursos

- https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/spot-requests.html
- https://aws.amazon.com/es/ec2/faqs/
