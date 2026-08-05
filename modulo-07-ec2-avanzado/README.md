# Módulo 07 — EC2 Avanzado (Spot, IPs, Placement Groups, CloudWatch)

## Resumen

### Instancias Spot: la opción más barata, con truco
AWS vende su capacidad sobrante como instancias Spot, con descuentos que pueden llegar al 90% frente al precio bajo demanda. La contrapartida es que si el precio de mercado del Spot sube por encima de lo que estás dispuesto a pagar, AWS te la puede quitar con solo dos minutos de aviso. Por eso solo tienen sentido para cargas que aguantan interrupciones sin drama: procesos batch, análisis de datos, cualquier tarea con margen de tiempo — nunca para una base de datos o algo que no puedas permitirte perder a mitad de ejecución.

Si necesitas más garantía de continuidad sin pagar precio completo, existe el **bloqueo de Spot**, que reserva la instancia sin interrupciones durante una ventana de 1 a 6 horas. Y para desplegar varias instancias Spot a la vez repartiendo el riesgo, se usa una **Flota Spot**: defines varios "pools" posibles (combinaciones de tipo de instancia, sistema operativo y AZ) y AWS va lanzando instancias de esos pools hasta alcanzar la capacidad que pediste, dentro del límite de precio que fijaste.

Un detalle que suele pillar a la gente: cancelar una solicitud Spot **no** apaga las instancias que ya se lanzaron a partir de ella — hay que terminarlas manualmente aparte, son dos acciones independientes.

### Direccionamiento: IP pública, privada, Elastic IP y ENI
- La **IP pública** identifica la instancia en todo internet, pero no es fija: si paras y vuelves a arrancar la instancia, cambia.
- La **IP privada** solo tiene sentido dentro de tu propia VPC — nadie desde fuera puede alcanzarla directamente.
- Si necesitas que la IP pública **no cambie nunca**, existe la **Elastic IP**: una IP fija que reservas y controlas tú, hasta que decides liberarla (el límite por defecto es de 5 por cuenta, ampliable). En la práctica se recomienda evitarlas cuando se pueda — normalmente hay una solución mejor apoyándote en un Load Balancer con su propio nombre DNS, que no ata tu arquitectura a una IP concreta.
- La **ENI (Elastic Network Interface)** es la tarjeta de red virtual de una instancia dentro de la VPC: puede llevar varias IPs, sus propios Security Groups y su propia MAC. Lo interesante es que se puede desconectar de una instancia y engancharla a otra de la misma AZ en segundos — útil como mecanismo de failover manual.

### Hibernación: arrancar rápido conservando el estado
En vez de un arranque en frío normal, la hibernación guarda el contenido de la RAM en el volumen EBS raíz (que debe estar cifrado) y lo restaura al reactivar la instancia, evitando tener que recalentar cachés o reiniciar procesos desde cero. Tiene condiciones: menos de 150GB de RAM, no vale para bare metal, y un máximo de 60 días hibernada — pero funciona tanto en instancias bajo demanda como reservadas o Spot.

### Placement Groups: dónde coloca AWS tus instancias físicamente
| Estrategia | Cómo agrupa las instancias | Para qué sirve | Su punto débil |
|---|---|---|---|
| **Cluster** | Muy juntas, en la misma AZ, con red de muy baja latencia | Cálculo científico, cargas que necesitan comunicación rápida entre nodos | Si cae esa AZ, se caen todas a la vez |
| **Spread** | Repartidas en hardware físico distinto | Aplicaciones críticas que no pueden compartir punto único de fallo | Tope de 7 instancias por AZ |
| **Partition** | Agrupadas en particiones que no comparten rack entre sí | Sistemas de Big Data (Hadoop, Kafka, Cassandra) | Hasta 7 particiones por AZ |

### Qué pasa al apagar una instancia desde dentro
El comportamiento de apagado se puede configurar como `Stop` (se conserva, se puede reanudar) o `Terminate` (se destruye). El detalle que suele salir en examen: si configuraste `Terminate`, **la protección contra terminación deja de aplicar** en cuanto haces `shutdown` desde dentro del sistema operativo — la instancia se borra igualmente, aunque tuvieras esa protección activada para borrados desde la consola.

### Cuando el lanzamiento de una EC2 falla
| Síntoma | Qué suele ser |
|---|---|
| Error de permisos | Faltan acciones IAM como `ec2:RunInstances` o `iam:PassRole` |
| Nombre de dispositivo inválido | Un volumen con nombre repetido o usando uno reservado (`/dev/xvda` es siempre el root) |
| Límite de instancias superado | Tope de vCPU de la región (afecta a bajo demanda y Spot) → pedir aumento de cuota o cambiar de región |
| Capacidad insuficiente | Probar otro tipo de instancia, otra AZ, esperar, o dividir la solicitud en varias más pequeñas |
| La instancia se termina justo después de lanzarse | Puede ser un snapshot EBS corrupto, el volumen raíz cifrado sin permisos KMS suficientes, una AMI incompleta, o haber llegado al límite de volúmenes EBS |

