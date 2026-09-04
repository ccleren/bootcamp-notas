# Módulo 25 — Proyecto práctico: Terraform en AWS

## Resumen

Proyecto práctico que despliega una aplicación full-stack completa en AWS, con toda la infraestructura definida y aprovisionada vía Terraform (visto en el [Módulo 24](../modulo-24-terraform/README.md)), incluyendo un pipeline de CI/CD.

### Componentes de la arquitectura
- **Aplicación Web**: desplegada en **AWS Amplify** (hosting + build de frontend).
- **API**: expuesta a través de **API Gateway**, con lógica en **AWS Lambda** y también un servicio en **AWS Fargate** detrás de un **ALB** (mezcla de serverless y contenedores según el caso de uso).
- **Base de datos**: **Amazon DynamoDB**, usada tanto por Lambda como por el servicio en Fargate.
- **API externa**: la app consume **TheMovieDB API** (fuente de datos de películas) y usa el **API de GitHub** para la parte de favoritos.
- **CI/CD**: el código vive en **GitHub**, y **AWS CodePipeline** + **AWS CodeBuild** construyen la imagen de la app, la publican en **Amazon ECR** y despliegan los cambios.

### Cómo encajan los conceptos ya vistos
| Pieza | Módulo relacionado |
|---|---|
| Terraform (infraestructura como código) | [Módulo 24](../modulo-24-terraform/README.md) |
| Lambda, API Gateway | [Módulo 15](../modulo-15-lambda/README.md) |
| ECR, Fargate, ALB | [Módulo 19](../modulo-19-docker-ecs/README.md) |
| DynamoDB | (mencionada como almacenamiento serverless en varios módulos) |

### Por qué mezclar Fargate y Lambda en el mismo proyecto
- No es una elección de "todo o nada": partes de la API con carga constante o que necesitan más control (contenedores) van a **Fargate**; partes puntuales o dirigidas por eventos van a **Lambda** — el mismo criterio de elección visto en la tabla comparativa del [Módulo 15](../modulo-15-lambda/README.md).

## Notas y gotchas

- Este proyecto es un buen ejemplo de que "serverless" y "contenedores" no son alternativas excluyentes dentro de una misma arquitectura — conviven según lo que mejor encaje en cada pieza (Lambda para la API ligera, Fargate para el servicio con más carga o control necesario).
- Definir todo esto en Terraform (en vez de crearlo a mano) es lo que permite reproducir el entorno completo (Amplify, API Gateway, Lambda, Fargate, ALB, DynamoDB, pipeline) de forma consistente entre entornos.
- El pipeline CodePipeline + CodeBuild + ECR es el mismo patrón CI/CD que se repite en cualquier despliegue de contenedores en AWS: construir la imagen, publicarla en ECR, y que el servicio (aquí Fargate) recoja la nueva versión.

## Recursos

- https://developer.hashicorp.com/terraform/language/providers
- https://docs.aws.amazon.com/codepipeline/latest/userguide/welcome.html
- https://docs.aws.amazon.com/amplify/latest/userguide/welcome.html
