# Módulo 21 — Kubernetes: Nivel 1

## Resumen

### Por qué hace falta un orquestador
Una vez la app está en un contenedor Docker, quedan preguntas que Docker por sí solo no resuelve: ¿cómo la ejecuto en producción?, ¿qué pasa si depende de otros contenedores (BBDD, mensajería)?, ¿cómo escalo cuando sube la carga y reduzco cuando baja? Todo ese trabajo de despliegue y gestión automática de contenedores es la **orquestación**.

### Opciones de orquestación
- **Kubernetes**: plataforma open source para automatizar despliegue, escalado y gestión de apps en contenedores.
- **Docker Swarm**: orquestador nativo de Docker.
- **Apache Mesos**: gestor de recursos más genérico, no limitado a contenedores.

### Arquitectura: nodos y clúster
- **Nodo**: una máquina (física o virtual) con Kubernetes instalado, donde se lanzan los contenedores (antes llamados "minions"). Conviene tener más de uno por si falla alguno.
- **Clúster**: conjunto de nodos agrupados — si un nodo cae, la app sigue accesible desde los demás, y varios nodos permiten repartir la carga.
- **Master**: un nodo especial configurado como maestro, responsable de orquestar y vigilar el resto de nodos del clúster.

### Ventajas de Kubernetes
- Tolera fallos de hardware (varias instancias en distintos nodos).
- Reparte el tráfico de forma equilibrada.
- Escala instancias en segundos ante un aumento de demanda, y también puede escalar el número de nodos sin parar la app.
- Todo se define de forma declarativa, vía archivos de configuración.

### Componentes del clúster
| Componente | Función |
|---|---|
| **API Server** | Punto de entrada único para todas las operaciones del clúster |
| **etcd** | Almacén clave-valor con todo el estado del clúster |
| **Kubelet** | Agente que corre en cada nodo worker |
| **Container Runtime** | Software que ejecuta los contenedores (Docker, pero también Rocket o CRI-O) |
| **Controller** | Ajusta el estado real al estado deseado automáticamente |
| **Scheduler** | Decide en qué nodo se coloca cada contenedor |

### Master vs. Worker
| | Nodo Master | Nodo Worker |
|---|---|---|
| Rol | Orquesta el clúster (`kube-apiserver`, `etcd`, `controller`, `scheduler`) | Aloja los contenedores en ejecución |
| Comunicación | Recibe estado de salud de los workers | Ejecuta las acciones que ordena el master, vía `kubelet` |

### Pods
- Kubernetes no despliega contenedores directamente sobre los nodos: los encapsula en un **Pod**, el objeto desplegable más pequeño de Kubernetes.
- Normalmente un Pod = un contenedor, pero un Pod **puede** tener varios contenedores si necesitan compartir red/almacenamiento y trabajar juntos (ej. un contenedor de ayuda que procesa datos para el contenedor principal — patrón *sidecar*).
- Escalar = crear más Pods (en el mismo nodo o en nodos nuevos si no queda capacidad), no hacer más grande un Pod existente.

### Definición de objetos en YAML
Todo objeto de Kubernetes (Pod, ReplicaSet, Deployment, Service...) se define en YAML con 4 campos principales:
- `apiVersion`: versión de la API usada (`v1`, `apps/v1`...).
- `kind`: tipo de objeto (`Pod`, `Service`, `ReplicaSet`, `Deployment`).
- `metadata`: nombre, etiquetas y demás info descriptiva del objeto.
- `spec`: la configuración específica del objeto.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    app: nginx
    tier: frontend
spec:
  containers:
    - name: contenedor-nginx
      image: nginx
```

### Replication Controller y ReplicaSet
- Mantienen varias instancias (Pods) de una misma app corriendo en el clúster, para que si un Pod falla los usuarios no se queden sin servicio (alta disponibilidad).
- Reparten los Pods entre varios nodos, actuando también como mecanismo de balanceo de carga y escalado.
- **Replication Controller** es la tecnología original; **ReplicaSet** es su sustituto recomendado hoy (usa `apiVersion: apps/v1` y selecciona los Pods a gestionar según sus etiquetas).

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx-replicaset
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx-container
          image: nginx
          ports:
            - containerPort: 80
```
- `spec.replicas`: número de Pods que debe mantener en ejecución (3 en este ejemplo).
- `spec.selector.matchLabels`: criterio para decidir qué Pods gestiona este ReplicaSet — debe coincidir con las etiquetas de `template.metadata.labels`.
- `spec.template`: la plantilla de Pod que se usa para crear cada nueva réplica.

