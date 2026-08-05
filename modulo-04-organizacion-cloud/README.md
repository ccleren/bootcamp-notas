# Módulo 04 — Crear una organización en el Cloud

## Resumen

### Fundamentos de cuenta AWS
- El **usuario raíz** de una cuenta tiene control total y **no puede ser restringido**.
- Una cuenta AWS es un **contenedor de identidades y recursos**: por defecto, todo acceso está denegado excepto para el usuario raíz.
- Los recursos se facturan al método de pago de esa cuenta específica.
- Usar **cuentas separadas** (DEV, TEST, PROD) contiene el impacto de errores administrativos o de actores maliciosos — si algo falla o se compromete en DEV, no afecta a PROD.

### Single Account vs. Multi Account
| | Single Account | Multi Account |
|---|---|---|
| Ventaja | Todo desde una consola, menos complejidad | Aísla recursos, reduce superficie de ataque, permisos granulares por cuenta, tracking de gastos por cuenta/proyecto |
| Desventaja | Un compromiso/error afecta a **todo** | Más esfuerzo de configuración/mantenimiento |

Práctica del curso: crear una segunda cuenta `-PRODUCTION` (usando el truco del `+` de Gmail — ver [[modulo-02-infraestructura-global]]), con su propio MFA en el root y su propio presupuesto.

### AWS Organizations
Servicio **global** para administrar varias cuentas de AWS desde una **cuenta de gestión** (management account); el resto son **cuentas miembro** (una cuenta miembro solo puede pertenecer a una organización).

Beneficios:
- **Facturación consolidada**: un único método de pago para todas las cuentas.
- **Descuentos por volumen agregado** (EC2, S3...) y **instancias reservadas compartidas** entre cuentas.
- API para automatizar la creación de cuentas.

**Entidades**: Cuenta de gestión → Unidades Organizativas (OU, se pueden anidar) → Cuentas miembro. Ej: OU raíz > OU (Dev) / OU (Prod) > cuentas individuales.

### Service Control Policies (SCP)
- Políticas que **limitan** qué pueden hacer las cuentas miembro de una OU, incluso si el usuario tiene permiso a nivel de IAM dentro de esa cuenta.
- **No aplican a la cuenta de gestión** (root de la organización — esa puede hacer cualquier cosa).
- Son **jerárquicas**: los `Deny` se heredan y acumulan en cascada por cada nivel de OU por el que pasa la cuenta.
- Una acción solo está permitida si **tanto** la política de IAM **como** el SCP la permiten (son un techo, no sustituyen a IAM).
- Ejemplos de SCP: `FullAWSAccess` (`Allow *`), `DenyS3` (`Deny s3:*`), `AllowS3EC2` (`Allow` solo `s3:*` y `ec2:*`).

### IAM Identity Center (antes AWS SSO)
- Login **único** (SSO) para acceder a múltiples cuentas de AWS y aplicaciones externas.
- Se integra con: almacén de identidad interno de AWS, Microsoft AD gestionado, AD on-premise, proveedores SAML 2.0.
- Es la solución de identidad **recomendada por AWS** para gestionar SSO y permisos a través de cuentas — evita crear un usuario IAM por cada cuenta.

### AWS Control Tower
- Automatiza la configuración y gobernanza de entornos **multi-cuenta grandes**, orquestando Organizations + IAM Identity Center + CloudFormation + Config.
- Componentes clave:
  - **Landing Zone**: entorno base para manejar múltiples cuentas.
  - **Guard Rails**: detectan malas configuraciones y hacen cumplir políticas de seguridad.
  - **Account Factory**: automatiza la creación de cuentas nuevas con configuración estándar ya aplicada.
  - **Dashboard**: vista y gestión centralizada de todo el entorno.

## Comandos clave

*(No aplica — práctica basada en consola: AWS Organizations, SCPs, IAM Identity Center.)*

## Notas y gotchas

- Un SCP **nunca da permisos por sí solo**, solo los **restringe**: el permiso final es la intersección entre lo que permite IAM y lo que permite el SCP en cascada. Muy típico de examen.
- La cuenta de gestión de una organización queda **fuera del alcance de los SCP** — con gran poder, gran responsabilidad: protégela especialmente bien (MFA, uso mínimo).
- Ver [[modulo-03-iam]] para la diferencia entre políticas de IAM (a nivel de usuario/rol) y SCP (a nivel de cuenta/OU dentro de la organización) — son capas distintas y complementarias.

## Recursos

- https://aws.amazon.com/blogs/mt/multi-account-strategy-for-small-and-medium-businesses/
