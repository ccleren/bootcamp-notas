# Módulo 19 — Docker en AWS: ECR, ECS, Fargate, EKS

## Resumen

### Qué es Docker
- Plataforma para empaquetar aplicaciones en **contenedores** que se ejecutan igual en cualquier sistema operativo o máquina — sin problemas de compatibilidad, comportamiento predecible.
- Más ligero que una máquina virtual: comparte el kernel del sistema operativo anfitrión en vez de virtualizar hardware completo.
- Se escala (crear/destruir contenedores) en segundos.

### Conceptos base
- **Dockerfile**: instrucciones de texto para construir una imagen (el "plano").
- **Imagen**: artefacto construido por capas con todo lo necesario para ejecutar la app.
- **Contenedor**: una instancia en ejecución de una imagen, aislada.
- Flujo típico: `Dockerfile` → `docker build` → **imagen** → `docker push` a un repositorio → `docker pull` desde otro sitio → `docker run` → **contenedor**.

### Dónde se almacenan las imágenes
- **Docker Hub**: repositorio público (o privado en planes de pago); incluye imágenes base de muchas tecnologías.
- **Amazon ECR** (Elastic Container Registry): repositorio de AWS, público (ECR Public Gallery) o privado; totalmente integrado con ECS, respaldado por S3. El acceso se controla vía IAM — un error de "permiso denegado" al hacer pull casi siempre es una política IAM, no un problema de Docker.

### Servicios de contenedores en AWS
| Servicio | Qué es |
|---|---|
| **Amazon ECS** | Orquestador de contenedores propio de AWS |
| **Amazon EKS** | Kubernetes administrado por AWS (open source) |
| **AWS Fargate** | Motor de ejecución serverless — funciona tanto con ECS como con EKS, sin gestionar instancias EC2 |
| **Amazon ECR** | Registro de imágenes de contenedores |

Lanzar contenedores Docker en AWS con ECS equivale a lanzar **tareas ECS dentro de un clúster ECS**.

### ECS: tipos de lanzamiento
| Tipo | Quién gestiona la infraestructura |
|---|---|
| **EC2** | Tú aprovisionas y mantienes las instancias EC2; cada instancia corre el **agente ECS** para registrarse en el clúster; controlas el SO y puedes instalar software propio |
| **Fargate** | AWS gestiona todo — sin instancias EC2 que administrar, solo defines la tarea; escala y factura según CPU/RAM usadas; aísla cada contenedor |

### Tarea vs. servicio
- **Definición de tarea**: JSON con imagen Docker, CPU, memoria, puertos, logging, volúmenes, red, variables de entorno, rol IAM y comportamiento ante fallo.
- **Tarea**: una ejecución concreta de una definición de tarea.
- **Servicio**: mantiene un número deseado de tareas en ejecución de forma continua — si una tarea falla, ECS lanza otra automáticamente. Puede vincularse a un ALB/NLB.

### Variables de entorno en tareas
- **Hardcoded**: valores directos en la definición (ej. URLs no sensibles).
- **SSM Parameter Store**: valores sensibles (claves API, configuración compartida).
- **Secrets Manager**: credenciales especialmente sensibles (contraseñas de BD).
- **Archivos de entorno masivos**: cargados desde un bucket S3.

### Roles IAM en ECS
- **Perfil de instancia EC2**: lo usa el agente ECS para llamar a la API de ECS, mandar logs a CloudWatch, hacer pull de imágenes de ECR, leer secretos.
- **Rol de tarea ECS**: permisos específicos por tarea/servicio, definidos en la propia definición de tarea — permite que distintos servicios tengan distintos permisos, en vez de compartir el rol de la instancia.

### Balanceo de carga
- Integración con ALB (HTTP/HTTPS) y NLB (TCP/UDP); las tareas se registran automáticamente en el balanceador al desplegarse.
- **Con EC2**: si solo defines el puerto del contenedor, ECS hace un mapeo dinámico a un puerto libre del host — el grupo de seguridad de la instancia debe permitir tráfico desde el grupo de seguridad del ALB en cualquier puerto.
- **Con Fargate**: cada tarea tiene su propia ENI con IP privada propia, así que solo hace falta definir el puerto del contenedor — más simple de configurar.

### Escalado automático de ECS
- Se integra con **AWS Application Auto Scaling**, basado en CPU/memoria del servicio, tiempo de respuesta, conexiones activas o recuento de solicitudes del ALB.
- Estrategias: **seguimiento de objetivo** (target tracking sobre una métrica), **escalado por pasos** (basado en una alarma de CloudWatch) o **programado** (fecha/hora fija).
- ⚠️ El autoescalado de un **servicio ECS** (nivel de tarea) es independiente del autoescalado de un **Auto Scaling Group de EC2** (nivel de instancia) — con lanzamiento EC2 puede hacer falta escalar ambos niveles (o usar un **proveedor de capacidad de clúster** que añade instancias EC2 automáticamente cuando falta capacidad).

### Volúmenes de datos en ECS
| Tipo | Para qué | Limitación |
|---|---|---|
| **Bind mounts** | Compartir datos entre contenedores de la misma tarea (ej. patrón sidecar de logs/métricas) | Funciona en EC2 y Fargate, pero los datos son efímeros/ligados al ciclo de vida |
| **Volúmenes EBS** | Ampliar almacenamiento, compartir datos entre contenedores de la misma instancia EC2 | Solo EC2 (no Fargate); si la tarea se mueve a otra instancia, pierde acceso al volumen |
| **Amazon EFS** | Almacenamiento compartido persistente multi-AZ | Funciona en EC2 y Fargate; S3 **no** se puede montar como sistema de archivos |

