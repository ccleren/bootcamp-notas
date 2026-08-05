# Módulo 06 — EC2 Básico

## Resumen

**EC2 (Elastic Compute Cloud)** es el servicio de AWS para "alquilar" servidores virtuales — la pieza IaaS más usada de todo el catálogo. Es el punto de partida para entender casi todo lo demás en AWS, porque muchos otros servicios acaban desplegándose sobre EC2 por debajo.

### User Data: automatizar el primer arranque
Al lanzar una instancia puedes pasarle un script que se ejecuta automáticamente, como usuario `root`, **solo en el primer arranque**. Es la forma estándar de dejar la máquina lista sin tener que conectarte manualmente después: instalar paquetes, actualizar el sistema, descargar configuración, etc. Un ejemplo mínimo para levantar un Apache de prueba nada más arrancar:
```bash
#!/bin/bash
yum update -y
yum install -y httpd
systemctl start httpd
systemctl enable httpd
echo "<h1>Hola Mundo desde $(hostname -f)</h1>" > /var/www/html/index.html
```

### Elegir el tipo de instancia correcto
| Familia | Está pensada para | Ejemplos de uso |
|---|---|---|
| Uso general | Un equilibrio entre CPU, memoria y red | Apps web típicas, backends, bases de datos pequeñas |
| Optimizada a computación | CPU potente por encima de todo | Procesos batch, cálculo científico, servidores con mucho tráfico |
| Optimizada a memoria | Mucha RAM en proporción a la CPU | Bases de datos en memoria, datasets grandes |
| Optimizada a almacenamiento | Lectura/escritura rápida en disco | NoSQL, sistemas de archivos distribuidos |
| Computación acelerada | GPU u otro hardware especializado | Entrenamiento de IA/ML |
| Optimizada a HPC | Procesamiento paralelo entre muchos nodos | Simulaciones científicas a gran escala |

### Grupos de seguridad
Cada instancia EC2 lleva asociado uno o varios **Security Groups**, que funcionan como un firewall a nivel de instancia. Una particularidad importante: **solo existen reglas de permiso**, no hay forma de "denegar" explícitamente algo dentro de un SG — todo lo que no está permitido, queda bloqueado por omisión. Puedes definir reglas separadas para tráfico entrante y saliente, referenciando tanto rangos de IP como otro Security Group. Por defecto, una instancia recién creada bloquea toda entrada y permite toda salida.

Un mismo Security Group se puede reutilizar en varias instancias a la vez — de hecho es buena práctica tener uno dedicado solo a SSH en vez de mezclarlo con las reglas de la aplicación. Para diagnosticar problemas de conexión rápido: si la conexión hace timeout, sospecha primero del Security Group; si en cambio da "connection refused", el problema suele estar en que la aplicación no llegó a arrancar bien dentro de la instancia.

### Conectarte por SSH
```bash
ssh -i /ruta/clave.pem usuario@dns-publico-instancia
```
El puerto por defecto es el 22. En Windows 10 o superior, macOS y Linux funciona el cliente SSH nativo; en versiones antiguas de Windows hace falta una herramienta como Putty. Como alternativa más segura existe **EC2 Instance Connect**, que evita mantener el puerto 22 permanentemente abierto: AWS inyecta una clave pública temporal (válida solo 60 segundos) a través de su propia API en el momento de conectarte.

### Errores típicos al conectar por SSH
| Error | Qué lo suele causar |
|---|---|
| `Unprotected private key file` | El archivo `.pem` tiene permisos demasiado abiertos → arreglar con `chmod 400 clave.pem` |
| `Permission denied` / `Host key not found` | Usuario incorrecto para el SO de la AMI (ej. usar `ubuntu` cuando debía ser `ec2-user`) |
| `Connection timed out` | Puede ser varias cosas a la vez: el Security Group no deja pasar el puerto 22 desde tu IP, un NACL está bloqueando el tráfico, la instancia no tiene IP pública, falta una ruta a internet en la tabla de rutas, o la instancia está saturada de CPU |

### Cuando el tamaño de la instancia se queda corto
Subir de tamaño una instancia (por ejemplo de una `t2.nano` con 0.5GB de RAM a algo con varios TB de RAM) es escalado vertical, y tiene un techo — llega un punto en el que hace falta repartir la carga entre varias instancias en paralelo (escalado horizontal), tema que se cubre en [[modulo-11-auto-scaling-groups]].

### Formas de pagar una instancia EC2
| Opción | En qué consiste |
|---|---|
| **Bajo demanda** | Pagas por hora/segundo de uso, sin compromiso, disponible al instante |
| **Planes de ahorro** | Descuento considerable (hasta ~70%) a cambio de comprometerte a un gasto/hora durante 1-3 años, con flexibilidad para cambiar de familia de instancia |
| **Instancias reservadas** | Descuento por reservar 1-3 años de uso; la variante **Estándar** es más barata pero rígida, la **Convertible** permite cambiar de familia/SO a cambio de menos descuento |
| **Instancias Spot** | El mayor descuento posible (hasta 90%), a cambio de que AWS pueda recuperarlas cuando las necesite — detalle en [[modulo-07-ec2-avanzado]] |
| **Hosts dedicados** | Servidor físico entero solo para ti, la opción más cara, pensada para requisitos de cumplimiento normativo |
| **Instancias dedicadas** | Hardware no compartido con otras cuentas, pero sin la visibilidad a nivel de socket/núcleo que sí da un host dedicado |
| **Reservas de capacidad** | Garantiza que tendrás capacidad disponible en una AZ concreta, sin comprometerte a usarla todo el tiempo |

### El cargo por IPv4 pública (cambio reciente que conviene tener presente)
Desde febrero de 2024, AWS cobra por cada dirección IPv4 pública que tengas asignada (a una EC2, RDS, un Load Balancer...), independientemente de si esa capa gratuita de cómputo te cubre la instancia en sí. Antes de ese cambio, la IP pública no tenía coste propio mientras la instancia estuviera encendida — ahora sí, y es un cargo aparte del de la propia instancia.

## Comandos clave

```bash
# Conectar por SSH a una instancia Linux
ssh -i /path/key-pair-name.pem ec2-user@instance-public-dns-name

# Arreglar el típico "Unprotected private key file"
chmod 400 nombre-del-archivo.pem
```

## Notas y gotchas

- El User Data solo se ejecuta en el **primer arranque** — si necesitas que algo corra en cada reinicio, hace falta otra estrategia (por ejemplo, un servicio gestionado por systemd), no vale confiar en que el script se repita solo.
- El nuevo cargo por IPv4 pública cambia los números de coste de arquitecturas que antes se consideraban "gratis mientras estuvieran encendidas".
- Diferencia clásica de examen: un **Security Group** actúa a nivel de instancia, solo tiene reglas de permitir, y es *stateful* (si dejas entrar el tráfico, la respuesta sale sola). Un **NACL** actúa a nivel de subred, admite tanto permitir como denegar, y es *stateless* — se detalla más en módulos de VPC avanzados.

## Recursos

- https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/user-data.html
- https://aws.amazon.com/es/ec2/instance-types/
- https://aws.amazon.com/es/blogs/aws/new-aws-public-ipv4-address-charge-public-ip-insights/
