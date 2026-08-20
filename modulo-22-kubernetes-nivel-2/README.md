# Módulo 22 — Kubernetes: Nivel 2

## Resumen

### Redes en Kubernetes (un solo nodo)
- Al configurarse, Kubernetes crea una red privada interna a la que se conectan todos los Pods.
- Cada Pod recibe su propia IP interna (ej. `10.244.x.x`), distinta de la IP del nodo que lo aloja.

### Redes en múltiples nodos
- Sin configuración adicional, cada nodo genera su propio rango de red interna — si dos nodos usan el mismo rango, sus Pods pueden acabar con IPs duplicadas y la comunicación entre nodos falla. Kubernetes **no configura la red entre nodos automáticamente**.
- Se resuelve con un plugin de red (CNI) que le da a cada nodo un rango distinto y permite que Pods y nodos se alcancen entre sí en todo el clúster. Opciones habituales: Calico, Flannel, Cilium, WeaveNet, Cisco ACI, VMware NSX-T.

### Services
- Resuelven el problema de exponer/conectar grupos de Pods de forma estable — los Pods tienen IPs internas, no estáticas, y se recrean constantemente, así que no sirven como punto de conexión fijo.
- Un Service da un punto de acceso fijo delante de un grupo de Pods, dentro y fuera del clúster.
- El enlace entre un Service y sus Pods se hace con **labels y selectors**: el Service selecciona los Pods cuya etiqueta coincida con la que él busca — sin esa coincidencia, el Service no sabe a qué Pods dirigir el tráfico.

### Tipos de Service
| Tipo | Qué hace |
|---|---|
| **NodePort** | Expone un Pod a través de un puerto fijo en el nodo (rango 30000–32767) |
| **ClusterIP** | Crea una IP virtual interna del clúster, para comunicación entre servicios (no accesible desde fuera) |
| **LoadBalancer** | Reparte tráfico entrante entre varios Pods/nodos, normalmente aprovechando el balanceador del proveedor cloud |

### NodePort en detalle
- `targetPort`: puerto del Pod al que el Service reenvía tráfico (normalmente 80).
- `port`: puerto del propio Service, con su IP interna (Cluster-IP).
- `nodePort`: puerto en el nodo para acceso externo (30000–32767).

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30008
```
- `spec.selector`: etiqueta que debe coincidir con la del Pod (o los Pods) al que este Service enruta — sin esta coincidencia, el Service no encuentra destino.
- `spec.ports.port`: puerto del propio Service (Cluster-IP).
- `spec.ports.targetPort`: puerto en el que escucha el contenedor dentro del Pod.
- `spec.ports.nodePort`: puerto del nodo por el que se accede desde fuera del clúster.

### ClusterIP en detalle
- Un sistema típico tiene capas (frontend, backend, base de datos) que necesitan comunicarse entre sí sin depender de la IP concreta de cada Pod (que cambia constantemente).
- ClusterIP agrupa cada capa detrás de una IP y nombre estables dentro del clúster — cada capa puede escalar, reiniciarse o moverse de nodo sin romper la comunicación con las demás.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-service
spec:
  type: ClusterIP
  selector:
    app: backend
  ports:
    - port: 80
      targetPort: 80
```
- Sin `type` especificado, `ClusterIP` es el tipo por defecto de un Service.
- Solo es alcanzable **dentro** del clúster (por ejemplo, desde los Pods del frontend) — no expone `nodePort` porque no está pensado para acceso externo.

### LoadBalancer y acceso externo
- Para exponer la app a usuarios finales hace falta un Load Balancer que reenvíe tráfico hacia las IPs de los nodos, con el DNS de la organización apuntando a él.
- Configurar y mantener ese balanceador manualmente es tedioso — por eso en la práctica se delega en el proveedor cloud (AWS, Azure, GCP), que provisiona el balanceador automáticamente al crear un Service de tipo `LoadBalancer`.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
spec:
  type: LoadBalancer
  selector:
    app: frontend
  ports:
    - port: 80
      targetPort: 80
