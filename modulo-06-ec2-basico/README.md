# Módulo 06 — EC2 Básico

## Resumen

**EC2 (Elastic Compute Cloud)** = IaaS, uno de los servicios más populares de AWS. Es la forma de "alquilar" servidores virtuales en el Cloud.

### Datos de usuario (User Data) / bootstrapping
- Script que se ejecuta **una sola vez**, en el **primer arranque** de la instancia, como usuario `root`.
- Sirve para automatizar: instalar actualizaciones, instalar software, descargar archivos, etc.
- Ejemplo clásico (levantar un servidor Apache "Hola Mundo"):
  ```bash
  #!/bin/bash
  yum update -y
  yum install -y httpd
  systemctl start httpd
  systemctl enable httpd
  echo "<h1>Hola Mundo desde $(hostname -f)</h1>" > /var/www/html/index.html
  ```

### Tipos de instancias EC2
| Familia | Para qué | Casos de uso |
|---|---|---|
| Uso general | Balance cómputo/memoria/red | Apps web, backend, BBDD |
| Optimizada a computación | Alto rendimiento de CPU | Batch, análisis científico, servidores web de alto tráfico |
| Optimizada a memoria | Más RAM por CPU | BBDD en memoria, apps con datasets grandes |
| Optimizada a almacenamiento | Acceso rápido a datos en disco | NoSQL, sistemas de archivos distribuidos, I/O intensiva |
| Computación acelerada | GPU/hardware especializado | Entrenamiento de IA/ML |
| Optimizada a HPC | Procesamiento paralelo, comunicación rápida entre nodos | Simulaciones científicas |

### Grupos de seguridad (Security Groups)
- **Firewall virtual** de la instancia EC2 ("el guardia de seguridad de la puerta").
- Solo contienen reglas de **permiso** (no hay reglas "deny" explícitas).
- Las reglas pueden referenciar por **IP** o por **otro grupo de seguridad**.
- Regulan: puertos, rangos de IP (IPv4/IPv6), tráfico de entrada y de salida por separado.
- Por defecto: **todo el tráfico de entrada bloqueado**, **todo el tráfico de salida permitido**.
- Se puede adjuntar a **varias** instancias a la vez. Buena práctica: un SG dedicado solo para SSH.
- Diagnóstico rápido: timeout de conexión → problema de Security Group; "connection refused" → problema de la propia aplicación (no arrancó bien).

### Conexión SSH
```bash
ssh -i /ruta/clave.pem usuario@dns-publico-instancia
```
- Puerto 22 por defecto. Windows < 10 necesita Putty; Windows ≥ 10, macOS y Linux usan SSH nativo.
- **EC2 Instance Connect**: alternativa más segura — usa una clave pública SSH temporal (expira en 60s) inyectada por AWS vía su API, en vez de mantener el puerto 22 abierto a IPs fijas.

### Errores SSH comunes
| Error | Solución |
|---|---|
| `Unprotected private key file` | Permisos del `.pem` muy abiertos → `chmod 400 clave.pem` |
| `Permission denied` / `Host key not found` | Usuario incorrecto (ej. `ec2-user` en vez de `ubuntu`, depende del SO de la AMI) |
| `Connection timed out` | Revisar: SG permite puerto 22 desde tu IP, el NACL no bloquea el tráfico, la instancia tiene IP pública, hay ruta a Internet (IGW) en la tabla de rutas, la instancia no está sobrecargada (CPU al 100%) |

### Escalado vertical
- Aumentar el tamaño de la instancia (ej. de `t2.nano` 0.5GB RAM/1 vCPU a `u-12tb1.metal` 12.3TB RAM/448 vCPUs).
- Tiene límites — para más capacidad allá de cierto punto, toca escalado horizontal (ver [[modulo-11-auto-scaling-groups]]).

### Opciones de compra de EC2
| Opción | Resumen |
|---|---|
| **Bajo demanda** | Pago por uso, sin compromiso, disponible al instante — para cargas cortas/impredecibles |
| **Planes de ahorro** (1-3 años) | Hasta 72% descuento por compromiso de uso ($/hora), flexible entre familias de instancia y SO |
| **Instancias reservadas** | Descuento por reservar 1-3 años; **Estándar** (más barata, pero no puedes cambiar familia/SO) vs **Convertible** (permite cambiar familia/SO/tenencia, y beneficiarte de bajadas de precio) |
| **Instancias Spot** | Hasta 90% descuento, pero AWS puede reclamarlas — ver [[modulo-07-ec2-avanzado]] |
| **Hosts dedicados** | Servidor físico dedicado a ti, facturación por host — la opción más cara, para compliance |
| **Instancias dedicadas** | Hardware dedicado pero facturación por instancia, sin visibilidad de sockets/núcleos |
| **Reservas de capacidad** | Reserva capacidad garantizada en una AZ concreta, sin compromiso de permanencia |

### Coste de IPv4 pública (importante, cambio reciente de AWS)
- Desde el **1 feb 2024**, AWS cobra por **cada IPv4 pública**: ~$0.005/hora (~$3.6/mes), incluso en EC2/RDS/ELB. Antes era gratis mientras la instancia estuviera activa.
- La capa gratuita de 750h/mes de EC2 no cubre esto — es un cargo aparte por la IP en sí.

## Comandos clave

```bash
# Conectar por SSH a una instancia Linux
ssh -i /path/key-pair-name.pem ec2-user@instance-public-dns-name

# Arreglar el típico "Unprotected private key file"
chmod 400 nombre-del-archivo.pem
```

## Notas y gotchas

- El **User Data solo corre una vez** (primer boot) — si necesitas ejecutar algo en cada arranque, no sirve User Data tal cual, hace falta otra estrategia (ej. script gestionado por systemd).
- El **cargo por IPv4 pública desde 2024** cambia el cálculo de coste de muchas arquitecturas — antes "gratis mientras esté encendida", ahora tiene coste por hora igual que la propia instancia.
- Diferencia clave a examen: **Security Group** = a nivel de instancia, solo reglas Allow, stateful. **NACL** = a nivel de subred, permite Allow y Deny, stateless (se ve en detalle en módulos de VPC más avanzados).

## Recursos

- https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/user-data.html
- https://aws.amazon.com/es/ec2/instance-types/
- https://aws.amazon.com/es/blogs/aws/new-aws-public-ipv4-address-charge-public-ip-insights/
