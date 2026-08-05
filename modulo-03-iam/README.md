# Módulo 03 — AWS IAM (Identity and Access Management)

## Resumen

**IAM** es el servicio (global, no atado a una región) que decide quién puede hacer qué dentro de tu cuenta de AWS. Es el equivalente al departamento de recursos humanos + seguridad de un edificio: da de alta a la gente, les asigna una tarjeta de acceso y define a qué plantas puede entrar cada una.

### Usuarios y grupos
- Toda cuenta AWS nace con un usuario **root**, con poderes totales — la norma no escrita es no tocarlo salvo para la configuración inicial y dejarlo guardado bajo llave (con MFA).
- Cada persona real que necesita acceso debería tener su **propio usuario IAM** — nunca compartir credenciales entre varias personas.
- Los usuarios se pueden agrupar (un grupo solo contiene usuarios, no otros grupos anidados), y un mismo usuario puede estar en varios grupos a la vez o en ninguno. Gestionar permisos por grupo es mucho más manejable que ir usuario por usuario.

### Políticas: el "qué puede hacer cada uno"
Los permisos se definen en documentos **JSON** llamados políticas, que se cuelgan de un usuario, un grupo o un rol. La regla de oro es el **mínimo privilegio**: si una acción no aparece explícitamente permitida, AWS la trata como prohibida por defecto — no hace falta "denegar" todo lo demás.

Una política tiene esta forma general:
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
Cada bloque `Statement` responde a tres preguntas: **¿permito o deniego? (`Effect`), ¿qué acción? (`Action`), ¿sobre qué recurso? (`Resource`)** — opcionalmente puedes acotar además a quién aplica (`Principal`) y bajo qué condiciones (`Condition`).

Hay dos formas de aplicar una política: **adjuntarla a un grupo** (la hereda automáticamente todo el que entra en ese grupo, y si la cambias, cambia para todos a la vez) o pegarla directamente a un usuario/recurso concreto como **política en línea**, para casos muy puntuales que no quieres que se propaguen.

### Reforzar el acceso: contraseñas y MFA
Puedes exigir una política de contraseñas (longitud mínima, mezcla de caracteres, caducidad, no repetir contraseñas anteriores), pero el salto de seguridad real viene de la **autenticación multifactor (MFA)**: además de la contraseña, pides un segundo factor (un código generado en el móvil, una llave física tipo YubiKey, o un token hardware). Así, aunque roben la contraseña, no pueden entrar sin ese segundo elemento.

### Tres puertas de entrada a AWS
| Vía | Se autentica con |
|---|---|
| Consola web | Usuario/contraseña + MFA |
| AWS CLI | Access Keys (par de claves: ID + secreta) |
| AWS SDK (código) | Access Keys también |

Las Access Keys funcionan como un usuario y contraseña pensados para máquinas/scripts en vez de personas — por eso nunca deben quedar escritas en el código fuente ni compartirse, y conviene rotarlas de vez en cuando. Un dato curioso: el propio AWS CLI, por debajo, está construido sobre el SDK de Python (**boto3**) — el mismo que se usa en el script de ETL del [[modulo-07-ec2-avanzado]].

### Roles IAM: permisos temporales, sin credenciales fijas
A diferencia de un usuario (acceso permanente con sus propias credenciales), un **rol** es una identidad que se "presta" temporalmente: alguien o algo lo asume, recibe unas credenciales que caducan solas, y cuando termina de usarlo esas credenciales dejan de servir. Esto se controla con dos piezas: una **política de confianza** que dice quién puede asumir el rol, y una **política de permisos** que dice qué puede hacer una vez lo asume (la llamada técnica de por medio es `sts:AssumeRole`).

Los roles se usan sobre todo para tres cosas: dar acceso entre distintas cuentas AWS, conectar con sistemas de identidad externos, y — el caso más habitual en el día a día — **darle permisos a un servicio de AWS**, como una instancia EC2 que necesita leer un bucket de S3 sin que tengas que meterle Access Keys a mano.

### Cómo auditar quién tiene acceso a qué
- El **Informe de credenciales** (a nivel de cuenta completa) lista todos los usuarios, cuándo usaron sus credenciales por última vez, qué políticas tienen y si activaron MFA.
- El **Access Advisor** (a nivel de un usuario concreto) muestra qué servicios tiene permitido usar y cuándo tocó cada uno por última vez — perfecto para detectar permisos que nadie usa y se pueden retirar.

### La parte de IAM en el modelo de responsabilidad compartida
AWS se encarga de que la infraestructura de IAM en sí funcione y esté protegida (no puedes "hackear" el servicio de IAM desde fuera). Tú te encargas de todo lo que configuras dentro: qué usuarios existen, qué permisos tienen, si obligas a MFA, si rotas las claves — ver también [[modulo-01-introduccion-cloud-devops]].

## Comandos clave

*(No hay CLI explícita en las slides del curso — la práctica de este módulo es en consola: crear usuarios, grupos, políticas y roles vía IAM Console.)*

## Notas y gotchas

- Regla que se repite en todo el curso: **nunca uses la cuenta root para el trabajo diario**, resérvala para tareas de configuración de cuenta.
- Es mejor gestionar permisos por grupo que uno a uno — cambias en un solo sitio y se propaga a todos los miembros.
- Pregunta típica de examen: "¿cómo le doy a mi instancia EC2 acceso a S3?" → la respuesta correcta casi siempre es un **rol IAM**, no credenciales metidas a mano en el código.
- Checklist rápido de buenas prácticas: nada de root para el día a día, un usuario real = un usuario IAM, permisos por grupo, mínimo privilegio, contraseñas fuertes + MFA, roles para servicios, Access Keys reservadas a CLI/SDK, auditar con el Credential Report/Access Advisor, y limpiar lo que no se usa.

## Recursos

-
