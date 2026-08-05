# Módulo 05 — Virtual Private Cloud (VPC): conceptos fundamentales

## Resumen

> Sección densa si nunca trabajaste con redes — no hace falta pillarlo todo a la primera. Cae en el temario de las certificaciones Solutions Architect Associate y SysOps Administrator.

### ¿Qué es una VPC?
Tu propia red aislada dentro de AWS: eliges rango de IPs, subredes, tablas de rutas y puertas de entrada/salida a internet. Meter recursos en subredes distintas permite aplicar reglas de seguridad distintas a cada grupo.

### VPC por defecto
- Toda cuenta nueva trae una VPC por defecto; las EC2 caen ahí si no especificas otra.
- Ya tiene salida a internet, IP pública y DNS público/privado automáticos.

### CIDR
- Notación: IP base + máscara (ej. `10.0.0.0/24`). Menor número tras la barra = más direcciones libres.
- Rango de VPC permitido: **mínimo /28 (16 IPs) — máximo /16 (65.536 IPs)**.
- Direcciones utilizables: `2^(32 - máscara) - 5` (AWS siempre se reserva 5 por subred).

### Direcciones reservadas por AWS (ejemplo `10.0.0.0/24`)
| Dirección | Para qué |
|---|---|
| `10.0.0.0` | Identifica la red |
| `10.0.0.1` | Router de la VPC |
| `10.0.0.2` | Resolución DNS |
| `10.0.0.3` | Reservada para uso futuro |
| `10.0.0.255` | Broadcast |

### Subredes
- Trozo del rango de la VPC con su propio CIDR, sin solaparse con otras subredes de la misma VPC.
- **Pública** = accesible desde internet. **Privada** = no accesible desde fuera. La diferencia la marca la tabla de rutas, no el rango en sí.
- Diseño típico: subredes públicas + privadas repartidas en 2+ AZ dentro de la misma VPC.

### Ejemplo de cálculo
- VPC `10.1.0.0/16` → `65.536 - 5 = 65.531` IPs utilizables.
- Subred `10.1.0.0/24` dentro → `256 - 5 = 251` IPs.
- Otra subred `10.1.1.0/24` → también 251, sin solaparse (tercer octeto distinto).

## Comandos clave

*(No aplica — base teórica. Security Groups, NACLs, Internet Gateway y NAT se ven en [[modulo-06-ec2-basico]] y módulos posteriores.)*

## Notas y gotchas

- `2^(32-máscara) - 5` conviene tenerlo automatizado — muy típico de examen con distintas máscaras.
- Pública/privada no es un atributo del CIDR: lo decide la tabla de rutas de la subred.

## Recursos

- https://docs.aws.amazon.com/es_es/vpc/latest/userguide/amazon-vpc-limits.html
