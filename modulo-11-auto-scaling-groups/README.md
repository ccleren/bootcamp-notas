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

```bash
# Crear una plantilla de lanzamiento
aws ec2 create-launch-template \
  --launch-template-name mi-plantilla \
  --launch-template-data file://launch-template-data.json

# Crear el Auto Scaling Group a partir de esa plantilla
aws autoscaling create-auto-scaling-group \
  --auto-scaling-group-name mi-asg \
  --launch-template LaunchTemplateName=mi-plantilla,Version='$Latest' \
  --min-size 1 --max-size 5 --desired-capacity 2 \
  --vpc-zone-identifier "subnet-xxxx,subnet-yyyy" \
  --target-group-arns arn:aws:elasticloadbalancing:...:targetgroup/mi-target-group/xxxx

# Política de target tracking (ej. mantener CPU media al 40%)
aws autoscaling put-scaling-policy \
  --auto-scaling-group-name mi-asg --policy-name cpu-target-tracking \
  --policy-type TargetTrackingScaling \
  --target-tracking-configuration '{"PredefinedMetricSpecification":{"PredefinedMetricType":"ASGAverageCPUUtilization"},"TargetValue":40.0}'

# Lanzar un Instance Refresh tras actualizar la plantilla
aws autoscaling start-instance-refresh \
  --auto-scaling-group-name mi-asg \
  --preferences '{"MinHealthyPercentage":70}'

# Ver el estado del grupo
aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names mi-asg
```

*(Cuando hagas la demo práctica de este módulo, sustituye estos ejemplos por tus propios comandos reales.)*

## Notas y gotchas

- El health check que usa el ASG puede venir del propio EC2 o del Load Balancer — hay que saber cuál está activo, cambia qué cuenta como "instancia mala".
- Para que el ASG reaccione rápido a picos de CPU, conviene monitorización detallada (1 min) — ver [[modulo-07-ec2-avanzado]].
- Instance Refresh respeta siempre el % mínimo de salud — en ASGs pequeños (mín=1) puede no tener margen para sustituir sin bajar momentáneamente de esa capacidad mínima real.

## Recursos

-
