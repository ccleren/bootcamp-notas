# Módulo 11 — Auto Scaling Groups (ASG)

## Resumen

Un ASG mantiene tu flota de EC2 ajustada a la carga real: añade instancias cuando hace falta, las retira cuando sobran, y siempre dentro de un rango mín/máx que tú defines. También registra automáticamente las instancias nuevas en el Load Balancer y reemplaza cualquiera que desaparezca o falle su health check. El ASG en sí es **gratis** — solo pagas las EC2 que lanza.

### Plantilla de lanzamiento
Define cómo se crea cada instancia nueva: AMI, tipo de instancia, User Data, par de claves SSH, volúmenes EBS, Security Groups, rol IAM, subredes, Load Balancer.

El ASG añade encima: **tamaño mínimo / máximo / capacidad deseada** + políticas de escalado.

### ASG + Load Balancer
El ELB comprueba la salud de las instancias y reparte tráfico solo entre las sanas. Si una falla, dos cosas ocurren: deja de recibir tráfico, y el ASG la reemplaza automáticamente.

### Políticas de escalado dinámico
| Tipo | Cómo funciona | Ejemplo |
|---|---|---|
| **Target tracking** | Mantiene una métrica en torno a un valor objetivo | "CPU media del ASG en torno al 40%" |
| **Step scaling** | Reacciona a una alarma de CloudWatch | "CPU >80% → +3 instancias", "CPU <20% → -1" |
| **Scheduled actions** | Anticipa carga conocida, sin depender de métrica | "Subir capacidad mínima los lunes a cierta hora" |

### Escalado predictivo
Usa patrones históricos (diario/semanal) para anticiparse en vez de reaccionar. Útil cuando: tráfico cíclico, patrones on/off recurrentes (batch), o instancias que tardan mucho en inicializar (el escalado reactivo llegaría tarde).

### Escalado con alarmas CloudWatch
Una alarma vigila una métrica agregada de **todo el grupo** (ej. CPU media de todas las instancias) y, al dispararse, activa la política de escalado.

### Instance Refresh
Para propagar un cambio (ej. cambiar el tipo de instancia de la plantilla) sin tumbar el servicio:
1. Crear nueva versión de la plantilla de lanzamiento.
2. Lanzar `StartInstanceRefresh` indicando el **% mínimo de salud** a mantener (ej. 70%).
3. El ASG sustituye instancias antiguas por nuevas de forma escalonada.

## Comandos clave

*(Pendiente — añadir comandos de AWS CLI/Terraform de la demo práctica, ej. `aws autoscaling create-auto-scaling-group`, `aws autoscaling put-scaling-policy`, `aws autoscaling start-instance-refresh`.)*

## Notas y gotchas

- El health check que usa el ASG puede venir del propio EC2 o del Load Balancer — hay que saber cuál está activo, cambia qué cuenta como "instancia mala".
- Para que el ASG reaccione rápido a picos de CPU, conviene monitorización detallada (1 min) — ver [[modulo-07-ec2-avanzado]].
- Instance Refresh respeta siempre el % mínimo de salud — en ASGs pequeños (mín=1) puede no tener margen para sustituir sin bajar momentáneamente de esa capacidad mínima real.

## Recursos

-
