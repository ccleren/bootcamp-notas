# Módulo 09 — Almacenamiento de EC2 (AMI, EBS, Instance Store, EFS)

## Resumen

### Amazon Machine Image (AMI)
- "Foto" completa de una instancia (SO + software + config) para lanzar instancias idénticas.
- Fuentes: AMI pública de AWS, tu propia AMI, o AWS Marketplace.
- Ligada a una región, copiable a otras. Se puede dar de baja (deregister) cuando no se necesita.

### EC2 Image Builder
- Automatiza crear/mantener/probar/distribuir AMIs.
- Flujo: definir componentes → pruebas automáticas → distribuir (incluso multi-región).
- Programable (ej. semanal). Servicio gratis, solo pagas los recursos que consume al construir.

### Amazon EBS
- Almacenamiento en bloques, se adjunta a una instancia mientras está encendida, persiste tras terminarla (salvo que configures lo contrario en el volumen raíz).
- Recurso de red (algo de latencia), pero se puede desconectar/reconectar rápido a otra instancia.
- **Atado a una única AZ** — moverlo a otra AZ exige snapshot + restaurar.
- Se paga toda la capacidad reservada, se use o no.

| Tipo | Para qué | ¿Vale como boot volume? |
|---|---|---|
| gp2/gp3 (SSD) | Uso general | Sí |
| io1/io2 Block Express (SSD) | Máximo rendimiento/mínima latencia | Sí |
| st1 (HDD) | Acceso frecuente, bajo coste | No |
| sc1 (HDD) | El más barato, acceso poco frecuente | No |

- **Multi-Attach** (solo io1/io2): mismo volumen en hasta 16 instancias de la misma AZ, lectura/escritura completa.
- Redimensionar: solo se puede **aumentar**, nunca reducir directamente (para reducir: crear uno nuevo + migrar datos).

**Snapshots**: backup de un volumen en un momento dado, copiable entre AZ/región. Archivado ~75% más barato (restauración 24-72h). Papelera de reciclaje para recuperarlos si se borran por error. **Fast Snapshot Restore**: precarga el volumen para rendimiento inmediato, con coste por minuto activo.

**Amazon DLM**: automatiza creación/retención/borrado de snapshots y AMIs por tags. Solo gestiona lo creado por él mismo — no toca snapshots manuales ni Instance Store.

### Instance Store
Almacenamiento físico conectado al host — muy rápido, pero **no persistente**: se pierde todo al parar/terminar la instancia. Solo para cachés temporales/buffers.

### Amazon EFS
- Sistema de archivos NFS gestionado, montable en muchas instancias EC2 a la vez, multi-AZ.
- Caso de uso típico: varios servidores compartiendo los mismos archivos (ej. contenido de un WordPress tras un Load Balancer).
- Solo Linux (POSIX). Escala automáticamente. Más caro que EBS.

| | EBS | EFS |
|---|---|---|
| Se adjunta a | 1 instancia (hasta 16 con Multi-Attach) | Miles simultáneamente |
| Alcance | Una AZ | Varias AZ |
| SO | Linux y Windows | Solo Linux |
| Coste | Más bajo | Más alto |
| Gestión de infraestructura | Tuya | De AWS |

- **Clases de almacenamiento**: Standard (frecuente, multi-AZ), IA (infrecuente, más barato con coste al leer), One Zone/One Zone-IA (una AZ, dev/backup). Ahorro >90% moviendo datos fríos a IA con política de ciclo de vida.
- **Access Points**: acceso a una ruta concreta (ej. `/datasets`) con UID/GID fijo y permisos IAM propios — evita exponer la raíz completa.
- Operaciones sin migración: cambiar lifecycle, crear Access Points. Con migración (vía **AWS DataSync**): cifrar, cambiar a modo Max I/O.
- Métricas CloudWatch: `PercentIOLimit` (al 100%, sin margen sin migrar a Max I/O), `BurstCreditBalance` (a 0, rendimiento cae), `StorageBytes` (cada 15 min).

## Comandos clave

### EBS por CLI
```bash
# Crear un volumen gp3 de 50GB en una AZ concreta
aws ec2 create-volume --volume-type gp3 --size 50 --availability-zone eu-west-1a

# Adjuntarlo a una instancia
aws ec2 attach-volume \
  --volume-id vol-xxxxxxxx --instance-id i-xxxxxxxxxxxxxxxxx --device /dev/sdf

# Crear un snapshot del volumen
aws ec2 create-snapshot --volume-id vol-xxxxxxxx --description "backup manual"

# Aumentar tamaño/IOPS de un volumen existente (nunca reducir)
aws ec2 modify-volume --volume-id vol-xxxxxxxx --size 100

# Listar volúmenes y snapshots
aws ec2 describe-volumes
aws ec2 describe-snapshots --owner-ids self
```

### Montar un EFS en dos instancias EC2 (Amazon Linux, `dnf`)

```bash
# En cada instancia
df -k
sudo dnf -y install amazon-efs-utils
sudo mkdir -p /efs/wp-content

# fstab (montaje persistente)
sudo vi /etc/fstab
# file-system-id:/ /efs/wp-content efs _netdev 0 0

sudo mount /efs/wp-content
df -k

# Prueba de que es compartido
cd /efs/wp-content
sudo touch demoefs.txt
```
- `file-system-id`: el ID del EFS (ej. `fs-0123456789abcdef0`).
- `_netdev`: espera a que la red esté lista antes de montar.
- Repetir `mkdir` + `fstab` + `mount` en **cada** instancia que comparta el filesystem.

## Notas y gotchas

- Examen: **EBS atado a una AZ, EFS no**.
- EBS no se puede reducir de tamaño, solo crecer.
- Instance Store pierde todo al parar/terminar — nunca para datos que no puedas perder.
- Si EFS no comparte archivos entre instancias, revisa el Security Group del EFS (NFS/puerto 2049) antes que el comando de montaje.
- DLM no gestiona nada creado manualmente fuera de él, ni Instance Store.

## Recursos

- https://aws.amazon.com/es/blogs/storage/making-it-even-simpler-to-get-started-with-amazon-efs/
