# Módulo 03 — AWS IAM (Identity and Access Management)

## Resumen

IAM = servicio **global** que decide quién puede hacer qué dentro de tu cuenta AWS.

### Usuarios y grupos
- Cuenta **root**: poderes totales, creada por defecto — no usar para el día a día.
- Un usuario real = un usuario IAM (nunca compartir credenciales).
- Grupos solo contienen usuarios (no otros grupos anidados). Un usuario puede estar en varios grupos o en ninguno.
- Gestionar permisos por grupo > gestionar uno a uno.

### Políticas (permisos en JSON)
- Regla de oro: **mínimo privilegio** — si una acción no está permitida explícitamente, se deniega por defecto.
- Estructura básica:
  ```json
  {
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Action": ["s3:GetObject", "s3:PutObject"],
        "Resource": "arn:aws:s3:::mi-bucket/*"
      }
    ]
  }
  ```
  - `Effect` (Allow/Deny) + `Action` (qué) + `Resource` (sobre qué) — opcional `Principal` y `Condition`.
- **Política heredada** (vía grupo, un cambio se propaga a todos) vs **política en línea** (pegada a un usuario/recurso concreto, para casos puntuales).

### Contraseñas y MFA
- Política de contraseñas: longitud mínima, mezcla de caracteres, caducidad, no repetir.
- **MFA**: segundo factor (código en móvil, YubiKey, token hardware) — protege aunque roben la contraseña.

### Tres puertas de entrada a AWS
| Vía | Se autentica con |
|---|---|
| Consola web | Usuario/contraseña + MFA |
| AWS CLI | Access Keys |
| AWS SDK | Access Keys |

- Access Keys = usuario+contraseña para máquinas/scripts. Nunca en código fuente, rotar periódicamente.
- Dato curioso: el AWS CLI está construido sobre boto3 (SDK Python) — usado en [[modulo-07-ec2-avanzado]].

### Roles IAM
- Identidad **temporal**, sin credenciales fijas (a diferencia de un usuario).
- Dos piezas: **política de confianza** (quién puede asumir el rol) + **política de permisos** (qué puede hacer). Se asume vía `sts:AssumeRole`.
- Uso más común: dar permisos a un **servicio** de AWS (ej. EC2 leyendo de S3), sin meter Access Keys a mano.

### Auditoría
- **Informe de credenciales** (a nivel de cuenta): todos los usuarios, último uso, políticas, MFA activo o no.
- **Access Advisor** (a nivel de usuario): qué servicios usa y cuándo por última vez — detecta permisos sobrantes.

## Comandos clave

*(No hay CLI en las slides — práctica en consola: crear usuarios, grupos, políticas y roles.)*

## Notas y gotchas

- Nunca uses root para el trabajo diario, solo para configuración inicial.
- Pregunta típica de examen: "¿cómo le doy a mi EC2 acceso a S3?" → rol IAM, no credenciales hardcodeadas.
- Checklist rápido: sin root, un usuario = un IAM, permisos por grupo, mínimo privilegio, contraseña fuerte + MFA, roles para servicios, Access Keys solo CLI/SDK, auditar con Credential Report/Access Advisor, limpiar lo que no se usa.

## Recursos

-
