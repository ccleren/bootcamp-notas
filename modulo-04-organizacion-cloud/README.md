# Módulo 04 — Crear una organización en el Cloud

## Resumen

### Por qué separar en varias cuentas
- El root de una cuenta tiene poder absoluto, sin forma de limitarlo.
- Por defecto todo bloqueado salvo para root.
- Mezclar dev/test/prod en la misma cuenta = un fallo en dev puede acabar tocando prod.
- Cuentas separadas por entorno actúan como cortafuegos entre ellas.

### Una cuenta vs. varias
| | Una sola cuenta | Varias cuentas |
|---|---|---|
| A favor | Todo desde una consola, menos piezas | Aísla el daño, permisos finos por cuenta, gasto claro por proyecto |
| En contra | Un fallo/brecha afecta a todo | Más trabajo de configuración |

Práctica del curso: crear cuenta `-PRODUCTION` (truco `+` de Gmail, ver [[modulo-02-infraestructura-global]]) con su propio MFA y presupuesto.

### AWS Organizations
- Servicio global: **cuenta de gestión** administra **cuentas miembro** (cada una solo pertenece a una organización).
- Cuentas agrupadas en **Unidades Organizativas (OU)**, anidables.
- Beneficios: facturación consolidada (una sola factura), descuentos por volumen compartidos entre cuentas, API para automatizar creación de cuentas.

### Service Control Policies (SCP)
- Limitan qué pueden hacer las cuentas de una OU, **aunque IAM lo permita**.
- Un SCP **nunca da permisos**, solo los quita.
- Permiso final = intersección entre lo que permite IAM **y** lo que permite el SCP heredado en cascada por la jerarquía de OUs.
- **No aplican a la cuenta de gestión** — esa puede hacer cualquier cosa, por eso hay que protegerla especialmente.

### IAM Identity Center (antes AWS SSO)
- Login único para acceder a varias cuentas AWS y apps externas, sin crear un usuario IAM por cuenta.
- Se integra con: identidad interna de AWS, AD gestionado, AD on-premise, SAML 2.0.
- Solución recomendada por AWS para identidad multi-cuenta.

### AWS Control Tower
Automatiza gobernanza multi-cuenta a gran escala, orquestando Organizations + IAM Identity Center + CloudFormation + Config:
- **Landing Zone**: entorno base para múltiples cuentas.
- **Guard Rails**: detectan/impiden configuraciones que rompen tus políticas.
- **Account Factory**: crea cuentas nuevas ya configuradas.
- **Dashboard**: vista y gestión centralizada.

## Comandos clave

```bash
# Crear la organización (desde la cuenta de gestión)
aws organizations create-organization --feature-set ALL

# Crear una cuenta miembro nueva
aws organizations create-account \
  --email produccion+cuenta@midominio.com \
  --account-name "Produccion"

# Crear una Unidad Organizativa (OU) bajo la raíz
aws organizations create-organizational-unit \
  --parent-id r-xxxx --name "Produccion-OU"

# Crear y aplicar un SCP
aws organizations create-policy \
  --name DenyS3 --type SERVICE_CONTROL_POLICY \
  --description "Deniega el acceso a S3" \
  --content file://deny-s3.json
aws organizations attach-policy \
  --policy-id p-xxxxxxxx --target-id ou-xxxx-xxxxxxxx

# Listar cuentas de la organización
aws organizations list-accounts
```

## Notas y gotchas

- Examen: un SCP nunca otorga permiso, solo restringe.
- La cuenta de gestión queda fuera del alcance de los SCP — MFA obligatorio, uso mínimo.
- Diferencia con [[modulo-03-iam]]: IAM opera a nivel usuario/rol dentro de una cuenta; SCP opera a nivel cuenta/OU dentro de la organización — capas distintas, se combinan.

## Recursos

- https://aws.amazon.com/blogs/mt/multi-account-strategy-for-small-and-medium-businesses/
