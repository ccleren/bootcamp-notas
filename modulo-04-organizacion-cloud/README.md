# Módulo 04 — Crear una organización en el Cloud

## Resumen

### Por qué una sola cuenta AWS no siempre es suficiente
El usuario root de una cuenta AWS tiene poder absoluto sobre ella y no hay forma de limitarlo — por diseño. Y por defecto, dentro de una cuenta nueva todo está bloqueado salvo para ese root. El problema aparece cuando metes en la misma cuenta tu entorno de pruebas, el de desarrollo y el de producción: un error humano o una credencial filtrada en "pruebas" puede acabar afectando a producción, porque técnicamente comparten el mismo espacio. Separar en cuentas distintas por entorno (dev, test, prod) actúa como un cortafuegos: lo que se rompe en una no se propaga a las demás.

### ¿Una cuenta para todo, o varias?
| | Una sola cuenta | Varias cuentas |
|---|---|---|
| A favor | Todo desde una única consola, menos piezas que mantener | Aísla el daño de un fallo/ataque, permisos más finos por cuenta, más fácil ver cuánto gasta cada proyecto |
| En contra | Un error o brecha puede afectar a todo lo que hay dentro | Más trabajo de configuración y mantenimiento |

Como ejercicio práctico del curso, se monta una segunda cuenta dedicada a producción (reutilizando el truco del alias `+` de Gmail — ver [[modulo-02-infraestructura-global]]), con su propio MFA en el root y su propio presupuesto, separada de la cuenta general.

### AWS Organizations: gestionar varias cuentas desde un único sitio
Es el servicio (global) que permite tener una **cuenta de gestión** desde la que administras todas las demás, llamadas **cuentas miembro** — cada cuenta miembro solo puede pertenecer a una organización a la vez. Las cuentas se agrupan en **Unidades Organizativas (OU)**, que a su vez se pueden anidar (por ejemplo, una OU raíz con una sub-OU de Desarrollo y otra de Producción, y dentro de cada una las cuentas reales).

¿Qué ganas al centralizar así?
- Un único método de pago y **una sola factura** para toda la organización.
- Los descuentos por volumen de uso (en EC2, S3...) y las instancias reservadas se comparten entre todas las cuentas — como si sumaran su consumo.
- Puedes automatizar la creación de cuentas nuevas vía API en vez de hacerlo a mano cada vez.

### Service Control Policies (SCP): el techo por encima de IAM
Un SCP es una política que se aplica a nivel de OU o de cuenta y **limita** lo que pueden hacer los usuarios/roles de esa cuenta, sin importar qué permisos les haya dado IAM por dentro. Es clave entender que un SCP nunca concede permisos — solo puede quitar. Para que una acción se ejecute realmente, tiene que estar permitida **a la vez** por la política de IAM del usuario **y** por el SCP heredado desde la jerarquía de OUs por la que pasa esa cuenta (los `Deny` se van acumulando cuanto más abajo estés en el árbol de OUs).

Un detalle importante: **la cuenta de gestión de la organización no está sujeta a ningún SCP** — puede hacer lo que quiera. Justo por eso conviene protegerla especialmente bien y usarla lo mínimo posible para operaciones del día a día.

### IAM Identity Center (el SSO de AWS)
Antes se llamaba AWS SSO. Permite iniciar sesión **una sola vez** y desde ahí acceder a varias cuentas de AWS (y hasta aplicaciones externas), sin tener que crear un usuario IAM distinto en cada cuenta. Puede conectarse con el propio directorio de identidades de AWS, con Active Directory (gestionado o on-premise), o con cualquier proveedor SAML 2.0. Es, de hecho, la vía que AWS recomienda para gestionar identidad en entornos multi-cuenta.

### AWS Control Tower: automatizar todo lo anterior a gran escala
Cuando el número de cuentas crece mucho, montar y mantener todo esto a mano se vuelve inviable — Control Tower orquesta Organizations, IAM Identity Center, CloudFormation y Config para automatizarlo, apoyándose en cuatro piezas:
- **Landing Zone**: el entorno base ya preparado para alojar múltiples cuentas.
- **Guard Rails**: reglas que detectan (o directamente impiden) configuraciones que rompen tus políticas de seguridad.
- **Account Factory**: crea cuentas nuevas ya con la configuración estándar aplicada, sin trabajo manual.
- **Dashboard**: panel único para ver y gestionar todo el entorno multi-cuenta.

## Comandos clave

*(No aplica — práctica basada en consola: AWS Organizations, SCPs, IAM Identity Center.)*

## Notas y gotchas

- Idea que conviene tener clara para exámenes: un SCP nunca otorga permiso, solo restringe — el permiso final siempre es la intersección entre lo que deja pasar IAM y lo que deja pasar el SCP.
- La cuenta de gestión queda fuera del alcance de los SCP, así que merece un cuidado extra (MFA obligatorio, uso mínimo posible).
- La diferencia con [[modulo-03-iam]]: IAM opera a nivel de usuario/rol dentro de una cuenta; los SCP operan a nivel de cuenta/OU dentro de una organización — son capas distintas que se combinan, no se sustituyen.

## Recursos

- https://aws.amazon.com/blogs/mt/multi-account-strategy-for-small-and-medium-businesses/
