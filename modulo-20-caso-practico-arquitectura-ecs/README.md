# Módulo 20 — Caso práctico: Arquitectura ECS

## Resumen

Caso práctico que construye, paso a paso, una arquitectura ECS completa de referencia usando los servicios vistos en el [Módulo 19](../modulo-19-docker-ecs/README.md), aplicados sobre una VPC ya vista en módulos anteriores. Es una diapositiva-diagrama que se va ampliando por capas, así que el resumen sigue ese mismo orden de construcción.

Implementación real de este caso práctico: [docker-ecs-fargate-ec2](https://github.com/ccleren/docker-ecs-fargate-ec2).

### 1. Red base (VPC)
- Una VPC con 2 subredes públicas y 2 subredes privadas (una por AZ, para alta disponibilidad).
- **Subredes públicas**: ruta hacia el Internet Gateway.
- **Subredes privadas**: ruta hacia un NAT Gateway (situado en la subred pública) para salir a internet sin ser alcanzables desde fuera.
- Grupos de seguridad: `ALBSG` permite HTTP (80) desde cualquier origen (`0.0.0.0/0`); `ECSInstanceSG` permite todo el tráfico, pero solo si viene del `ALBSG` — el patrón de "el origen permitido es otro grupo de seguridad" en vez de un rango de IPs.

### 2. Clúster ECS sobre EC2
- El clúster ECS corre en instancias EC2 situadas en las **subredes privadas** (no expuestas directamente a internet).
- Las instancias asumen un `ecsInstanceRole` (rol IAM) para poder registrarse en el clúster y comunicarse con otros servicios AWS.
- Las imágenes de los distintos microservicios (`web`, `cats`, `dogs`) viven en repositorios separados de **Amazon ECR**.

### 3. Definiciones de tarea
- Por cada microservicio hay una definición de tarea propia (`webdef`, `catsdef`, `dogsdef`), cada una apuntando a su imagen correspondiente en ECR.
- Separar definiciones de tarea por microservicio permite escalar, actualizar y versionar cada uno de forma independiente.

### 4. Servicios ECS + balanceo + auto scaling
- Cada definición de tarea se ejecuta como un **servicio ECS** independiente (`service web`, `service cats`, `service dogs`), manteniendo el número deseado de tareas activas.
- Un **ALB** en las subredes públicas recibe el tráfico de los usuarios y lo enruta hacia los servicios correspondientes en las subredes privadas.
- Un **Auto Scaling Group** cubre las instancias EC2 del clúster, para añadir/quitar capacidad de cómputo según la demanda de los servicios.

### 5. Observabilidad
- **CloudWatch Logs**: centraliza los logs de los contenedores.
- **CloudWatch (métricas)**: sigue el rendimiento del clúster y de cada servicio.
- **Container Insights**: capa de monitorización específica para contenedores (CPU/memoria a nivel de tarea y servicio, no solo de instancia).

### Arquitectura final (de un vistazo)
```
Usuarios
   │
   ▼
Internet Gateway ── Subredes públicas (ALB, NAT Gateway)
                          │
                          ▼
                   Subredes privadas
              ┌───────────┴───────────┐
        Auto Scaling Group      Cluster ECS (EC2)
                                  ├─ service web  → webdef  → ECR:web
                                  ├─ service cats → catsdef → ECR:cats
                                  └─ service dogs → dogsdef → ECR:dogs
                                       │
                                       ▼
                    CloudWatch Logs · Métricas · Container Insights
```

## Notas y gotchas

- Colocar las instancias EC2 del clúster ECS en subredes **privadas** (con el ALB como única puerta de entrada pública) reduce la superficie de ataque — es el mismo patrón visto con ALB + EC2 privadas en el [Módulo 10](../modulo-10-elb/README.md).
- El grupo de seguridad de las instancias ECS permitiendo tráfico solo desde el grupo de seguridad del ALB (en vez de un rango de IPs) es más robusto: si el ALB cambia de IP, la regla sigue funcionando sin tocarla.
- Separar una definición de tarea y un servicio por cada microservicio (en vez de mezclar varios contenedores no relacionados en la misma tarea) facilita el escalado y despliegue independiente de cada uno.
- Container Insights no es lo mismo que las métricas básicas de CloudWatch para EC2 — da visibilidad a nivel de tarea/contenedor, que es el nivel que realmente importa en un clúster ECS.

## Recursos

- https://docs.aws.amazon.com/AmazonECS/latest/developerguide/Welcome.html
- https://docs.aws.amazon.com/AmazonECS/latest/developerguide/cloudwatch-container-insights.html
