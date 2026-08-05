# Módulo 05 — Virtual Private Cloud (VPC): conceptos fundamentales

## Resumen

> Nota del curso: esta sección puede resultar densa si nunca trabajaste con redes — no hace falta entenderlo al 100% a la primera, se refuerza más adelante. Es clave para las certificaciones AWS Solutions Architect Associate y SysOps Administrator.

### ¿Qué es una VPC?
Servicio que permite lanzar recursos AWS en una **red virtual definida por ti**, con control total sobre:
- Rangos de IP.
- Subredes.
- Tablas de rutas.
- Gateways de red.

Mejora la seguridad aislando recursos en distintas subredes y aplicando políticas por subred.

### VPC por defecto
- Toda cuenta nueva de AWS trae una **VPC por defecto**.
- Las EC2 se lanzan ahí si no especificas otra VPC.
- Tiene conectividad a internet, y las instancias reciben IP pública + nombre DNS público y privado automáticamente.

### Direccionamiento (CIDR)
- **CIDR** = IP base + máscara de red (ej. `10.0.0.0/24`). La máscara indica cuántos bits identifican la red (a menor número tras la `/`, más direcciones disponibles).
- Rango VPC permitido: **mínimo /28 (16 IPs) — máximo /16 (65.536 IPs)**.
- Fórmula de direcciones disponibles: `2^(32-máscara) - 5` (AWS siempre reserva 5 direcciones por subred).

### Direcciones reservadas por AWS (ejemplo con `10.0.0.0/24`)
| Dirección | Uso reservado |
|---|---|
| `10.0.0.0` | Dirección de red |
| `10.0.0.1` | Router de la VPC |
| `10.0.0.2` | Reservado por AWS (DNS) |
| `10.0.0.3` | Reservado para uso futuro |
| `10.0.0.255` | Dirección de difusión (broadcast) |

### Subredes
- Dividen el espacio de direcciones de la VPC en segmentos más pequeños; cada una tiene su propio rango CIDR, **sin solaparse** con otras subredes de la misma VPC.
- **Subred pública**: accesible desde internet.
- **Subred privada**: no accesible desde internet.
- Patrón típico multi-AZ: una VPC `/16` dividida en subredes públicas y privadas repartidas en 2+ AZs (ej. Subred pública A `10.0.10.0/20`, Subred privada A `10.0.20.0/20`, Subred pública B `10.0.30.0/20`, Subred privada B `10.0.40.0/20`).

### Ejercicio de cálculo (para fijar el concepto)
- VPC `10.1.0.0/16` → 65.536 - 5 = **65.531 IPs** disponibles.
- Subred A `10.1.0.0/24` → 256 - 5 = **251 IPs**.
- Subred B `10.1.1.0/24` → 256 - 5 = **251 IPs**.

## Comandos clave

*(No aplica en esta parte — es la base teórica; los componentes prácticos de red — Security Groups, NACLs, Internet Gateway, NAT — se ven en [[modulo-06-ec2-basico]] y módulos posteriores.)*

## Notas y gotchas

- El cálculo de IPs disponibles (`2^(32-máscara) - 5`) es un clásico de examen — practica con distintos tamaños de máscara.
- No confundir: **subred pública/privada** es una etiqueta lógica que depende de si su tabla de rutas apunta a un Internet Gateway, no de ningún atributo propio del rango CIDR.

## Recursos

- https://docs.aws.amazon.com/es_es/vpc/latest/userguide/amazon-vpc-limits.html
