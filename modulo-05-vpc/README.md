# Módulo 05 — Virtual Private Cloud (VPC): conceptos fundamentales

## Resumen

> Aviso para mí mismo: esta parte de redes no se asimila del todo a la primera lectura, y no pasa nada — se vuelve a repasar en módulos posteriores según se van usando estos conceptos en la práctica. Aparece también en el temario de las certificaciones Solutions Architect Associate y SysOps Administrator.

### ¿Qué es una VPC?
Es tu propia porción aislada de red dentro de AWS, sobre la que decides tú: qué rango de direcciones IP usar, cómo la divides en subredes, cómo enruta el tráfico, y qué puertas de salida/entrada tiene hacia internet. Al meter recursos distintos en subredes distintas puedes aplicar reglas de seguridad diferentes a cada grupo, en vez de tratar toda tu infraestructura como un único bloque.

### La VPC que ya viene incluida
Cada cuenta nueva de AWS trae de fábrica una VPC por defecto, y si lanzas una instancia EC2 sin decir explícitamente en qué VPC quieres que viva, cae ahí. Esa VPC por defecto ya tiene salida a internet configurada, y cada instancia recibe automáticamente una IP pública y sus correspondientes nombres DNS público/privado.

### Direccionamiento con CIDR
Una VPC se define con una notación CIDR: una IP base más una "máscara" tras la barra (ej. `10.0.0.0/24`) que indica cuántos de los 32 bits de la dirección quedan fijos para identificar la red — cuanto más bajo el número tras la barra, más direcciones libres quedan disponibles. AWS limita el tamaño de una VPC entre **/28 como mínimo (16 IPs)** y **/16 como máximo (65.536 IPs)**.

Para saber cuántas direcciones puedes usar realmente, la cuenta es `2^(32 - máscara) - 5`: siempre hay que restar 5, porque AWS se reserva un puñado de direcciones en cada subred, no las deja libres para tus recursos.

### Qué direcciones se reserva AWS (ejemplo con una subred `10.0.0.0/24`)
| Dirección | Para qué la usa AWS |
|---|---|
| `10.0.0.0` | Identifica la propia red |
| `10.0.0.1` | El router interno de la VPC |
| `10.0.0.2` | Resolución DNS |
| `10.0.0.3` | Reservada para uso futuro |
| `10.0.0.255` | Dirección de broadcast |

### Dividir en subredes
Una subred es un trozo del rango de la VPC con su propio CIDR, que no puede solaparse con el de otra subred de la misma VPC. La diferencia entre **pública** (accesible desde internet) y **privada** (no accesible desde fuera) no depende del tamaño ni del rango elegido, sino de cómo esté configurada su tabla de rutas — algo que se verá con más detalle en módulos posteriores.

Un diseño típico y resistente a fallos reparte subredes públicas y privadas en al menos dos AZ distintas dentro de la misma VPC: por ejemplo, dentro de una VPC `/16`, una subred pública y otra privada en la AZ A, y el mismo par en la AZ B.

### Para fijar el cálculo con un ejemplo
- Una VPC `10.1.0.0/16` te deja `65.536 - 5 = 65.531` direcciones utilizables.
- Si dentro divides una subred `10.1.0.0/24`, tienes `256 - 5 = 251` direcciones en esa subred.
- Otra subred `10.1.1.0/24` en la misma VPC te da, igualmente, `251` direcciones — y no se solapa con la anterior porque el tercer octeto es distinto.

## Comandos clave

*(No aplica en esta parte — es la base teórica; los componentes prácticos de red — Security Groups, NACLs, Internet Gateway, NAT — se ven en [[modulo-06-ec2-basico]] y módulos posteriores.)*

## Notas y gotchas

- El cálculo `2^(32-máscara) - 5` conviene tenerlo automatizado mentalmente — aparece mucho en preguntas de examen con distintos tamaños de máscara.
- No caer en el error de pensar que "pública/privada" es un atributo del propio rango CIDR: es puramente la tabla de rutas la que decide si una subred sale a internet o no.

## Recursos

- https://docs.aws.amazon.com/es_es/vpc/latest/userguide/amazon-vpc-limits.html
