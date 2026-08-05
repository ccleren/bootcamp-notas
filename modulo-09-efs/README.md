# Módulo 09 — Almacenamiento en EC2: Amazon EFS

## Resumen

- **Amazon EFS (Elastic File System)**: sistema de archivos **NFS gestionado** que se puede montar simultáneamente en **muchas instancias EC2 a la vez**, incluso en **multi-AZ**. A diferencia de EBS (que solo se adjunta a una instancia, o pocas con Multi-Attach y limitado a una AZ), EFS es la solución cuando varios servidores necesitan compartir los mismos archivos (ej. `wp-content` de un WordPress en varios servidores detrás de un Load Balancer).
- Solo compatible con instancias **Linux (POSIX)**.
- Escala automáticamente (hasta petabytes) sin interrumpir la aplicación. Más caro que EBS, pago por uso.

### EBS vs EFS

| | EBS | EFS |
|---|---|---|
| Adjuntable a | 1 instancia (o varias con Multi-Attach, solo io1/io2) | Miles de instancias a la vez |
| Alcance | Bloqueado a una AZ | Multi-AZ |
| SO | Linux y Windows | Solo Linux (POSIX) |
| Precio | Más barato | Más caro |
| Gestión | Tú gestionas la infraestructura | Totalmente gestionado |

### Clases de almacenamiento (con política de ciclo de vida: mover archivos tras N días sin acceso)
- **Standard**: acceso frecuente, Multi-AZ, para producción.
- **EFS-IA** (Acceso Infrecuente): mucho más barato en almacenamiento, con coste de recuperación al leer. Se activa vía política de ciclo de vida.
- **One Zone / One Zone-IA**: una sola AZ, ideal para dev/backup, más barato, tiene copia de seguridad activada por defecto.
- Ahorro potencial: >90% pasando archivos fríos a IA.

### Access Points
- Puerta de entrada específica a una ruta del EFS (ej. `/datasets`, `/builds`) con **UID/GID fijos** (POSIX) y permisos propios por grupo de usuarios.
- Se controla el acceso con políticas **IAM** por Access Point, evitando dar acceso a la raíz completa del filesystem.

### Operaciones
- **Sin migración** (in-place): cambiar política de lifecycle (IA/Archive), cambiar modo automático/aprovisionado, crear/modificar Access Points.
- **Con migración vía AWS DataSync**: migrar a un EFS cifrado, o cambiar el modo de rendimiento (ej. a Max I/O).

### Métricas de CloudWatch para EFS
- `PercentIOLimit` — % de I/O usado; al 100% no puedes ir más rápido → considerar migrar a Max I/O.
- `BurstCreditBalance` — créditos de ráfaga; a 0 el rendimiento cae hasta que se recargan.
- `StorageBytes` — tamaño actual (Standard + IA + Total), se actualiza cada 15 min.

## Comandos clave

Montaje de un EFS en dos instancias EC2 (Amazon Linux, `dnf`):

```bash
# En cada instancia que va a montar el EFS
df -k                                        # ver el estado actual de discos
sudo dnf -y install amazon-efs-utils        # utilidad de montaje EFS
sudo mkdir -p /efs/wp-content               # punto de montaje

# Añadir entrada persistente en fstab (se monta automáticamente al reiniciar)
sudo vi /etc/fstab
# file-system-id:/ /efs/wp-content efs _netdev 0 0

sudo mount /efs/wp-content                  # montar ahora mismo
df -k                                        # verificar que aparece montado

# Prueba de que el filesystem es compartido: crear un archivo desde una
# instancia y verificar que aparece en la otra con `ls -la`
cd /efs/wp-content
sudo touch demoefs.txt
```

Puntos clave del comando de fstab:
- `file-system-id` es el ID del EFS (ej. `fs-0123456789abcdef0`).
- `_netdev` le dice al sistema que espere a que la red esté disponible antes de montar (necesario porque EFS es un recurso de red).
- Hay que repetir el `mkdir` + `fstab` + `mount` en **cada** instancia que vaya a compartir el filesystem.

## Notas y gotchas

- Si montas EFS y no ves el archivo creado desde otra instancia, revisa el **Security Group del EFS** (necesita permitir NFS/2049 desde el SG de las instancias EC2) antes de sospechar del montaje.
- El nombre de la clase de almacenamiento activa (IA vs Standard) no cambia cómo accedes al archivo — es transparente, solo afecta al coste.

## Recursos

- https://aws.amazon.com/es/blogs/storage/making-it-even-simpler-to-get-started-with-amazon-efs/
