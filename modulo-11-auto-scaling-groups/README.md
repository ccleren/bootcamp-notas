# Módulo 11 — Auto Scaling Groups (ASG)

## Resumen

Un **Auto Scaling Group** gestiona un conjunto de instancias EC2 para que la infraestructura se adapte automáticamente a la carga real. Objetivos:
- Añadir instancias cuando sube la carga, eliminarlas cuando baja.
- Mantener siempre un número mínimo y máximo de instancias en marcha.
- Registrar automáticamente las nuevas instancias en un Load Balancer.
- Volver a crear una instancia si se elimina o deja de estar sana (health check).
- **Los ASG son gratis**: solo pagas por las instancias EC2 que lanza.

### Plantilla de lanzamiento (Launch Template)
Define cómo se lanzan las nuevas instancias del ASG:
- AMI, tipo de instancia, EC2 User Data, par de claves SSH.
- Volúmenes EBS, grupos de seguridad, rol IAM.
- Información de red/subredes y de Load Balancer.

Además del template, el ASG define:
- **Tamaño mínimo / tamaño máximo / capacidad deseada.**
- Políticas de escalado.

### ASG + Load Balancer
El ELB delante del ASG comprueba la salud de las instancias (health checks) y reparte tráfico solo entre las que están sanas; si una falla, el ASG la reemplaza automáticamente.

### Políticas de escalado dinámico

| Tipo | Cómo funciona | Ejemplo |
|---|---|---|
| **Target tracking** | El más simple: defines un valor objetivo de una métrica | "Mantener la CPU media del ASG en torno al 40%" |
| **Simple / Step scaling** | Reacciona a una alarma de CloudWatch | "CPU > 80% → añade 3 instancias", "CPU < 20% → elimina 1" |
| **Scheduled actions** | Anticipa cambios de carga conocidos, sin depender de una métrica | "Subir la capacidad mínima de 8 a 15 los lunes a cierta hora" |

### Escalado predictivo
Usa patrones históricos (diarios/semanales) de tráfico para anticipar el escalado, en vez de reaccionar quando ya subió la carga. Útil cuando:
- El tráfico es cíclico (horario laboral, por ejemplo).
- Hay patrones recurrentes on/off (procesamiento por lotes).
- Las instancias tardan mucho en inicializarse (el escalado reactivo llegaría tarde).

### Escalado con alarmas de CloudWatch
Una alarma monitoriza una métrica agregada de **todo el ASG** (ej. CPU media de todas las instancias) y, al dispararse, activa el escalado (subir o bajar el número de instancias).

### Actualización de instancias (Instance Refresh)
Para desplegar cambios (ej. cambiar el tipo de instancia en la plantilla de `t2.micro` a `t3.2xlarge`):
1. Se crea una **nueva plantilla de lanzamiento**.
2. Se lanza `StartInstanceRefresh`, indicando el **% mínimo de salud** que debe mantenerse durante el proceso (ej. 70%).
3. El ASG va sustituyendo instancias de la plantilla antigua por la nueva de forma controlada, sin tirar toda la capacidad de golpe.

## Comandos clave

*(Pendiente — añade aquí los comandos de AWS CLI o Terraform que uses en la demo práctica del módulo, ej. `aws autoscaling create-auto-scaling-group`, `aws autoscaling put-scaling-policy`, `aws autoscaling start-instance-refresh`.)*

## Notas y gotchas

- El health check que usa el ASG para decidir si reemplaza una instancia puede venir del propio EC2 o del Load Balancer (health check de ELB) — asegúrate de saber cuál está configurado, porque cambia qué se considera "instancia no sana".
- Para que el ASG reaccione rápido a la CPU, conviene monitorización detallada de EC2 (cada 1 min) — ver [[modulo-07-ec2-avanzado]] sobre monitorización estándar vs. detallada.
- El Instance Refresh no es instantáneo: respeta el % mínimo de salud, así que en ASGs pequeños (ej. min=1) puede no poder reemplazar sin bajar temporalmente de la capacidad mínima real.

## Recursos

-
