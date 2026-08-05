# Módulo 07 — EC2 Avanzado (Spot, IPs, Placement Groups, CloudWatch Agent)

## Resumen

### Instancias Spot
- Hasta **90% de descuento** frente a on-demand, a cambio de que AWS puede reclamar la instancia en cualquier momento (2 min de gracia) si el precio spot supera tu precio máximo.
- Ideales para cargas tolerantes a fallos: batch jobs, análisis de datos, workloads con hora de inicio/fin flexible. **No usar** para bases de datos o cargas críticas.
- **Bloqueo de Spot**: reserva la instancia sin interrupciones durante 1–6h (raro que se reclame).
- **Flota Spot (Spot Fleet)**: conjunto de instancias Spot (+ opcionalmente on-demand) que intenta alcanzar una capacidad objetivo dentro de restricciones de precio, usando varios pools de lanzamiento (tipo de instancia, SO, AZ).
- Terminar una solicitud Spot son **dos pasos**: cancelar la solicitud y luego terminar manualmente las instancias asociadas (cancelar no termina las instancias).

### IP pública vs. privada / Elastic IP / ENI
- IP pública: única en toda internet, identificable, cambia si paras/arrancas la instancia.
- IP privada: única solo dentro de la red privada (VPC), no accesible desde fuera (necesita NAT).
- **Elastic IP**: IP pública fija que te pertenece hasta que la liberas. Máximo 5 por cuenta (ampliable). AWS recomienda evitarlas: mejor un Load Balancer o un nombre DNS.
- **ENI (Elastic Network Interface)**: tarjeta de red virtual de una VPC, puede tener varias IPs, grupos de seguridad, dirección MAC propia. Se puede desconectar y mover a otra instancia de la misma AZ (failover).

### Hibernación de EC2
- Conserva el estado de la RAM (se escribe en el volumen EBS raíz, que debe estar **cifrado**) → arranque mucho más rápido que un boot normal.
- Límite: RAM < 150 GB, no bare metal, máx. 60 días hibernada, disponible en on-demand/reservadas/spot.

### Placement Groups
| Estrategia | Qué hace | Caso de uso | Contras |
|---|---|---|---|
| **Cluster** | Instancias muy juntas, misma AZ, baja latencia (hasta 10 Gbps) | Simulación científica, HPC | Si cae la AZ, caen todas |
| **Spread** | Instancias en hardware físico distinto | Apps críticas | Máx. 7 instancias por AZ |
| **Partition** | Grupos de instancias en particiones aisladas (no comparten rack) | Big Data (Hadoop, Kafka, Cassandra) | Hasta 7 particiones por AZ |

### Shutdown behavior y protección
- `Stop` (se conserva) vs `Terminate` (se elimina) al apagar desde SO.
- ⚠️ **Ojo examen**: si `shutdown behavior = terminate`, la protección contra terminación **no aplica** — la instancia se borra igual al hacer `shutdown` desde dentro del SO.

### Troubleshooting típico al lanzar EC2
- **Permisos insuficientes** → faltan `ec2:RunInstances` / `iam:PassRole`.
- **Invalid device name** → nombre de volumen duplicado o reservado (`/dev/xvda` es el root).
- **InstanceLimitExceeded** → límite de vCPU por región (solo on-demand/spot) → pedir aumento de quota o cambiar de región.
- **InsufficientInstanceCapacity** → esperar, dividir la solicitud, probar otro tipo de instancia o cambiar de AZ.
- **La instancia termina inmediatamente** → snapshot EBS dañado, volumen root cifrado sin permisos KMS, AMI incompleta, o límite de volúmenes EBS alcanzado.

### CloudWatch Agent (demo práctica)
Agente unificado que se instala **dentro** de la instancia para mandar métricas que CloudWatch no trae por defecto (RAM, uso de disco detallado) y logs de aplicación (Apache, etc.) a CloudWatch Logs.

## Comandos clave

### CloudWatch Agent — instalación y configuración (Amazon Linux 2)
```bash
# Rol IAM necesario en la instancia: CloudWatchAgentServerPolicy
# (+ opcional CloudWatchAgentAdminPolicy para guardar la config en SSM)

sudo su
yum update -y
yum install -y httpd
echo "Bienvenido al servidor Apache con CloudWatch Agent" > /var/www/html/index.html
systemctl start httpd
systemctl enable httpd

# Instalar el agente
yum install -y amazon-cloudwatch-agent

# Evitar errores si se activa CollectD en el wizard
sudo mkdir -p /usr/share/collectd
sudo touch /usr/share/collectd/types.db

# Asistente de configuración interactivo
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-config-wizard
# Respuestas típicas: Linux, EC2 sí, resolución 60s, logs de
# /var/log/httpd/access_log y /var/log/httpd/error_log, guardar en SSM: sí

# Arrancar el agente con la config guardada en SSM Parameter Store
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
  -a fetch-config -m ec2 -c ssm:AmazonCloudWatch-linux -s

# ...o con un archivo local
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
  -a fetch-config -m ec2 -c file:/opt/aws/amazon-cloudwatch-agent/bin/config.json -s
```
Verificación: CloudWatch → Log groups (`access_log`, `error_log`) y CloudWatch → Metrics → `CWAgent`.

### Mini ETL en una instancia Spot (Python + boto3)
```bash
pip3 install boto3
```
```python
import boto3

S3_BUCKET = "demoetlspotec2instance"
ARCHIVO_ORIGEN = "input_data.txt"
ARCHIVO_DESTINO = "output_data.txt"

s3 = boto3.client('s3')

# Extract
with open(ARCHIVO_ORIGEN, 'r') as archivo:
    datos = archivo.readlines()

# Transform
datos_transformados = [linea.upper() for linea in datos]
with open(ARCHIVO_DESTINO, 'w') as archivo:
    archivo.writelines(datos_transformados)

# Load
s3.upload_file(ARCHIVO_DESTINO, S3_BUCKET, ARCHIVO_DESTINO)
```
Patrón típico de uso: lanzar una Spot Fleet que ejecuta este ETL como *batch job* — si AWS reclama la instancia a mitad de proceso, no pasa nada crítico porque el trabajo es reanudable.

## Notas y gotchas

- Las Elastic IP "suelen reflejar malas decisiones de arquitectura" (cita textual del curso) — prioriza Load Balancer + DNS.
- La monitorización estándar de EC2 es cada 5 min; la detallada (de pago, con capa gratuita de 10 métricas) es cada 1 min — necesaria si quieres que el ASG escale rápido.
- El uso de memoria de EC2 **no se envía por defecto** a CloudWatch — hace falta el CloudWatch Agent para eso (de ahí la demo).

## Recursos

- https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/spot-requests.html
- https://aws.amazon.com/es/ec2/faqs/