```
- Al crear este Service en un clúster cloud (ej. EKS), el proveedor aprovisiona automáticamente un balanceador de carga real (en AWS, un ELB) y lo enlaza con el Service.
- Sigue teniendo por debajo un `ClusterIP` y, en la mayoría de proveedores, también un `nodePort` — `LoadBalancer` añade una capa encima, no sustituye a las anteriores.

### Kubernetes en la nube: autogestionado vs. gestionado
| | Autogestionado | Gestionado (Kubernetes-as-a-Service) |
|---|---|---|
| Aprovisionar VMs | Tú | El proveedor |
| Instalar Kubernetes | Tú (con scripts) | El proveedor |
| Mantenimiento de las VMs | Tú | El proveedor |

### Amazon EKS
- Kubernetes administrado por AWS — alternativa a ECS, con el mismo objetivo (orquestar contenedores) pero con la API estándar de Kubernetes (no propietaria de AWS).
- Caso de uso típico: empresas que ya usan Kubernetes on-premise o en otra nube y quieren migrar a AWS sin cambiar de herramienta — Kubernetes es agnóstico de nube.

### Tipos de nodos en EKS
| Tipo | Quién gestiona qué |
|---|---|
| **Grupos de nodos gestionados** | AWS crea y gestiona las instancias EC2 (dentro de un ASG administrado por EKS); soporta on-demand o Spot |
| **Nodos autogestionados** | Tú creas las instancias y las registras en el clúster; el ASG lo gestionas tú; soporta on-demand o Spot |
| **Fargate** | Sin nodos que gestionar — EKS lanza los Pods directamente en Fargate |

### Arquitectura típica de un clúster EKS
- VPC repartida en varias zonas de disponibilidad, con subredes públicas (Load Balancers, NAT Gateway) y privadas (nodos worker y Pods).
- Puede tener un Load Balancer de Servicio EKS **público** (subredes públicas) y otro **privado** (subredes privadas), según si el tráfico es externo o interno al VPC.
- Los nodos worker (en subredes privadas) están cubiertos por un Auto Scaling Group, igual que en el patrón visto para ECS en el [Módulo 20](../modulo-20-caso-practico-arquitectura-ecs/README.md).

## Comandos clave

```bash
# Crear un clúster EKS (el plano de control administrado por AWS)
aws eks create-cluster \
  --name <cluster-name> --role-arn <cluster-role-arn> \
  --resources-vpc-config subnetIds=<subnet-id-1>,<subnet-id-2>

# Configurar kubectl para apuntar al clúster EKS
aws eks update-kubeconfig --name <cluster-name> --region <region>

# Crear un grupo de nodos gestionado
aws eks create-nodegroup \
  --cluster-name <cluster-name> --nodegroup-name <nodegroup-name> \
  --node-role <node-role-arn> --subnets <subnet-id-1> <subnet-id-2>

# Listar los Services del clúster (kubectl, ya con kubeconfig apuntando a EKS)
kubectl get services
kubectl get svc

# Exponer un Deployment como un Service (ej. tipo LoadBalancer)
kubectl expose deployment <deployment-name> --type=LoadBalancer --port=80 --target-port=80

# En un clúster local de pruebas (minikube), obtener la URL de acceso a un Service NodePort
minikube service <service-name> --url
```

## Notas y gotchas

- Kubernetes **no** configura automáticamente la red entre nodos — es fácil asumir que "viene incluido" y no lo está; hace falta un plugin de red (Calico, Flannel...) para que Pods de distintos nodos se vean entre sí.
- Sin `labels`/`selectors` coincidentes entre un Service y sus Pods, el Service existe pero no enruta tráfico a ningún sitio — es el fallo más típico al montar un Service desde cero.
- `ClusterIP` no es accesible desde fuera del clúster por diseño — si necesitas exposición externa hace falta `NodePort` o `LoadBalancer`, ClusterIP no es una opción "más simple" para eso.
- Igual que con ECS, en EKS el autoescalado de **Pods** y el autoescalado de las **instancias EC2 del ASG** son capas independientes — con nodos gestionados o autogestionados hay que pensar en ambas; con Fargate solo la primera aplica.
- Kubernetes es agnóstico de nube por diseño — es la principal razón para elegir EKS/Kubernetes en vez de ECS cuando ya hay inversión en Kubernetes fuera de AWS o se quiere evitar depender de un único proveedor.

## Recursos

- https://kubernetes.io/docs/concepts/services-networking/service/
- https://kubernetes.io/docs/concepts/cluster-administration/networking/
- https://docs.aws.amazon.com/eks/latest/userguide/what-is-eks.html
- https://docs.aws.amazon.com/eks/latest/userguide/eks-compute.html
