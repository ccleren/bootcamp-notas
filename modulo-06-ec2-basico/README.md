# Módulo 06 — EC2 Básico

## Resumen

**EC2 (Elastic Compute Cloud)** = servicio IaaS para "alquilar" servidores virtuales, el más usado de AWS.

### User Data (bootstrapping)
- Script que corre **una sola vez**, en el **primer arranque**, como `root`.
- Sirve para instalar/actualizar/configurar sin conectarte manualmente después.
```bash
#!/bin/bash
yum update -y
yum install -y httpd
systemctl start httpd
systemctl enable httpd
echo "<h1>Hola Mundo desde $(hostname -f)</h1>" > /var/www/html/index.html
```

### Tipos de instancia
| Familia | Para qué | Casos de uso |
|---|---|---|
| Uso general | Balance CPU/RAM/red | Apps web, backend, BBDD pequeñas |
| Optimizada a computación | CPU potente | Batch, cálculo científico |
| Optimizada a memoria | Más RAM por CPU | BBDD en memoria, datasets grandes |
| Optimizada a almacenamiento | I/O rápida en disco | NoSQL, sistemas distribuidos |
| Computación acelerada | GPU | Entrenamiento IA/ML |
| Optimizada a HPC | Procesamiento paralelo | Simulaciones científicas |

### Security Groups
- Firewall a nivel de instancia. **Solo reglas de permiso** (no hay "deny" explícito) — lo no permitido queda bloqueado.
- Reglas separadas para entrada/salida, por IP o por otro SG.
- Por defecto: toda entrada bloqueada, toda salida permitida.
- Un mismo SG se reutiliza en varias instancias — buena práctica: un SG dedicado solo a SSH.
- Diagnóstico rápido: timeout → problema de SG; "connection refused" → la app no arrancó bien.

### Conexión SSH
```bash
ssh -i /ruta/clave.pem usuario@dns-publico-instancia
```
- Puerto 22. Windows ≥10/macOS/Linux usan SSH nativo; Windows <10 necesita Putty.
- **EC2 Instance Connect**: alternativa más segura — clave pública temporal (60s) inyectada por la API de AWS, sin mantener el 22 abierto permanentemente.

### Errores SSH típicos
| Error | Causa |
|---|---|
| `Unprotected private key file` | Permisos del `.pem` muy abiertos → `chmod 400 clave.pem` |
| `Permission denied` / `Host key not found` | Usuario incorrecto según el SO de la AMI |
| `Connection timed out` | SG no permite el 22 desde tu IP, NACL bloqueando, sin IP pública, falta ruta a internet, o CPU al 100% |

### Escalado vertical
Subir el tamaño de la instancia — tiene techo. Más allá, toca escalado horizontal ([Módulo 11 — Auto Scaling Groups](../modulo-11-auto-scaling-groups/README.md)).

### Opciones de compra
| Opción | Resumen |
|---|---|
| Bajo demanda | Pago por uso, sin compromiso, al instante |
| Planes de ahorro | Hasta ~70% descuento por compromiso $/hora, flexible entre familias |
| Reservadas | Descuento 1-3 años; Estándar (rígida, más barata) vs Convertible (flexible) |
| Spot | Hasta 90% descuento, AWS puede reclamarlas — ver [Módulo 07 — EC2 Avanzado](../modulo-07-ec2-avanzado/README.md) |
| Hosts dedicados | Servidor físico entero para ti, la más cara, para compliance |
| Instancias dedicadas | Hardware no compartido, sin visibilidad de socket/núcleo |
| Reservas de capacidad | Capacidad garantizada en una AZ, sin compromiso de uso |

### Cargo por IPv4 pública (desde feb 2024)
AWS cobra por cada IP pública asignada (~$0.005/h), aparte del coste de la instancia. Antes era gratis mientras la instancia estuviera encendida.

## Comandos clave

```bash
ssh -i /path/key-pair-name.pem ec2-user@instance-public-dns-name
chmod 400 nombre-del-archivo.pem
```

Crear el Security Group y el par de claves antes de lanzar nada:
```bash
# Crear el Security Group
aws ec2 create-security-group \
  --group-name MiSecurityGroup --description "Mi security group" --vpc-id <vpc-id>

# Abrir el puerto 22 (SSH) solo desde tu IP, y el 80 (HTTP) a todo el mundo
aws ec2 authorize-security-group-ingress \
  --group-id <security-group-id> --protocol tcp --port 22 --cidr <tu-ip>/32
aws ec2 authorize-security-group-ingress \
  --group-id <security-group-id> --protocol tcp --port 80 --cidr 0.0.0.0/0

# Crear un par de claves (guarda el .pem que devuelve, no se puede recuperar después)
aws ec2 create-key-pair --key-name mi-clave --query "KeyMaterial" --output text > mi-clave.pem
```

Gestionar instancias por CLI (alternativa a la consola):
```bash
# Lanzar una instancia con User Data
aws ec2 run-instances \
  --image-id <ami-id> --instance-type t2.micro --count 1 \
  --key-name <key-pair-name> --security-group-ids <security-group-id> \
  --subnet-id <subnet-id> --associate-public-ip-address \
  --user-data file://user-data.sh

# Etiquetar la instancia recién creada
aws ec2 create-tags --resources <instance-id> --tags Key=Name,Value="Servidor Web 1"

# Ver el estado de las instancias
aws ec2 describe-instances --query "Reservations[].Instances[].[InstanceId,State.Name]"

# Parar / terminar (admite varios IDs a la vez)
aws ec2 stop-instances --instance-ids <instance-id>
aws ec2 terminate-instances --instance-ids <instance-id> <instance-id-2>
```

Limpieza al terminar (evita costes olvidados):
```bash
aws ec2 delete-security-group --group-id <security-group-id>
```

## Notas y gotchas

- User Data solo corre en el primer arranque — para repetir en cada reinicio hace falta otra estrategia (ej. servicio systemd).
- El cargo por IPv4 pública cambia el cálculo de coste de arquitecturas antes "gratis mientras encendidas".
- Examen: **Security Group** = nivel instancia, solo Allow, *stateful*. **NACL** = nivel subred, Allow y Deny, *stateless*.

## Recursos

- https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/user-data.html
- https://aws.amazon.com/es/ec2/instance-types/
- https://aws.amazon.com/es/blogs/aws/new-aws-public-ipv4-address-charge-public-ip-insights/