### Instancias con rendimiento de ráfaga (familia T)
Las instancias `T` (T2, T3, T3a, T4g...) están pensadas para cargas que no necesitan CPU al máximo todo el rato: cuando usan poca CPU van acumulando "créditos", y cuando llega un pico de demanda, gastan esos créditos para rendir por encima de su nivel base. Si el pico se prolonga y se agotan los créditos, la instancia cae de vuelta a su rendimiento base — y si eso ocurre constantemente, es señal de que ese tipo de instancia se ha quedado pequeño para la carga real. Encajan bien en servidores web con tráfico irregular, microservicios o bases de datos pequeñas. Existe también un modo **Unlimited**, que permite mantener el rendimiento alto incluso sin créditos, a cambio de un coste extra — cómodo para picos impredecibles, pero hay que vigilarlo porque puede disparar la factura si no lo monitorizas.

### Qué mide CloudWatch de una instancia EC2 (y qué no)
Por defecto, AWS ya te manda automáticamente métricas de CPU, red, disco local y estado de la instancia, actualizadas cada 5 minutos (o cada minuto si pagas monitorización detallada). Si necesitas algo que no viene incluido —el ejemplo más habitual es el **uso de memoria RAM, que AWS nunca reporta por defecto**— tienes que mandarlo tú mismo como métrica personalizada, con los permisos IAM adecuados. Ahí es donde entra el CloudWatch Agent.

### El agente unificado de CloudWatch
Es un programa que instalas dentro de la propia instancia (o incluso en un servidor fuera de AWS) para enviar a CloudWatch justo esas métricas y logs que no llegan por defecto. Necesita un rol IAM con permisos, y su configuración se puede centralizar a través de SSM Parameter Store — ver [[modulo-08-aws-systems-manager]]. Trae además un complemento llamado `procstat` para vigilar procesos concretos (uso de CPU, memoria, número de hilos), identificando el proceso por su PID, su ruta de ejecutable o un patrón de texto.

### Comprobaciones de estado (Health Checks) — tema recurrente en examen
AWS ejecuta automáticamente, cada minuto y sin posibilidad de desactivarlo, tres tipos de comprobación sobre cada instancia:

| Tipo de check | Detecta problemas en... | Ejemplo |
|---|---|---|
| De instancia | Tu propia instancia | Red mal configurada, memoria agotada, SO colgado |
| De sistema | El host físico de AWS por debajo | Fallo de hardware, de energía o de red del propio host |
| De EBS adjunto | Los volúmenes conectados | Volumen inaccesible o con errores de I/O |

La diferencia importante entre las dos primeras: si falla el check de **sistema**, AWS puede migrar automáticamente tu instancia a otro host físico sin que tengas que hacer nada; si falla el check de **instancia**, el problema está dentro de tu propia máquina y la recuperación es manual (reiniciar o detener/arrancar tú mismo). Se pueden enganchar alarmas de CloudWatch a cualquiera de los tres resultados.

## Comandos clave

### Instalar y configurar el CloudWatch Agent (Amazon Linux 2)
```bash
# La instancia necesita un rol IAM con la política CloudWatchAgentServerPolicy
# (y opcionalmente CloudWatchAgentAdminPolicy si vas a guardar la config en SSM)

sudo su
yum update -y
yum install -y httpd
echo "Bienvenido al servidor Apache con CloudWatch Agent" > /var/www/html/index.html
systemctl start httpd
systemctl enable httpd

# Instalar el agente
yum install -y amazon-cloudwatch-agent

# Evitar errores si el wizard activa CollectD
sudo mkdir -p /usr/share/collectd
sudo touch /usr/share/collectd/types.db

# Asistente de configuración interactivo
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-config-wizard
# Respuestas típicas: Linux, EC2 sí, resolución 60s, logs de
# /var/log/httpd/access_log y /var/log/httpd/error_log, guardar en SSM: sí

# Arrancar el agente con la configuración guardada en SSM Parameter Store
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
  -a fetch-config -m ec2 -c ssm:AmazonCloudWatch-linux -s

# ...o directamente desde un archivo local
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
  -a fetch-config -m ec2 -c file:/opt/aws/amazon-cloudwatch-agent/bin/config.json -s
```
Para comprobar que funciona: en CloudWatch → Log groups deberían aparecer `access_log` y `error_log`, y en CloudWatch → Metrics debería aparecer el namespace `CWAgent`.

### Mini ETL corriendo en una instancia Spot (Python + boto3)
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
La idea de fondo de este ejemplo: es exactamente el tipo de tarea que tiene sentido lanzar sobre una Spot Fleet — si AWS reclama la instancia a mitad de proceso, el trabajo se puede relanzar sin pérdida real, porque es un batch job corto y reanudable.

## Notas y gotchas

- Antes de usar una Elastic IP, piénsalo dos veces: casi siempre hay una alternativa mejor (Load Balancer + DNS) que no ata tu arquitectura a una dirección fija que hay que gestionar a mano.
- Diferencia clave entre monitorización estándar (cada 5 min, gratis) y detallada (cada 1 min, de pago con cierta capa gratuita) — la detallada importa si quieres que tu Auto Scaling Group reaccione rápido a picos de carga.
- Vuelve a subrayarse aquí: **la RAM nunca se reporta por defecto** a CloudWatch, de ahí que la demo del agente sea prácticamente obligatoria si necesitas vigilar memoria.

## Recursos

- https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/spot-requests.html
- https://aws.amazon.com/es/ec2/faqs/
