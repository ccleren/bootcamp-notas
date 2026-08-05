# Módulo 09 — Almacenamiento de EC2 (AMI, EBS, Instance Store, EFS)

## Resumen

### Amazon Machine Image (AMI)
- Una AMI es una **personalización de una instancia EC2**: tu propio software, configuración y sistema operativo, lista para lanzar nuevas instancias idénticas.
- Puedes lanzar instancias desde: una **AMI pública** (de AWS), **tu propia AMI**, o una AMI del **AWS Marketplace** (de terceros).
- Las AMIs son **específicas de región**, pero se pueden **copiar entre regiones**. Cuando ya no la necesitas, puedes cancelar su registro (deregister).

### EC2 Image Builder
- Automatiza la **creación, mantenimiento, validación y prueba** de AMIs (o imágenes de contenedores).
- Flujo: crear → construir componentes (personalizar software) → ejecutar el conjunto de pruebas (¿funciona? ¿es segura?) → distribuir la AMI (puede ser a varias regiones).
- Se puede ejecutar de forma programada (ej. semanalmente o cada vez que se actualizan paquetes).
- Servicio **gratuito** (solo pagas por los recursos que usa por debajo).

### Amazon EBS (Elastic Block Store)
- Volumen = unidad de red (no física) que se **adjunta a una instancia EC2** mientras corre; permite persistir datos incluso tras terminar la instancia. Piénsalo como "un USB de red".
- Al ser de red, tiene algo de latencia, pero se puede desconectar de una instancia y conectarlo a otra rápidamente.
- **Bloqueado a una Zona de Disponibilidad concreta** — no se puede adjuntar directamente a una instancia de otra AZ; para moverlo, primero hay que hacer un **snapshot** y restaurarlo en la otra AZ.
- Tiene capacidad **provisionada** (GB + IOPS) — se factura toda la capacidad reservada, la uses o no.
- Por defecto, el volumen root/raíz **se elimina al terminar la instancia** (se puede desactivar ese comportamiento).

#### Tipos de volumen EBS
| Tipo | Descripción | ¿Sirve como volumen de arranque? |
|---|---|---|
| **gp2 / gp3 (SSD)** | Uso general, equilibrio precio/rendimiento | Sí |
| **io1 / io2 Block Express (SSD)** | Mayor rendimiento, baja latencia, misión crítica | Sí |
| **st1 (HDD)** | Bajo coste, acceso frecuente y alto throughput | No |
| **sc1 (HDD)** | El más barato, acceso poco frecuente | No |

- **EBS Multi-Attach**: adjunta el mismo volumen a **hasta 16 instancias EC2** en la misma AZ con lectura/escritura completa — solo disponible en **io1/io2 Block Express**.
- **Redimensionar**: solo se puede **aumentar** tamaño/IOPS (gp2/gp3/io1/io2), nunca reducir directamente — para "reducir" hay que crear un volumen nuevo más pequeño y copiar los datos. Tras aumentar, el volumen pasa por `Modifying → Optimizing` (sigue siendo utilizable) y hace falta repartición (resize del filesystem) dentro del SO.

#### Snapshots de EBS
- Copia de seguridad de un volumen en un momento dado; se puede copiar entre AZs o regiones.
- **Archivado de snapshots**: mover a un nivel ~75% más barato, restauración tarda 24-72h, periodo mínimo de archivo 90 días — para snapshots mensuales/trimestrales/anuales.
- **Papelera de reciclaje de snapshots**: reglas de retención para recuperar snapshots borrados por error.
- **Fast Snapshot Restore (FSR)**: por defecto, restaurar un volumen desde snapshot puede sentirse lento al principio (los bloques se cargan desde S3 bajo demanda). FSR precarga el volumen completo al crearlo, sin lentitud inicial — pero se cobra por minuto mientras está activo, y puede salir caro; úsalo solo si necesitas rendimiento inmediato.

#### Amazon Data Lifecycle Manager (DLM)
- Automatiza el ciclo de vida de **snapshots y AMIs** de volúmenes EBS: crea, retiene y elimina automáticamente según reglas — evita hacer backups manuales.
- Usa **tags** en tus volúmenes/instancias para decidir qué gestionar.
- Soporta copias entre cuentas (cross-account).
- **No** puede gestionar snapshots/AMIs creados manualmente (fuera de DLM), ni instancias respaldadas por almacenamiento local (Instance Store).

### Instance Store (almacén de instancias)
- Almacenamiento de bloques **físicamente conectado** al host — mucho más rápido que EBS, pero **no persistente**: si paras o terminas la instancia, se pierden los datos.
- Casos de uso: caché temporal, buffers, workspaces, cargas que necesitan baja latencia/alta velocidad y pueden permitirse perder los datos.

### Amazon EFS (Elastic File System)
- Sistema de archivos **NFS gestionado** que se puede montar simultáneamente en **muchas instancias EC2 a la vez**, incluso en **multi-AZ**. A diferencia de EBS (que solo se adjunta a una instancia, o pocas con Multi-Attach y limitado a una AZ), EFS es la solución cuando varios servidores necesitan compartir los mismos archivos (ej. `wp-content` de un WordPress en varios servidores detrás de un Load Balancer).
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

- **EBS está atado a una AZ**, EFS no — es la diferencia que más se pregunta en examen. Si necesitas mover un volumen EBS de AZ, siempre pasa por snapshot + restaurar.
- Un volumen EBS **no se puede reducir** directamente — solo aumentar. Para "encoger" hay que crear uno nuevo y migrar datos.
- **Instance Store pierde los datos** al parar/terminar la instancia — nunca lo uses para nada que no puedas permitirte perder. Contraste directo con EBS y EFS, que sí persisten.
- Si montas EFS y no ves el archivo creado desde otra instancia, revisa el **Security Group del EFS** (necesita permitir NFS/2049 desde el SG de las instancias EC2) antes de sospechar del montaje.
- El nombre de la clase de almacenamiento activa (IA vs Standard) no cambia cómo accedes al archivo — es transparente, solo afecta al coste.
- DLM automatiza snapshots/AMIs pero **no** puede tocar nada creado manualmente fuera de DLM ni instancias con Instance Store — quedan fuera de su alcance.

## Recursos

- https://aws.amazon.com/es/blogs/storage/making-it-even-simpler-to-get-started-with-amazon-efs/
