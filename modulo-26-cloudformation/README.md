# Módulo 26 — AWS CloudFormation

## Resumen

### Por qué Infraestructura como Código
- El trabajo manual en la consola es difícil de reproducir: en otra región, en otra cuenta, o si se borra todo dentro de la misma región.
- La solución es que la infraestructura sea código: ese código se despliega y crea/actualiza/elimina la infraestructura de forma repetible.

### Qué es CloudFormation
- Forma **declarativa** de describir infraestructura AWS: en una plantilla dices qué recursos quieres (EC2, Security Group, S3, ELB...) y CloudFormation los crea en el orden correcto con la configuración exacta especificada.
- Gratis en sí mismo — solo pagas los recursos que la plantilla aprovisiona.
- Disponible en todas las regiones.

### Cómo funciona
- Subes una plantilla (JSON/YAML), local o desde un bucket S3, a CloudFormation.
- CloudFormation la traduce en llamadas a la API y crea un **stack** (conjunto de recursos).
- Para actualizar, no se edita la plantilla anterior — se sube una nueva versión.
- Los stacks se identifican por nombre; **borrar un stack borra todos los recursos que creó**.

### Ventajas
- **IaC**: sin creación manual, control de versiones (git), revisión de cambios vía código.
- **Coste**: cada recurso de un stack se puede etiquetar para ver su coste; se puede estimar el coste desde la propia plantilla; estrategia típica en dev — destruir el stack fuera de horario laboral y recrearlo al día siguiente.
- **Productividad**: destruir/recrear infraestructura sobre la marcha, generación automática de diagramas.
- **Separación de intereses**: dividir en varios stacks por capa (ej. Stack VPC, Stack Subredes, Stack Aplicaciones) en vez de un monolito.
- **No reinventar la rueda**: aprovechar plantillas y documentación ya existentes.

### Bloques de una plantilla
| Bloque | Para qué |
|---|---|
| `AWSTemplateFormatVersion` | Versión del formato |
| `Description` | Comentario sobre la plantilla |
| `Resources` | **Obligatorio** — los recursos AWS a crear |
| `Parameters` | Entradas dinámicas de la plantilla |
| `Mappings` | Variables estáticas (codificadas en la plantilla) |
| `Outputs` | Referencias a lo creado, exportables a otros stacks |
| `Conditions` | Condiciones para crear recursos/salidas |
| `Metadata` | Metadatos arbitrarios opcionales |

- Formato: YAML o JSON (pares clave-valor, objetos anidados, arrays).

### Recursos
- El único bloque obligatorio; hay más de 700 tipos soportados (casi todos los servicios AWS).
- No se puede generar una cantidad dinámica de recursos — todo tiene que estar declarado explícitamente, no hay generación de código dentro de la plantilla.
```yaml
Resources:
  MyEC2Instance:
    Type: AWS::EC2::Instance
    Properties:
      ImageId: <ami-id>
```

### Ejemplo introductorio: una instancia EC2 sencilla
```yaml
Resources:
  MyInstance:
    Type: AWS::EC2::Instance
    Properties:
      AvailabilityZone: us-east-1a
      ImageId: ami-0a3c3a20c09d6f377
      InstanceType: t2.micro
```
- Es el ejemplo mínimo del curso: un único recurso, sin parámetros ni referencias — a partir de aquí se le va añadiendo IP elástica y grupos de seguridad para ilustrar cómo crecen las plantillas.

### Ejemplo ampliado: EIP y grupos de seguridad
Partiendo del ejemplo anterior, se añade una IP elástica y dos grupos de seguridad (uno para SSH, otro con un parámetro para su descripción):
```yaml
Parameters:
  SecurityGroupDescription:
    Description: Descripcion del grupo de seguridad
    Type: String

Resources:
  MyInstance:
    Type: AWS::EC2::Instance
    Properties:
      AvailabilityZone: us-east-1a
      ImageId: ami-0a3c3a20c09d6f377
      InstanceType: t2.micro
      SecurityGroups:
        - !Ref SSHSecurityGroup
        - !Ref ServerSecurityGroup

  # una IP elastica para nuestra instancia
  MyEIP:
    Type: AWS::EC2::EIP
    Properties:
      InstanceId: !Ref MyInstance

  # nuestro grupo de seguridad EC2 para SSH
  SSHSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Habilitar acceso SSH mediante puerto 22
      SecurityGroupIngress:
      - CidrIp: 0.0.0.0/0
        FromPort: 22
        IpProtocol: tcp
        ToPort: 22

  # nuestro segundo grupo de seguridad EC2
  ServerSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: !Ref SecurityGroupDescription
      SecurityGroupIngress:
      - IpProtocol: tcp
        FromPort: 80
        ToPort: 80
        CidrIp: 0.0.0.0/0
      - IpProtocol: tcp
        FromPort: 22
        ToPort: 22
        CidrIp: 192.168.1.1/32
```
- `MyInstance` ahora referencia sus grupos de seguridad con `!Ref` — CloudFormation detecta esa dependencia automáticamente y crea primero los Security Groups.
- `ServerSecurityGroup` usa el parámetro `SecurityGroupDescription` en vez de un texto fijo, para poder reutilizar la plantilla con una descripción distinta en cada despliegue.
- Al volver a subir esta plantilla sobre el stack creado con el ejemplo mínimo, CloudFormation calcula la diferencia y solo crea lo que falta (la EIP y los dos Security Groups) — así es una **actualización de stack**, no una creación desde cero.

