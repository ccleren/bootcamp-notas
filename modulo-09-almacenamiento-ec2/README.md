# Módulo 09 — Almacenamiento de EC2 (AMI, EBS, Instance Store, EFS)

## Resumen

### Amazon Machine Image (AMI): la "foto" de una instancia
Una AMI captura el estado completo de una instancia — sistema operativo, software instalado, configuración — para poder reutilizarlo y lanzar instancias nuevas idénticas a partir de ahí, en vez de configurar cada máquina desde cero. Puedes partir de una AMI pública mantenida por AWS, construir la tuya propia, o comprar una del AWS Marketplace. Están ligadas a una región concreta, aunque se pueden copiar a otras; y cuando ya no la necesitas, puedes simplemente darla de baja (deregister).

**EC2 Image Builder** automatiza todo ese proceso de crear/mantener AMIs: define los componentes que quieres instalar, corre pruebas automáticas para verificar que la imagen funciona y es segura, y distribuye el resultado (incluso a varias regiones a la vez) — se puede programar para que corra periódicamente, por ejemplo cada vez que se publican actualizaciones de seguridad. El propio Image Builder no tiene coste, solo pagas por los recursos que consume mientras construye y prueba la imagen.

### Amazon EBS: el disco duro "de red" de una instancia
Un volumen EBS es almacenamiento en bloques que se adjunta a una instancia EC2 mientras está encendida, y sobrevive aunque la instancia se apague o se termine (a menos que configures lo contrario para el volumen raíz). Al ser un recurso de red y no un disco físico soldado a la máquina, tiene algo de latencia — pero a cambio se puede desconectar de una instancia y reconectar a otra en segundos.

La limitación más importante: **un volumen EBS queda atado a una única Zona de Disponibilidad**. Si necesitas moverlo a otra AZ, el único camino es sacarle un snapshot y restaurarlo ahí. Además, se paga por toda la capacidad que reservaste, la estés usando o no.

Los tipos de volumen se diferencian por rendimiento y precio:

| Tipo | Pensado para | ¿Vale como volumen de arranque? |
|---|---|---|
| gp2 / gp3 (SSD) | Uso general, buen equilibrio precio/rendimiento | Sí |
| io1 / io2 Block Express (SSD) | Cargas críticas que necesitan máximo rendimiento y mínima latencia | Sí |
| st1 (HDD) | Acceso frecuente, throughput alto, coste bajo | No |
| sc1 (HDD) | El más barato, para datos que casi no se tocan | No |

Un volumen normal solo se puede adjuntar a una instancia (salvo con **Multi-Attach**, exclusivo de io1/io2, que permite compartirlo entre hasta 16 instancias de la misma AZ con lectura y escritura completas). Y en cuanto a redimensionar: solo se puede **aumentar** tamaño o IOPS, nunca reducir directamente — si necesitas un volumen más pequeño, tienes que crear uno nuevo y migrar los datos a mano.

**Snapshots**: son la forma de respaldar un volumen en un momento dado, y se pueden copiar entre AZs o regiones. Puedes archivarlos a un nivel de almacenamiento mucho más barato (aunque tarda entre 24 y 72 horas restaurarlos desde ahí), y existe una papelera de reciclaje para recuperar snapshots borrados por error dentro de un plazo. Si necesitas que un volumen restaurado desde snapshot rinda al máximo desde el primer segundo (por defecto tarda un poco en "calentarse" porque los bloques se traen bajo demanda), existe **Fast Snapshot Restore**, aunque tiene coste por minuto mientras está activo.

**Amazon Data Lifecycle Manager (DLM)** automatiza la creación, retención y borrado de snapshots y AMIs según reglas basadas en etiquetas — para no depender de que alguien se acuerde de hacer backups a mano. Eso sí, solo gestiona lo que él mismo creó: no toca snapshots hechos manualmente ni instancias que usan Instance Store.