### Actualización continua (rolling update)
- ECS despliega las tareas nuevas gradualmente mientras mantiene las antiguas activas, según un **porcentaje mínimo de salud** (ej. 50%) y un **porcentaje máximo** (ej. 100%).
- Si el porcentaje de tareas sanas cae por debajo del mínimo durante el despliegue, ECS pausa y espera a que se recupere antes de seguir.
- Al completar el despliegue con todas las tareas nuevas sanas, ECS retira gradualmente las tareas de la versión antigua.

### Disparadores de tareas ECS
- **EventBridge**: programar tareas (ej. cada hora) o reaccionar a eventos (ej. un objeto subido a S3 dispara una tarea Fargate que procesa el archivo y guarda el resultado en DynamoDB) — todo sin gestionar servidores.
- **SQS**: un servicio en Fargate puede hacer long polling sobre una cola SQS y procesar los mensajes.

### Colocación de tareas (solo lanzamiento EC2, no aplica a Fargate)
- Proceso: identificar instancias que cumplen requisitos de CPU/memoria/puerto → que cumplen restricciones de colocación → que cumplen la estrategia → elegir.
- **Estrategias**: `binpack` (llenar instancias para minimizar el número usado, ahorra coste), `random`, `spread` (repartir uniformemente, ej. por zona de disponibilidad) — se pueden combinar.
- **Restricciones**: `distinctInstance` (cada tarea en una instancia distinta) o `memberOf` (expresión sobre atributos de la instancia, ej. tipo de instancia).

### ECS en cualquier lugar (ECS Anywhere)
- Permite ejecutar tareas ECS nativas en infraestructura propia (on-premise, otra VM) con el plano de control totalmente gestionado por AWS — requiere instalar el agente ECS y el agente SSM. Tipo de lanzamiento `EXTERNAL`. Útil para requisitos normativos o de latencia que impiden usar la nube pública.

### Otras herramientas relacionadas
- **AWS App Runner**: despliega apps web/APIs directamente desde un repo (GitHub) o una imagen en ECR, gestionando servidores, escalado y balanceo automáticamente — más sencillo que ECS para casos simples.
- **AWS Copilot**: CLI que provisiona toda la infraestructura típica de una app en contenedores (ECS, VPC, ELB, ECR) para simplificar el despliegue a producción.

## Comandos clave

```bash
# Crear un repositorio ECR
aws ecr create-repository --repository-name <repo-name>

# Autenticar Docker contra ECR
aws ecr get-login-password --region <region> \
  | docker login --username AWS --password-stdin <account-id>.dkr.ecr.<region>.amazonaws.com

# Subir/descargar una imagen a/desde ECR
docker push <account-id>.dkr.ecr.<region>.amazonaws.com/<repo-name>:<tag>
docker pull <account-id>.dkr.ecr.<region>.amazonaws.com/<repo-name>:<tag>

# Crear un clúster ECS
aws ecs create-cluster --cluster-name <cluster-name>

# Registrar una definición de tarea (a partir de un JSON local)
aws ecs register-task-definition --cli-input-json file://<ruta-task-definition.json>

# Crear un servicio ECS con lanzamiento Fargate
aws ecs create-service \
  --cluster <cluster-name> --service-name <service-name> \
  --task-definition <task-definition-family> --desired-count <n> \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={subnets=[<subnet-id>],securityGroups=[<security-group-id>],assignPublicIp=ENABLED}"

# Escalar un servicio existente (nº de tareas deseadas)
aws ecs update-service --cluster <cluster-name> --service <service-name> --desired-count <n>
```

## Notas y gotchas

- Un error de permisos al hacer `docker pull` de una imagen en ECR casi siempre es un problema de política IAM, no de configuración de Docker — es lo primero a revisar.
- No confundir el autoescalado de un **servicio ECS** (cuántas tareas corren) con el autoescalado de un **Auto Scaling Group de EC2** (cuántas instancias hay) — con lanzamiento EC2 son dos capas independientes que pueden necesitar ajustarse por separado; con Fargate solo existe la primera.
- Los volúmenes EBS montados en tareas ECS **no siguen a la tarea** si esta se reprograma en otra instancia — para persistencia real multi-instancia hace falta EFS, no EBS.
- Las estrategias de colocación de tareas (`binpack`, `random`, `spread`) y las restricciones (`distinctInstance`, `memberOf`) **solo aplican al lanzamiento EC2** — con Fargate, AWS decide la colocación y estas opciones no tienen efecto.
- El rolling update de ECS puede quedarse parado indefinidamente si el porcentaje mínimo de salud nunca se alcanza — vale la pena revisar el health check de las tareas nuevas antes de asumir que el despliegue está "colgado" por otra causa.

## Recursos

- https://docs.aws.amazon.com/AmazonECS/latest/developerguide/Welcome.html
- https://docs.aws.amazon.com/AmazonECR/latest/userguide/what-is-ecr.html
- https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task_definitions.html
- https://docs.aws.amazon.com/AmazonECS/latest/developerguide/AWS_Fargate.html
- https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task-placement.html
