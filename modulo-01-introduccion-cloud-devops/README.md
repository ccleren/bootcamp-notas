# Módulo 01 — Introducción a Cloud Computing y DevOps

## Resumen

### Qué problema resuelve el Cloud
- Infraestructura propia = pagar local, luz, refrigeración, personal 24/7, y planificar para desastres.
- Escalar hardware físico lleva tiempo (semanas), no minutos.
- El Cloud externaliza todo eso: alquilas cómputo/almacenamiento/BBDD bajo demanda, pagas por uso, escalas en minutos.

### Las 5 características que definen "Cloud de verdad"
- **Autoservicio**: pides un recurso sin que nadie del proveedor intervenga a mano.
- **Acceso por red** desde cualquier dispositivo/plataforma.
- **Multi-tenancy**: varios clientes comparten infraestructura física, aislados entre sí.
- **Elasticidad rápida**: escala arriba/abajo casi al instante según demanda.
- **Pago por medición real**, no cuota fija.

### Modelos de despliegue
- **Público**: recursos de un proveedor externo, accesibles por internet.
- **Privado**: infraestructura tipo-Cloud de uso exclusivo interno (control total, datos sensibles).
- **Híbrido**: mezcla de ambos.

### IaaS / PaaS / SaaS — cuánto gestiona el proveedor
| Capa | El proveedor da | Tú gestionas | Ejemplos |
|---|---|---|---|
| IaaS | Red, VMs, almacenamiento en bruto | SO, runtime, app | EC2, Azure VMs, DigitalOcean |
| PaaS | + runtime y entorno de ejecución | Solo tu código | Elastic Beanstalk, Heroku |
| SaaS | El producto entero, funcionando | Nada de infra | Gmail, Dropbox, QuickSight |

### Cómo se paga en AWS
Tres ejes: **cómputo** (tiempo usado), **almacenamiento** (GB guardados), **salida de datos** (la entrada es gratis).

### Modelo de responsabilidad compartida
- **AWS** protege el Cloud: centros de datos, hardware, red global.
- **Tú** proteges lo que hay dentro: IAM, cifrado, parches de tus instancias, diseño de red.
- Regla rápida: un bucket S3 mal configurado por ti no es culpa de AWS.

### DevOps y CI/CD
- **DevOps**: unir Dev y Ops para entregar software más rápido y con menos fricción.
- **CI/CD**: automatizar todo el camino desde el código hasta producción.
- Pipeline típico: `PLAN → CODE → BUILD → TEST → RELEASE → DEPLOY → OPERATE → MONITOR`.

## Comandos clave

*(No aplica — módulo puramente conceptual.)*

## Notas y gotchas

- El modelo de responsabilidad compartida es la base de IAM, VPC y seguridad — vuelve aquí cuando dudes "¿esto lo arregla AWS o yo?".
- Estar en el Cloud no te hace seguro por defecto: la mitad "dentro del Cloud" sigue siendo tuya.

## Recursos

- https://aws.amazon.com/es/compliance/shared-responsibility-model/
- https://aws.amazon.com/aup/ (política de uso aceptable)