### Instance Store: rápido pero volátil
Es almacenamiento físicamente conectado al servidor host — mucho más veloz que EBS, pero con una condición innegociable: **si paras o terminas la instancia, los datos desaparecen**. Solo tiene sentido para cachés temporales, buffers o cargas donde la velocidad importa más que la persistencia.

### Amazon EFS: cuando varias instancias necesitan ver los mismos archivos
EFS es un sistema de archivos NFS totalmente gestionado que se monta simultáneamente en muchas instancias EC2 a la vez, incluso repartidas en varias AZ. Es la respuesta cuando EBS se queda corto porque necesitas que **varios servidores lean y escriban el mismo conjunto de archivos** — el ejemplo típico es la carpeta de contenido de un WordPress servido por varias instancias detrás de un balanceador. Solo funciona con Linux (POSIX) y escala automáticamente sin que tengas que planificar capacidad, aunque sale más caro que EBS.

| | EBS | EFS |
|---|---|---|
| Se adjunta a | Una instancia (o hasta 16 con Multi-Attach) | Miles de instancias simultáneamente |
| Alcance | Una sola AZ | Varias AZ a la vez |
| Sistema operativo | Linux y Windows | Solo Linux |
| Coste | Más bajo | Más alto |
| Quién administra la infraestructura | Tú | AWS |

Para ahorrar en EFS existen distintas **clases de almacenamiento**: Standard (para archivos de acceso frecuente, multi-AZ), IA/Acceso Infrecuente (mucho más barata, con coste al recuperar), y las variantes One Zone (una sola AZ, ideal para entornos de desarrollo o backups). Con una política de ciclo de vida bien configurada (mover a IA tras N días sin tocar un archivo) el ahorro puede superar el 90% en los datos fríos.

Los **Access Points** permiten dar acceso solo a una carpeta concreta del EFS (por ejemplo `/datasets`) con un UID/GID fijo y permisos propios, en vez de exponer la raíz completa del filesystem a todo el mundo — el control fino se hace con políticas IAM por Access Point.

Hay operaciones que se hacen sin migración (cambiar la política de lifecycle, crear Access Points) y otras que requieren pasar por **AWS DataSync** (migrar a un EFS cifrado, o cambiar a modo de rendimiento Max I/O).

Métricas de CloudWatch a vigilar en EFS: `PercentIOLimit` (si llega al 100%, no hay más margen de I/O sin migrar a Max I/O), `BurstCreditBalance` (si se agota, el rendimiento cae) y `StorageBytes` (tamaño actual, se actualiza cada 15 minutos).

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

Puntos a tener en cuenta sobre la entrada de `fstab`:
- `file-system-id` es el ID del EFS (ej. `fs-0123456789abcdef0`).
- `_netdev` indica que hay que esperar a que la red esté lista antes de montar, porque EFS depende de la red para funcionar.
- El `mkdir` + `fstab` + `mount` hay que repetirlo en **cada** instancia que vaya a compartir el filesystem, no basta con hacerlo una vez.

## Notas y gotchas

- La diferencia que más aparece en examen: **EBS está atado a una AZ, EFS no**. Mover un volumen EBS de AZ siempre implica pasar por snapshot + restaurar.
- Un volumen EBS no se puede reducir de tamaño directamente — solo crecer. "Encoger" implica crear uno nuevo y migrar los datos.
- Instance Store pierde todo al parar/terminar la instancia — nunca lo uses para algo que no puedas permitirte perder, a diferencia de EBS y EFS que sí persisten.
- Si montas EFS y no ves el archivo creado desde otra instancia, revisa primero el Security Group del EFS (tiene que permitir NFS/puerto 2049 desde el SG de tus EC2) antes de sospechar del comando de montaje.
- DLM automatiza snapshots/AMIs pero no puede gestionar nada creado manualmente fuera de él, ni instancias que dependen de Instance Store.

## Recursos

- https://aws.amazon.com/es/blogs/storage/making-it-even-simpler-to-get-started-with-amazon-efs/
