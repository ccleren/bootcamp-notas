# Módulo 01 — Introducción a Cloud Computing y DevOps

## Resumen

### La idea central del Cloud, con mis palabras
Antes de que existiera el Cloud, si querías montar una app tenías que comprar o alquilar servidores físicos, pagar el local, la luz, la refrigeración y a alguien que vigilara esas máquinas 24/7. Si tu tráfico se disparaba, no podías "añadir más servidor" en cinco minutos — había que comprar hardware, instalarlo, configurarlo. Y si tu única sala de servidores se quedaba sin luz o sufría una inundación, tu aplicación caía sin más.

El Cloud Computing resuelve justo eso: en vez de ser dueño del hardware, **alquilas capacidad de cómputo, almacenamiento y bases de datos bajo demanda** a un proveedor (AWS, Azure, GCP...) y pagas solo por lo que usas, casi como pagar la luz o el agua. Puedes pedir exactamente el tamaño que necesitas y tenerlo funcionando en minutos, no semanas.

### Los cinco rasgos que definen "es Cloud de verdad"
Un servicio no es realmente Cloud Computing solo por estar "en internet". Para que lo sea de verdad, suele cumplir estos cinco puntos (son la definición estándar del sector, no algo específico de AWS):

- **Puedes autoservirte**: pides un recurso (una VM, una base de datos...) y lo tienes sin que nadie del proveedor tenga que intervenir manualmente.
- **Se accede por red desde cualquier sitio**: no importa si te conectas desde el portátil, el móvil o un servidor de otra empresa.
- **Varios clientes comparten la misma infraestructura física** (multi-tenancy) sin verse entre ellos ni pisarse datos — el proveedor se encarga del aislamiento.
- **Escala arriba y abajo casi al instante**: si tu app se vuelve viral hoy, en minutos tienes más capacidad; si baja el tráfico, la sueltas y dejas de pagarla.
- **Pagas por medición real de uso**, no por una cuota fija — de ahí que en AWS el coste de una instancia parada sea (casi) cero.

### Público, privado o híbrido
- Cuando todos los recursos son de un único proveedor externo y accesibles por internet, hablamos de **Cloud público**.
- Si una empresa levanta su propia infraestructura tipo-Cloud pero de uso exclusivo interno (por regulación, control total, datos muy sensibles), es **Cloud privado**.
- El **Cloud híbrido** combina ambos: parte de la infraestructura se queda on-premise y parte se mueve al Cloud público.

### Las tres capas de "cuánto gestiona el proveedor" (IaaS / PaaS / SaaS)
Cuanto más alto subes en esta pirámide, menos cosas tienes que administrar tú mismo — pero también menos control tienes:

| Capa | Qué te da el proveedor | Qué gestionas tú | Ejemplos |
|---|---|---|---|
| **IaaS** (infraestructura) | Red, servidores virtuales, almacenamiento en bruto | El sistema operativo, el runtime, la app | Amazon EC2, Azure VMs, DigitalOcean |
| **PaaS** (plataforma) | Todo lo anterior + el runtime y el entorno de ejecución | Solo tu código y su configuración | AWS Elastic Beanstalk, Heroku, Google App Engine |
| **SaaS** (software) | El producto entero, ya funcionando | Nada de infraestructura, solo lo usas | Gmail, Dropbox, Amazon QuickSight |

### Cómo se paga en AWS
El modelo de facturación de AWS se apoya en tres ejes: lo que **computas** (tiempo de CPU/instancia), lo que **almacenas** (GB guardados) y lo que **sacas** de la red de AWS hacia fuera (la entrada de datos, en cambio, no se cobra). Este último punto sorprende a mucha gente al principio: subir datos a AWS es gratis, pero descargarlos puede tener coste.

### Quién es responsable de qué (Shared Responsibility Model)
Esto es clave y aparece constantemente en el resto del curso: AWS y tú os repartís la seguridad, pero en capas distintas.
- **AWS protege el Cloud en sí**: los centros de datos, el hardware físico, la red global, la disponibilidad de los servicios base.
- **Tú proteges lo que hay dentro del Cloud**: cómo configuras tus permisos de IAM, si cifras tus datos, si parcheas el sistema operativo de tus propias instancias, cómo diseñas tu red.

Dicho de forma simple: a AWS no le puedes echar la culpa de una brecha de seguridad causada por un bucket S3 mal configurado por ti — eso cae dentro de tu parte de la responsabilidad.

### DevOps y CI/CD, en corto
**DevOps** no es una herramienta sino una forma de trabajar: juntar al equipo que desarrolla (Dev) con el que opera en producción (Ops) para que los cambios lleguen más rápido y con menos fricción, en vez de tirar el código "por encima del muro" de un equipo a otro.

Esa forma de trabajar se apoya en **CI/CD** (integración y despliegue continuos): automatizar todo el camino desde que escribes código hasta que está sirviendo tráfico real, para poder liberar cambios con frecuencia sin que cada release sea un drama. El recorrido típico de un cambio pasa por: planificar → escribir el código → compilarlo/construirlo → probarlo → prepararlo para lanzar → desplegarlo → mantenerlo funcionando → vigilar que todo va bien.

## Comandos clave

*(No aplica — módulo puramente conceptual, sin práctica de CLI.)*

## Notas y gotchas

- El modelo de responsabilidad compartida es la base de casi todo lo que viene después en IAM, VPC y seguridad — cuando dudes "¿esto lo arregla AWS o yo?", vuelve a este esquema.
- No caigas en pensar que "estar en el Cloud" ya te hace seguro por defecto: la parte "dentro del Cloud" sigue siendo tuya.

## Recursos

- https://aws.amazon.com/es/compliance/shared-responsibility-model/
- https://aws.amazon.com/aup/ (política de uso aceptable)
