# Módulo 03 — AWS IAM (Identity and Access Management)

## Resumen

**IAM** es el servicio (global, no regional) que controla de forma centralizada quién puede acceder a qué recursos de AWS. "Llaves digitales" para personas y servicios.

### Usuarios y grupos
- La cuenta **root** se crea por defecto y **no debe usarse ni compartirse** para el día a día.
- Los **usuarios** son personas, pueden tener permisos individuales o heredados de un grupo.
- Los **grupos** solo contienen usuarios (no otros grupos). Un usuario puede pertenecer a varios grupos, o a ninguno.
- Buena práctica: un usuario "físico" = un usuario de AWS (no compartir cuentas).

### Políticas (Policies)
- Documentos **JSON** que definen permisos, adjuntados a usuarios, grupos o roles.
- Principio de **mínimo privilegio**: si una acción no está explícitamente permitida, se considera prohibida.
- Estructura de una política:
  ```json
  {
    "Version": "2012-10-17",
    "Id": "PolicyID12345",
    "Statement": [
      {
        "Sid": "1",
        "Effect": "Allow",
        "Principal": { "AWS": ["arn:aws:iam::123456789012:user/MyUser"] },
        "Action": ["s3:GetObject", "s3:PutObject", "s3:DeleteObject"],
        "Resource": ["arn:aws:s3:::mybucket/*", "arn:aws:s3:::mybucket2/*"]
      }
    ]
  }
  ```
  - `Version`: siempre `"2012-10-17"`.
  - `Statement`: obligatorio, una o más reglas.
  - Cada `Statement`: `Sid` (opcional), `Effect` (`Allow`/`Deny`), `Principal` (a quién aplica), `Action` (qué acciones), `Resource` (sobre qué recursos), `Condition` (opcional).
- **Políticas heredadas** (vía grupo, cambias en un sitio y afecta a todos) vs **políticas en línea** (adjuntas directamente a un usuario/recurso concreto, para casos personalizados).

### Seguridad de acceso
- **Política de contraseñas**: longitud mínima, tipos de caracteres, caducidad, evitar reutilización, permitir auto-cambio.
- **MFA (Multi-Factor Authentication)**: combina algo que sabes (contraseña) + algo que tienes (dispositivo) o algo que eres (biometría). Protege aunque roben la contraseña.
  - Opciones: app virtual MFA (Google Authenticator, Authy), clave de seguridad U2F (YubiKey), dispositivo MFA hardware.

### Formas de acceder a AWS
| Método | Se protege con |
|---|---|
| Consola de administración | Contraseña + MFA |
| AWS CLI | Access Keys (Access Key ID + Secret Access Key) |
| AWS SDK | Access Keys, para acceso programático desde código |

- Las **Access Keys** son como usuario+contraseña para programas: nunca se comparten ni se guardan en código fuente, se rotan periódicamente.
- El AWS CLI está construido, de hecho, sobre el AWS SDK para Python (**boto3**) — ver [[modulo-07-ec2-avanzado]] donde se usa boto3 directamente.

### Roles IAM
- **Identidades temporales** que otorgan permisos sin credenciales permanentes (a diferencia de un usuario IAM, que es acceso permanente).
- Usos típicos: acceso entre cuentas AWS, integración con identidades externas (federación), y sobre todo **dar permisos a servicios de AWS** (ej. una instancia EC2 que necesita leer de S3).
- Funcionamiento: la **política de confianza** (trust policy) define quién puede asumir el rol; la **política de permisos** define qué puede hacer una vez asumido. Se asume vía `sts:AssumeRole`, que devuelve credenciales temporales.
- Recomendación: dar solo los permisos necesarios al rol (igual que con usuarios).

### Herramientas de auditoría
- **Informe de credenciales de IAM** (a nivel de cuenta): lista todos los usuarios, último uso de credenciales, políticas asignadas, si tienen MFA.
- **IAM Access Advisor** (a nivel de usuario): qué servicios tiene permitidos un usuario y cuándo accedió por última vez a cada uno — útil para detectar permisos sobrantes.

### Modelo de responsabilidad compartida aplicado a IAM
- **AWS**: seguridad de la infraestructura/red global, análisis de configuración/vulnerabilidades, conformidad.
- **Tú**: gestión de usuarios/grupos/roles/políticas, activar MFA, rotar Access Keys, revisar patrones de acceso.

## Comandos clave

*(No hay CLI explícita en las slides — la práctica de este módulo es en consola: crear usuarios, grupos, políticas y roles vía IAM Console.)*

## Notas y gotchas

- **Nunca uses la cuenta root para el día a día** — solo para configuración inicial de la cuenta.
- Prefiere asignar permisos a **grupos**, no a usuarios individuales — cambias en un sitio y se propaga.
- Los roles IAM no son solo para usuarios: son la forma estándar de dar permisos a **servicios** (típico examen: "¿cómo le doy a mi EC2 acceso a S3?" → rol IAM, no Access Keys hardcodeadas).
- Checklist de buenas prácticas del curso: no usar root, un usuario físico = un usuario AWS, permisos vía grupos, mínimo privilegio, política de contraseñas fuerte, MFA activo, roles para servicios, Access Keys solo para CLI/SDK, revisión periódica con Credential Report/Access Advisor, eliminar lo que no se usa, no compartir credenciales.

## Recursos

-
