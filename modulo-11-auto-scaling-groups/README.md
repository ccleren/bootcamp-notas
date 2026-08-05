# Módulo 11 — Auto Scaling Groups (ASG)

## Resumen

Un Auto Scaling Group es el mecanismo que hace que tu infraestructura EC2 respire con la carga real en vez de quedarse fija: añade instancias cuando hace falta, las retira cuando sobran, y se asegura de que siempre haya un número de instancias dentro de un rango que tú defines. También se encarga de dos cosas más que fácilmente se olvidan: registrar automáticamente cada instancia nueva en el Load Balancer que tenga configurado, y sustituir cualquier instancia que desaparezca o deje de pasar sus comprobaciones de salud, sin que tengas que intervenir. Y lo mejor: el ASG en sí no cuesta nada, solo pagas las instancias EC2 que efectivamente levanta.

### La plantilla de lanzamiento: el "molde" de cada instancia nueva
Antes de que el ASG pueda crear instancias, necesita saber exactamente cómo hacerlo — eso es la plantilla de lanzamiento (Launch Template): qué AMI usar, el tipo de instancia, el script de User Data, el par de claves SSH, los volúmenes EBS, los Security Groups, el rol IAM, en qué subredes desplegar y a qué Load Balancer conectarse. Por encima de esa plantilla, el propio ASG añade sus propios parámetros: la capacidad mínima, la máxima, la deseada, y las políticas que deciden cuándo escalar.

### La combinación con un Load Balancer
Cuando pones un ELB delante del ASG, el balanceador se encarga de comprobar continuamente qué instancias están sanas y reparte tráfico solo entre esas — si una falla el chequeo, dos cosas ocurren en paralelo: deja de recibir tráfico nuevo, y el ASG la da de baja y lanza una de reemplazo siguiendo la plantilla.

### Tres formas de decidir cuándo escalar

| Estrategia | La idea | Ejemplo típico |
|---|---|---|
| **Seguimiento de objetivo (target tracking)** | Le dices al ASG qué valor quieres mantener en una métrica, y él ajusta la capacidad solo para acercarse a ese valor | "Que la CPU media del grupo ronde el 40%" |
| **Escalado por pasos (step scaling)** | Reacciona directamente a que salte una alarma de CloudWatch, con reglas explícitas | "Si la CPU supera el 80%, añade 3 instancias; si baja del 20%, quita 1" |
| **Acciones programadas** | No depende de ninguna métrica en tiempo real, solo de un horario que ya conoces de antemano | "Todos los lunes a primera hora, subir la capacidad mínima" |

Existe también el **escalado predictivo**, que en vez de reaccionar a lo que ya está pasando, se apoya en el histórico de tráfico (patrones diarios o semanales) para anticiparse. Tiene sentido sobre todo cuando el tráfico sigue un patrón cíclico conocido, cuando hay ráfagas de trabajo por lotes recurrentes, o cuando tus instancias tardan tanto en arrancar que un escalado puramente reactivo llegaría demasiado tarde para absorber el pico.

### Cómo entra CloudWatch en la ecuación
Una alarma de CloudWatch vigila una métrica agregada de todo el grupo (por ejemplo, la CPU media de todas las instancias juntas, no de una sola) y, al dispararse, es la que activa la política de escalado correspondiente — subir o bajar el número de instancias.

### Actualizar instancias sin tumbar el servicio (Instance Refresh)
Cuando necesitas propagar un cambio a todas las instancias del grupo — por ejemplo, pasar de un tipo de instancia más pequeño a uno más grande en la plantilla — no tiene sentido apagar todo de golpe. El flujo habitual es: crear una nueva versión de la plantilla de lanzamiento, lanzar un Instance Refresh indicando qué **porcentaje mínimo de capacidad sana** quieres mantener durante todo el proceso (por ejemplo, no bajar nunca del 70%), y dejar que el ASG vaya sustituyendo instancias antiguas por nuevas de forma escalonada, sin sacrificar disponibilidad de golpe.

## Comandos clave

*(Pendiente — añadir aquí los comandos de AWS CLI o Terraform de la demo práctica del módulo, por ejemplo `aws autoscaling create-auto-scaling-group`, `aws autoscaling put-scaling-policy`, `aws autoscaling start-instance-refresh`.)*

## Notas y gotchas

- El chequeo de salud que usa el ASG para decidir si reemplaza una instancia puede venir del propio EC2 o del Load Balancer — conviene tener claro cuál está activo, porque cambia qué se considera "instancia mala".
- Si quieres que el ASG reaccione rápido ante picos de CPU, necesitas monitorización detallada (cada 1 minuto) en vez de la estándar — detalle cubierto en [[modulo-07-ec2-avanzado]].
- El Instance Refresh respeta siempre el porcentaje mínimo de salud configurado, así que en grupos muy pequeños (por ejemplo, mínimo de 1 instancia) puede no tener margen para sustituir sin bajar momentáneamente de esa capacidad mínima real.

## Recursos

-
