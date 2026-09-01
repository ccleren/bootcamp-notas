# Módulo 23 — Caso práctico: Kubernetes en AWS

## Resumen

Caso práctico que despliega una aplicación de microservicios completa sobre un clúster de Kubernetes en AWS, combinando los conceptos de los [Módulo 21](../modulo-21-kubernetes-nivel-1/README.md) y [Módulo 22](../modulo-22-kubernetes-nivel-2/README.md) (Pods, Services, tipos de Service) con servicios AWS ya vistos (ECR, SES).

Implementación real de este caso práctico: [kubernetes-eks](https://github.com/ccleren/kubernetes-eks).

### Componentes de la aplicación
- **Aplicación Web** (frontend): expuesta al navegador del usuario.
- **API NodeJS**: microservicio backend.
- **API .NET**: otro microservicio backend, independiente del anterior.
- **MongoDB**: base de datos usada por los microservicios.
- **Amazon SES**: envío de emails desde la aplicación.

### Origen de las imágenes
- Las imágenes propias de la app (web, APIs) se publican en **Amazon ECR**.
- La imagen de MongoDB se trae directamente de **Docker Hub** (imagen base pública, no propia).

### Arquitectura en el clúster
Cada componente corre en su propio **Pod**, expuesto mediante el tipo de **Service** que le corresponde según si necesita acceso externo o solo interno:

| Componente | Puerto del contenedor | Tipo de Service | Accesible desde |
|---|---|---|---|
| Aplicación Web | 80 | LoadBalancer | Fuera del clúster (navegador) |
| API NodeJS | 3000 | LoadBalancer | Fuera del clúster |
| API .NET | 8080 | LoadBalancer | Fuera del clúster |
| MongoDB | 27017 | ClusterIP | Solo dentro del clúster |

- Cada microservicio con necesidad de acceso externo tiene su propio `LoadBalancer` — no comparten uno solo, cada uno obtiene su propio balanceador y puerto público.
- MongoDB, al no necesitar exposición externa, usa `ClusterIP` — coherente con lo visto en el [Módulo 22](../modulo-22-kubernetes-nivel-2/README.md): una base de datos no debería ser alcanzable desde fuera del clúster.

## Notas y gotchas

- Este caso confirma la regla del Módulo 22: solo se usa `LoadBalancer` en los componentes que un usuario o sistema externo necesita alcanzar directamente — la base de datos se queda deliberadamente en `ClusterIP`.
- Mezclar imágenes propias (en ECR) con imágenes públicas de terceros (MongoDB desde Docker Hub) en el mismo clúster es habitual — no hace falta subir a ECR una imagen que ya es pública y no vas a modificar.
- Tener un microservicio en .NET y otro en Node.js en el mismo clúster ilustra la ventaja central de los contenedores: cada Pod es independiente del lenguaje/runtime de los demás, mientras hablen por red.

## Recursos

- https://kubernetes.io/docs/concepts/services-networking/service/
- https://docs.aws.amazon.com/ses/latest/dg/Welcome.html