### Deployments
- Gestiona un ReplicaSet (y por tanto un conjunto de Pods) para ejecutar una carga de trabajo, añadiendo la capacidad de **actualizar de forma continua** sin parar el servicio.
- Cada creación o actualización de un Deployment dispara un **rollout**, que genera una nueva **revisión** — permite hacer seguimiento de cambios y volver atrás (`rollback`) si algo falla.

### Estrategias de despliegue
| Estrategia | Cómo actualiza | Downtime |
|---|---|---|
| **Recreate** | Destruye todas las instancias antiguas y luego despliega las nuevas | Sí, la app queda inaccesible durante el cambio |
| **RollingUpdate** (por defecto) | Sustituye las instancias una por una, manteniendo el servicio activo | No |

## Comandos clave

```bash
# Crear un Pod directamente desde una imagen, sin archivo YAML (útil para pruebas rápidas)
kubectl run <pod-name> --image=<imagen>

# Listar los Pods del clúster
kubectl get pods

# Listar los Pods con más detalle (IP, nodo asignado, etc.)
kubectl get pods -o wide

# Ver el detalle completo de un Pod (eventos, estado, contenedores, IP...)
kubectl describe pod <pod-name>

# Borrar un Pod
kubectl delete pod <pod-name>

# Crear un objeto (ej. un ReplicaSet) a partir de un archivo YAML
kubectl create -f <replicaset.yaml>

# Ver el detalle de un ReplicaSet (réplicas deseadas/actuales, eventos, Pods gestionados...)
kubectl describe replicaset <replicaset-name>

# Cambiar el número de réplicas de un ReplicaSet ya creado
kubectl scale replicaset <replicaset-name> --replicas=<n>

# Borrar un ReplicaSet (y por defecto, los Pods que gestiona)
kubectl delete replicaset <replicaset-name>

# Aplicar/crear un objeto a partir de un archivo YAML
kubectl apply -f <archivo.yaml>

# Listar los Deployments del clúster
kubectl get deployments

# Ver el detalle de un Deployment (estrategia, réplicas, ReplicaSets asociados, eventos...)
kubectl describe deployment <deployment-name>

# Editar un Deployment existente en un editor interactivo (aplica los cambios al guardar)
kubectl edit deployment <deployment-name>

# Cambiar la imagen de un contenedor dentro de un Deployment (dispara un rollout)
kubectl set image deployment/<deployment-name> <container-name>=<imagen>:<tag>

# Consultar el estado de un rollout en curso
kubectl rollout status deployment/<deployment-name>

# Ver el historial de revisiones de un Deployment
kubectl rollout history deployment/<deployment-name>

# Revertir un Deployment a la revisión anterior
kubectl rollout undo deployment/<deployment-name>
```

## Notas y gotchas

- Un Pod nunca "crece" para escalar — escalar en Kubernetes siempre significa crear **más Pods**, no agrandar uno existente.
- Multi-contenedor en un Pod es la excepción, no la norma: solo tiene sentido cuando los contenedores están fuertemente acoplados y necesitan compartir red/almacenamiento (patrón sidecar) — para servicios independientes, van en Pods separados.
- Replication Controller está obsoleto en la práctica — usar siempre **ReplicaSet** (y normalmente ni eso directamente, sino a través de un **Deployment**, que ya gestiona el ReplicaSet por ti).
- `Recreate` no es la estrategia por defecto precisamente porque genera downtime — hay que elegirla explícitamente si de verdad se necesita (ej. cuando la app no soporta tener dos versiones corriendo a la vez).
- El container runtime de un nodo Kubernetes **no tiene que ser Docker** — es una idea que se repite del [Módulo 19](../modulo-19-docker-ecs/README.md): Kubernetes desacopla el orquestador del motor de contenedores concreto.

## Recursos

- https://kubernetes.io/docs/concepts/overview/components/
- https://kubernetes.io/docs/concepts/workloads/pods/
- https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/
- https://kubernetes.io/docs/concepts/workloads/controllers/deployment/
- https://kubernetes.io/docs/reference/kubectl/quick-reference/
