# Módulo 08 — AWS Systems Manager (SSM)

## Resumen

Systems Manager resuelve un problema muy concreto: gestionar decenas o cientos de servidores (EC2, on-prem, incluso de otros proveedores cloud) sin tener que entrar uno por uno por SSH o RDP. Todo pasa por un **agente** instalado en cada máquina, que ya viene preinstalado en las AMIs oficiales de AWS y que se puede instalar a mano en cualquier otro servidor. Para máquinas que viven fuera de AWS existe un proceso de "activación híbrida": un código de activación más un rol IAM las conecta de forma segura al servicio, como si fueran una instancia más.

Para que todo esto funcione hacen falta tres piezas a la vez: el agente instalado, conectividad de esa máquina al endpoint de SSM, y un **rol IAM** con los permisos adecuados — sin ese rol, el agente puede estar corriendo perfectamente y aun así no lograr registrarse.

### Para qué se usa en el día a día
- Mantener inventario y aplicar parches de seguridad a muchas máquinas de una sola vez.
- Ejecutar comandos remotos y asegurarte de que las instancias mantienen la configuración que quieres (el "estado deseado").
- El caso más práctico de todos: **entrar a una instancia EC2 que vive en una subred privada, sin necesidad de SSH ni de montar un servidor bastión** — solo hace falta el agente y el rol correcto.

### Run Command: lanzar comandos a escala sin SSH
En vez de conectarte máquina a máquina, defines un **documento de comando** (una plantilla en JSON o YAML con los pasos a ejecutar) y lo lanzas contra un conjunto de instancias, ya sea eligiéndolas una a una, por etiquetas, o por grupo de recursos. Puedes controlar cuántas instancias se tocan a la vez (concurrencia) y a partir de cuántos fallos se da por perdida la tarea completa (umbral de error). Los resultados se pueden mandar a S3 o notificar por SNS, y todo el proceso queda registrado en CloudTrail: siempre puedes saber quién ejecutó qué comando y cuándo. El acceso está controlado por IAM, así que decides con precisión quién puede lanzar comandos sobre qué máquinas.

### Documentos SSM
Son la base reutilizable detrás de Run Command y de otras funciones del servicio (Automation, State Manager, Maintenance Windows, Distributor). Definen pasos secuenciales con parámetros dinámicos, se guardan versionados en un almacén central, y puedes escribir los tuyos propios o partir de plantillas ya hechas por AWS.

### Patch Manager: mantener parcheadas las instancias sin trabajo manual
Automatiza la aplicación de parches de seguridad usando **Patch Baselines** — reglas predefinidas por AWS según el sistema operativo (hay una específica para Amazon Linux 2, otra para Ubuntu, otra para Windows...) o personalizadas por ti. Sirve tanto para mantener producción actualizada de forma continua como para auditar el nivel de cumplimiento, y tiene un modo de emergencia ("Patch Now") para aplicar un parche crítico de inmediato ante una vulnerabilidad recién descubierta.

El flujo típico junta varias piezas: agrupas tus instancias en **grupos de parches**, cada uno con su propia baseline; defines una **ventana de mantenimiento** (horario, duración, a qué instancias afecta); Run Command aplica los parches dentro de esa ventana; y los resultados van a parar a **SSM Inventory**, que centraliza el historial para poder auditarlo después.

## Comandos clave

*(No hay comandos de CLI específicos en el material del curso — la práctica es vía consola: activar Session Manager o lanzar un Run Command sobre una instancia que tenga el rol IAM `AmazonSSMManagedInstanceCore`.)*

## Notas y gotchas

- El requisito que más se olvida: sin el rol IAM `AmazonSSMManagedInstanceCore` (o equivalente) adjunto a la instancia, el agente no puede registrarse contra el servicio aunque esté correctamente instalado.
- Encaja con lo visto en [[modulo-03-iam]]: Systems Manager es otro ejemplo más de servicio que necesita un rol asignado a la instancia — no credenciales de usuario metidas a mano.

## Recursos

-
