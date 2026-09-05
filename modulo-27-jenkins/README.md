# Módulo 27 — Jenkins

## Resumen

### Por qué automatizar builds
- Es el corazón de DevOps: entregas más rápidas, seguras y eficientes.
- Cubre: obtener código de un sistema de control de versiones, ejecutar pruebas automatizadas, compilar (binario o imagen Docker), guardar artefactos en un repositorio accesible, desplegarlos.
- Hacerlo a mano cada vez es lento y propenso a errores: iniciar sesión en el repo correcto, montar un entorno de pruebas, ejecutar tests bloqueando el resto del trabajo, construir y publicar una imagen Docker, fusionar cambios a la rama principal.

### La solución: un servidor dedicado de automatización
- Prepara el entorno automáticamente, gestiona credenciales (ej. Docker), asegura que las herramientas necesarias estén instaladas, y ejecuta un build al detectar cambios en el repo.
- Flujo típico: probar código → construir la app → publicar en un repositorio → desplegar al servidor.

### Qué es Jenkins
- La herramienta más usada para automatizar builds y CI/CD.
- Código abierto y gratuito, corre en cualquier nube (AWS, Azure, GCP), soporta múltiples lenguajes (Java, Python, .NET, Node.js...), se integra con GitHub/Docker/etc. mediante plugins, y permite definir flujos de trabajo personalizados con un `Jenkinsfile`.
- Alternativas populares: GitHub Actions, GitLab CI/CD, Travis CI, Bamboo, TeamCity.

### Plugins
La integración con otras tecnologías se hace vía plugins:
| Categoría | Ejemplos |
|---|---|
| Control de versiones | Git, GitHub, GitLab, Bitbucket |
| Herramientas de compilación | Gradle, Maven, Ant |
| Despliegue | AWS EC2/ECS, Azure VM, Google Cloud Run |
| Notificaciones | Slack, Telegram, Email, Microsoft Teams |
| Repositorios de artefactos | Nexus, JFrog Artifactory, Amazon S3 |
| Calidad de código | SonarQube, Checkmarx, Codecov |
| Contenedores | Docker, Kubernetes, OpenShift |

### Instalar Jenkins
| Opción | Ventajas | Cuándo usarla |
|---|---|---|
| **Como contenedor Docker** | Instalación rápida, portátil entre servidores, fácil de actualizar/eliminar | Desarrollo, pruebas, CI/CD dinámico |
| **Directamente en el SO** | Mejor rendimiento (sin overhead de Docker), más control/personalización | Producción y entornos críticos |

### Roles en Jenkins
| Rol | Quién | Responsabilidades |
|---|---|---|
| **Administrador de Jenkins** | Operaciones/DevOps | Configura el clúster, instala plugins, gestiona nodos y credenciales, hace backups |
| **Usuario de Jenkins** | Desarrolladores | Crea jobs para ejecutar sus flujos de trabajo — acceso limitado, sin gestión de plugins/nodos/credenciales |

### Herramientas de construcción
- Cada tecnología necesita su herramienta disponible en Jenkins (ej. `npm` para una app Node.js).
- Dos formas de instalar/configurar herramientas: vía **plugin** desde la UI de Jenkins, o **instalación directa** (acceso SSH al servidor, o dentro del contenedor si Jenkins corre en Docker).

### Docker dentro de Jenkins
- Si Jenkins corre en un contenedor Docker, y dentro de ese contenedor se quiere usar Docker (para construir imágenes), hace falta **Docker in Docker (DinD)** — un runtime Docker separado dentro del contenedor de Jenkins.
- Por aislamiento, ese Docker interno **no tiene acceso directo al Docker del host** por defecto — sin configuración adicional, los contenedores lanzados desde Jenkins no pueden hablar con el Docker real del servidor.

### Tipos de Job
| Tipo | Cómo se configura | Cuándo usarlo |
|---|---|---|
| **Freestyle Job** | Vía UI, limitado a las opciones de los plugins instalados | Tareas simples |
| **Pipeline Job** | Como código (`Jenkinsfile`) o UI, con etapas (`stages`), condicionales, bucles, variables y ejecución en paralelo | Flujos de trabajo complejos |

### Freestyle Job vs. Pipeline Job
- Un flujo complejo con **Freestyle** se resuelve encadenando varios Jobs independientes (verificar código → tests → build → deploy) — difícil de mantener, dependiente de plugins para coordinar cada salto, cada Job hay que actualizarlo por separado.
- El mismo flujo con **Pipeline** es un único Job con varias **etapas** dentro — fácil de mantener, versionable en Git (vía `Jenkinsfile`), reutilizable, y los cambios se aplican en un solo sitio.
- Pipeline soporta ejecución en paralelo y condicionales (`if`, `when`) de forma nativa; Freestyle necesita lógica externa entre Jobs para lograr lo mismo.

### Pipeline as Code
- El pipeline se define en un `Jenkinsfile`, versionado junto al código — permite automatizar build/test/deploy con control de flujo (condicionales, bucles) y ejecución paralela.

```groovy
pipeline {
    agent any
    environment {
        DOCKER_REPO      = '<docker-repo>'
        APP_NAME         = '<app-name>'
        VERSION          = '<version>'
        DOCKER_USERNAME  = credentials('docker-username')
        DOCKER_PASSWORD  = credentials('docker-password')
    }
    stages {
        stage('Build Docker Image') {
            steps {
                sh 'docker build -t $DOCKER_REPO/$APP_NAME:$VERSION .'
            }
        }
        stage('Login to Docker Registry') {
            steps {
                sh 'docker login -u $DOCKER_USERNAME -p $DOCKER_PASSWORD'
            }
        }
        stage('Push Docker Image') {
            steps {
                sh 'docker push $DOCKER_REPO/$APP_NAME:$VERSION'
            }
        }
    }
}
```
- Las credenciales (`docker-username`, `docker-password`) se gestionan desde Jenkins, no se escriben en texto plano en el `Jenkinsfile`.

## Notas y gotchas

- Docker in Docker no da acceso automático al Docker del host — es la causa típica de que "el build funciona pero no puede publicar/ejecutar contenedores" cuando Jenkins corre dentro de un contenedor.
- La ventaja real de Pipeline sobre Freestyle no es solo "más funciones" — es que el flujo completo vive en **un único archivo versionado**, en vez de repartido entre la configuración de varios Jobs en la UI, que nadie versiona.
- Elegir Docker vs. instalación directa para Jenkins no es solo preferencia — Docker es la opción rápida para dev/test, pero producción suele preferir instalación directa por rendimiento y control, igual que la distinción vista en otros módulos entre entornos ágiles y entornos críticos.
- Las credenciales en un `Jenkinsfile` deben referenciarse vía `credentials('<id>')`, nunca como texto plano — es el mismo patrón de "nunca hardcodear secretos" ya visto con variables de entorno en Lambda y ECS.

## Recursos

- https://www.jenkins.io/doc/
- https://www.jenkins.io/doc/book/pipeline/jenkinsfile/
- https://www.jenkins.io/doc/book/pipeline/syntax/
- https://www.jenkins.io/doc/book/installing/docker/