### Parámetros
- Forma de dar entradas a la plantilla — útil si la plantilla se va a reutilizar o si un valor no se puede conocer de antemano.
- Regla práctica: si es probable que ese valor cambie, conviértelo en parámetro.
- Límite: máximo 200 parámetros por plantilla; cada uno necesita un ID lógico único, un tipo compatible y un valor (o default).
```yaml
Parameters:
  InstanceTypeParameter:
    Type: String
    Default: t2.micro
    AllowedValues:
      - t2.micro
      - m1.small
      - m1.large
```
- Se referencian con `!Ref <NombreParametro>` (abreviatura de `Fn::Ref`) en cualquier parte de la plantilla.

### Pseudoparámetros
- Predefinidos por AWS, no se declaran — se usan igual que un parámetro normal vía `Ref`.

| Pseudoparámetro | Devuelve |
|---|---|
| `AWS::AccountId` | ID de la cuenta |
| `AWS::Region` | Región actual |
| `AWS::StackId` | ARN del stack |
| `AWS::StackName` | Nombre del stack |
| `AWS::NotificationARNs` | ARNs de notificación del stack |
| `AWS::NoValue` | No devuelve nada (útil para omitir una propiedad condicionalmente) |

### Mappings
- Variables fijas codificadas en la plantilla — útiles para diferenciar por región, entorno, tipo de AMI, etc., cuando ya conoces todos los valores posibles de antemano.
- Se usan con `!FindInMap [ MapName, TopLevelKey, SecondLevelKey ]`.
```yaml
Mappings:
  RegionMap:
    us-east-1:
      HVM64: ami-6411e20d
    us-west-1:
      HVM64: ami-c9c7978c
Resources:
  MyEC2Instance:
    Type: AWS::EC2::Instance
    Properties:
      ImageId: !FindInMap [RegionMap, !Ref "AWS::Region", HVM64]
```
- Regla general: **mappings** cuando el valor se puede deducir de antemano (región, AZ, cuenta, entorno); **parámetros** cuando el valor es específico del usuario.

### Outputs
- Declaran valores de salida que se pueden ver en consola/CLI, o importar en otros stacks (si se exportan).
- ⚠️ No se puede borrar un stack cuyas salidas están siendo referenciadas por otro stack.
```yaml
Outputs:
  StackSSHSecurityGroup:
    Description: Grupo de seguridad SSH
    Value: !Ref StackSSHSecurityGroup
    Export:
      Name: SSHSecurityGroup
```
- En otra plantilla, se consume con `!ImportValue SSHSecurityGroup`.

### Condiciones
- Controlan si un recurso o salida se crea, en función de una condición (entorno, región, valor de un parámetro...).
- Funciones lógicas disponibles: `Fn::And`, `Fn::Equals`, `Fn::If`, `Fn::Not`, `Fn::Or`.
```yaml
Conditions:
  CreateDevResources: !Equals [!Ref EnvType, dev]
Resources:
  MountPoint:
    Type: AWS::EC2::VolumeAttachment
    Condition: CreateDevResources
```

### Funciones intrínsecas principales
| Función | Para qué |
|---|---|
| `Ref` / `!Ref` | Valor de un parámetro o ID físico de un recurso |
| `Fn::GetAtt` | Un atributo concreto de un recurso (ej. la AZ de una EC2) |
| `Fn::FindInMap` | Buscar un valor en `Mappings` |
| `Fn::ImportValue` | Importar un output exportado por otro stack |
| `Fn::Base64` | Codificar un string en Base64 (típico para `UserData`) |
| `Fn::Sub`, `Fn::Join`, `Fn::Split`, `Fn::Select` | Manipulación de strings/listas |

### Rollback
- Si falla la **creación** de un stack: por defecto se revierte todo (se borra); se puede desactivar el rollback para depurar el problema.
- Si falla la **actualización**: el stack vuelve automáticamente al último estado bueno conocido.

### Solución de problemas frecuentes
| Error | Causa típica | Solución |
|---|---|---|
| `DELETE_FAILED` | Recurso no vacío (bucket S3), SG aún asociado a una EC2 | Automatizar con Lambda, o usar `DeletionPolicy: Retain` |
| `UPDATE_ROLLBACK_FAILED` | Cambios hechos fuera de CloudFormation, permisos insuficientes, falta señal de un ASG | Arreglar manualmente y usar `ContinueUpdateRollback` |

### Rol de servicio de CloudFormation
- Rol IAM que permite a CloudFormation actuar en tu nombre sobre los recursos del stack — permite aplicar mínimo privilegio: el usuario no necesita todos los permisos de los recursos, solo `iam:PassRole` sobre ese rol de servicio.

### DeletionPolicy
| Valor | Efecto |
|---|---|
| `Delete` (por defecto) | Borra el recurso — ⚠️ no funciona en un bucket S3 no vacío |
| `Retain` | Conserva el recurso aunque se borre el stack |
| `Snapshot` | Crea una snapshot antes de borrar (EBS, RDS, ElastiCache, Redshift, Neptune, DocumentDB) |

### Política de stack
- Por defecto, una actualización de stack permite actualizar cualquier recurso.
- Una **política de stack** (documento JSON) restringe esto — al definirla, todo queda protegido salvo lo permitido explícitamente.
```json
{
  "Statement": [
    { "Effect": "Allow", "Action": "Update:*", "Principal": "*", "Resource": "*" },
    { "Effect": "Deny", "Action": "Update:*", "Principal": "*", "Resource": "LogicalResourceId/ProductionDatabase" }
  ]
}
```

### Protección de terminación
- Evita el borrado accidental de un stack completo (independiente de `DeletionPolicy`, que actúa por recurso).

### Recursos personalizados (Custom Resources)
- Para recursos aún no soportados por CloudFormation, lógica de aprovisionamiento personalizada, o ejecutar scripts en creación/actualización/borrado (ej. vaciar un bucket S3 antes de eliminarlo).
- Respaldados normalmente por una función Lambda (o SNS), referenciada vía `ServiceToken`.
- Se definen en plantilla con `AWS::CloudFormation::CustomResource` o `Custom::MyCustomResourceTypeName`

### StackSets
- Crear/actualizar/eliminar el mismo stack en **múltiples cuentas y regiones** con una sola operación.
- Solo la cuenta administradora puede crear StackSets; se puede aplicar a toda una organización.

### Generador de IaC y Application Composer
- **Generador de IaC**: escanea recursos ya existentes en una cuenta y genera una plantilla CloudFormation (o CDK) a partir de ellos, para "importar" infraestructura creada manualmente.
- **Application Composer**: editor visual de arrastrar y soltar para componer infraestructura, generando la plantilla IaC automáticamente.

### User Data y Helper Scripts
- Se puede incluir un script (`UserData`) en la plantilla, codificado en Base64 con `Fn::Base64`.
- Problemas del User Data simple: scripts complejos difíciles de mantener, sin cambios dinámicos, sin confirmación de éxito/fallo.
- Solución: **CloudFormation Helper Scripts** (`cfn-init`, `cfn-signal`, `cfn-get-metadata`, `cfn-hup`):
  - `cfn-init`: instala paquetes, crea archivos de configuración, arranca servicios, leyendo la sección `AWS::CloudFormation::Init` del recurso.
  - `cfn-signal`: informa a CloudFormation si la configuración tuvo éxito o falló, usado junto a `CreationPolicy`/`WaitCondition` para que el stack no avance hasta confirmar que la instancia está lista.

### Stacks anidados (Nested Stacks)
- Un stack puede crear otros stacks dentro (stack raíz + stacks hijos) — permite reutilizar componentes, organizar por bloques y reducir duplicación. Se considera buena práctica.
- Al modificar un stack anidado, hay que actualizar también el stack principal que lo referencia.

### DependsOn
- Indica que un recurso debe crearse después de otro, cuando hay una dependencia que CloudFormation no detecta automáticamente (usar `!Ref`/`!GetAtt` sí crea la dependencia implícita).
```yaml
Resources:
  EC2Instance:
    Type: AWS::EC2::Instance
    DependsOn: DBInstance
```

## Notas y gotchas

- No se puede borrar un stack cuyos outputs estén importados por otro stack — hay que quitar primero esa dependencia (`Fn::ImportValue`) antes de poder eliminarlo.
- `DeletionPolicy: Delete` (el valor por defecto) falla silenciosamente su propósito en un bucket S3 no vacío — es de las causas más comunes de `DELETE_FAILED`.
- La política de stack protege por defecto **todos** los recursos en cuanto se define una — hay que declarar explícitamente el `Allow` para lo que sí se puede actualizar, no es una lista de excepciones sobre un estado permisivo.
- El generador de IaC y Application Composer no sustituyen el aprendizaje de la sintaxis — generan una plantilla de partida, pero conviene revisarla como cualquier otro código.
- `Mappings` vs. `Parameters` es una decisión de diseño, no solo de sintaxis: si el valor se puede deducir de una variable conocida (región, entorno), usa Mapping; si depende de quien despliega, usa Parameter.

## Recursos

- https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/Welcome.html
- https://docs.aws.amazon.com/AWSCloudFormation/latest/TemplateReference/aws-template-resource-type-ref.html
- https://docs.aws.amazon.com/AWSCloudFormation/latest/TemplateReference/intrinsic-function-reference.html
- https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/using-cfn-nested-stacks.html
- https://docs.aws.amazon.com/AWSCloudFormation/latest/TemplateReference/cfn-helper-scripts-reference.html
- https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/stacksets-concepts.html
