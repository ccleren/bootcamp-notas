# Módulo 34 — Python en AWS: Boto3

## Resumen

### SDKs de AWS
- Permiten interactuar con AWS directamente desde el código de una app, sin pasar por la consola ni la CLI.
- SDKs oficiales: Python (**Boto3**), Java, JavaScript, Kotlin, .NET, Node.js, PHP, Go, Ruby, C++, Rust, SAP ABAP.

### Qué es Boto3
- SDK de AWS para Python, desarrollado y mantenido por Amazon.
- Soporta más de 200 servicios de AWS.
- Dato curioso: la propia **AWS CLI está construida sobre el SDK de Python** — cada comando `aws ...` que se ejecuta por CLI, internamente usa las mismas llamadas que expone Boto3.

### Instalación y laboratorios del curso
Los laboratorios se organizan en notebooks Jupyter por servicio: IAM, S3, Cost Explorer, EC2 (incluyendo conexión SSH), DynamoDB, Lambda y Textract.

```
boto3==1.35.71
paramiko==3.3.1
pillow
matplotlib
requests
```
```bash
pip install -r requirements.txt
```
- `boto3`: el propio SDK de AWS.
- `paramiko`: cliente SSH en Python — se usa junto a Boto3 para conectarse por SSH a una instancia EC2 recién creada (Boto3 no incluye cliente SSH propio, son dos librerías complementarias).
- `pillow` / `matplotlib`: manipulación y visualización de imágenes, para los ejercicios con Amazon Textract (extracción de texto de documentos/imágenes).
- `requests`: llamadas HTTP genéricas, para ejercicios que combinan Boto3 con APIs externas.

### Configurar las credenciales
Dos formas de darle a Boto3 con qué credenciales operar:
1. **Consola de IAM**: crear una clave de acceso para tu usuario (o uno nuevo) en *Credenciales de seguridad*, y guardarla en `~/.aws/credentials`:
   ```ini
   [default]
   aws_access_key_id = <tu-clave-de-acceso>
   aws_secret_access_key = <tu-clave-secreta>
   ```
2. **AWS CLI**: ejecutar `aws configure` y seguir el asistente — escribe el mismo archivo por ti.

Verificar que todo funciona:
```python
import boto3
print(boto3.__version__)

s3 = boto3.resource('s3')
list(s3.buckets.all())  # si no da error, las credenciales son válidas
```
- Errores típicos al probarlo:
  - `ClientError` de clave inválida → revisar `~/.aws/credentials`.
  - `ClientError: AccessDenied` en `ListBuckets` → la clave es válida pero el usuario IAM no tiene permisos; hay que añadir el usuario a un grupo IAM con una política adecuada (ej. `AmazonS3FullAccess` para poder listar/usar S3).

Listar todos los servicios disponibles en el SDK instalado:
```python
session = boto3.Session()
services = session.get_available_services()
print(services)  # más de 300 nombres de servicio, ej. 's3', 'ec2', 'dynamodb', 'lambda'...
```

### IAM con Boto3
```python
iam_client = boto3.client('iam')

# Crear un usuario IAM
iam_client.create_user(UserName='<nombre-usuario>')

# Listar usuarios
response = iam_client.list_users()
for user in response['Users']:
    print(user['UserName'])

# Listar grupos
response = iam_client.list_groups()
for group in response['Groups']:
    print(group['GroupName'])

# Borrar un usuario
iam_client.delete_user(UserName='<nombre-usuario>')
```
- `delete_user` falla si el usuario todavía tiene recursos adjuntos (claves de acceso, políticas, pertenencia a grupos...) — hay que desvincularlos primero, AWS no borra en cascada.

### Client, Paginadores, Waiters y Resource — cuándo usar cada uno
| Concepto | Qué es |
|---|---|
| **Client** | Acceso de bajo nivel, llamadas 1:1 con la API — control fino sobre parámetros |
| **Paginator** | Itera automáticamente resultados paginados de la API (ej. listados muy largos) sin gestionar tokens a mano |
| **Waiter** | Sondea la API hasta que un recurso alcanza cierto estado (ej. esperar a que una EC2 esté "running") |
| **Resource** | Interfaz orientada a objetos y de más alto nivel sobre el mismo servicio — más cómoda, no cubre el 100% de las operaciones |

Flujo recomendado para cualquier servicio nuevo: comprobar que está disponible en Boto3 → añadir el permiso IAM correspondiente al usuario/grupo → mirar en la documentación qué funciones ofrece → elegir client o resource según el caso.

### S3 con Boto3
```python
client = boto3.client('s3')

# Crear un bucket (el nombre debe ser único a nivel global)
client.create_bucket(
    ACL='private', Bucket='<bucket-name>',
    CreateBucketConfiguration={'LocationConstraint': '<region>'}
)

# Subir / descargar / borrar un objeto
client.upload_file(Filename='<archivo-local>', Bucket='<bucket-name>', Key='<key-en-bucket>')
client.download_file(Bucket='<bucket-name>', Key='<key-en-bucket>', Filename='<archivo-local-destino>')
client.delete_object(Bucket='<bucket-name>', Key='<key-en-bucket>')

# Vía resource: instanciar un bucket concreto y operar sobre él
s3 = boto3.resource('s3')
bucket = s3.Bucket('<bucket-name>')
bucket.upload_file(Filename='<archivo-local>', Key='<key-en-bucket>')
for obj in bucket.objects.all():
    print(obj.key)

# Filtrar objetos por prefijo (ej. simular listar una "carpeta")
list(bucket.objects.filter(Prefix='<prefijo>'))

# Paginar un listado grande de objetos
paginator = client.get_paginator('list_objects')
pages = paginator.paginate(Bucket=bucket.name)
for item in pages.search('Contents'):
    print(item['Key'])

# Consultar cifrado y ACL del bucket
client.get_bucket_encryption(Bucket='<bucket-name>')
client.get_bucket_acl(Bucket='<bucket-name>')

# Configurar una regla de lifecycle (expira objetos a los 180 días)
client.put_bucket_lifecycle_configuration(
    Bucket='<bucket-name>',
    LifecycleConfiguration={
        'Rules': [{'Expiration': {'Days': 180}, 'Prefix': '', 'Status': 'Enabled'}]
    }
)

# Vaciar el bucket antes de borrarlo, y borrarlo
bucket.objects.all().delete()
client.delete_bucket(Bucket='<bucket-name>')
```
- Subir un archivo con una key tipo `<carpeta>/<archivo>` "simula" una carpeta — en S3 no existen directorios reales, ver [Módulo 13](../modulo-13-s3/README.md).
- `delete_bucket` falla si el bucket no está vacío — hay que borrar todos los objetos primero (`bucket.objects.all().delete()`), el mismo comportamiento visto por CLI en el Módulo 13.

### Cost Explorer con Boto3
- El cliente de Cost Explorer (`ce`) exige indicar región explícitamente (`us-east-1`) — sin ella, Boto3 lanza una excepción.
- Necesita un permiso IAM propio (`ce:*`), no viene incluido en políticas generales como `AmazonS3FullAccess`.
```python
from datetime import datetime, timedelta

ce_client = boto3.client('ce', region_name='us-east-1')

end_date = datetime.now().strftime('%Y-%m-%d')
start_date = (datetime.now() - timedelta(days=90)).strftime('%Y-%m-%d')

# Coste y uso de los últimos 90 días, agrupado por mes
response = ce_client.get_cost_and_usage(
    TimePeriod={'Start': start_date, 'End': end_date},
    Granularity='MONTHLY',
    Metrics=['UnblendedCost', 'UsageQuantity']
)

# Qué servicios han generado coste/uso en ese periodo
response = ce_client.get_dimension_values(
    TimePeriod={'Start': start_date, 'End': end_date},
    Dimension='SERVICE'
)

# Coste agrupado por servicio (útil para ver qué se está llevando el gasto)
response = ce_client.get_cost_and_usage(
    TimePeriod={'Start': start_date, 'End': end_date},
    Granularity='MONTHLY',
    Metrics=['UnblendedCost'],
    GroupBy=[{'Type': 'DIMENSION', 'Key': 'SERVICE'}]
)

# Coste previsto para el próximo mes
tomorrow = (datetime.now() + timedelta(days=1)).strftime('%Y-%m-%d')
next_month = (datetime.now() + timedelta(days=31)).strftime('%Y-%m-%d')
forecast = ce_client.get_cost_forecast(
    TimePeriod={'Start': tomorrow, 'End': next_month},
    Metric='UNBLENDED_COST',
    Granularity='MONTHLY'
)
print(forecast['Total']['Amount'])
```

### EC2 con Boto3
```python
ec2 = boto3.resource('ec2', region_name='<region>')

# Crear una instancia (vía resource; MinCount/MaxCount e ImageId son obligatorios)
instances = ec2.create_instances(
    ImageId='<ami-id>', MinCount=1, MaxCount=1, InstanceType='t2.nano'
)
instance_id = instances[0].id

# Listar todas las instancias de la cuenta/región
list(ec2.instances.all())

# Detalle completo de una instancia (vía client)
client = boto3.client('ec2', region_name='<region>')
client.describe_instances(InstanceIds=[instance_id])

# Detener / iniciar / terminar una instancia
client.stop_instances(InstanceIds=[instance_id])
client.start_instances(InstanceIds=[instance_id])
client.terminate_instances(InstanceIds=[instance_id])
```
- Si `create_instances` devuelve `UnauthorizedOperation`, falta el permiso EC2 en la política IAM del usuario/grupo.
- **Detener** (`stop_instances`) no es lo mismo que **terminar** (`terminate_instances`): una instancia detenida conserva su volumen EBS (y su coste de almacenamiento) y se puede volver a arrancar con el mismo ID; una instancia terminada se destruye junto con ese almacenamiento.

### Conectar por SSH a una instancia EC2 (Boto3 + Paramiko)
Boto3 no incluye cliente SSH — para conectarse a la instancia recién creada se combina con `paramiko`.
```python
import paramiko

# Generar un par de claves SSH (la privada solo se devuelve una vez, en la creación)
key_pair = client.create_key_pair(KeyName='<key-pair-name>')
with open('<archivo-clave.pem>', 'w') as f:
    f.write(key_pair['KeyMaterial'])

# La instancia necesita un Security Group que permita el puerto 22, y la KeyName asignada
security_groups = client.describe_security_groups()

instances = ec2.create_instances(
    ImageId='<ami-id>', MinCount=1, MaxCount=1, InstanceType='t2.nano',
    KeyName='<key-pair-name>',
    SecurityGroupIds=['<security-group-id>']
)

# Conectar con Paramiko usando la clave privada generada
ssh = paramiko.SSHClient()
ssh.set_missing_host_key_policy(paramiko.AutoAddPolicy())
private_key = paramiko.RSAKey(filename='<archivo-clave.pem>')

ip = client.describe_instances(InstanceIds=[instances[0].id])['Reservations'][0]['Instances'][0]['PublicIpAddress']
ssh.connect(hostname=ip, username='ubuntu', pkey=private_key)

stdin, stdout, stderr = ssh.exec_command('ls')
print(stdout.read())
ssh.close()
```
- La clave privada (`KeyMaterial`) solo se devuelve en el momento de crear el par de claves — si se pierde el archivo `.pem`, no hay forma de recuperarla desde AWS, hay que generar un par nuevo.
- Sin un Security Group que permita el puerto 22 (SSH) hacia la instancia, la conexión con Paramiko se queda colgada o falla por timeout — no es un problema de la librería, es que el puerto está bloqueado por defecto.
- El nombre de usuario SSH depende de la AMI (`ubuntu` para imágenes Ubuntu, `ec2-user` para Amazon Linux...), no es un valor fijo de Boto3/Paramiko.

### DynamoDB con Boto3
```python
client_dynamodb = boto3.client('dynamodb', region_name='<region>')

# Crear una tabla (modo aprovisionado: hace falta indicar RCU/WCU)
client_dynamodb.create_table(
    TableName='<table-name>',
    KeySchema=[{'AttributeName': '<clave-particion>', 'KeyType': 'HASH'}],
    AttributeDefinitions=[{'AttributeName': '<clave-particion>', 'AttributeType': 'S'}],
    ProvisionedThroughput={'ReadCapacityUnits': 5, 'WriteCapacityUnits': 5}
)

# Insertar un elemento (vía resource, más cómodo que el client para esto)
dynamodb = boto3.resource('dynamodb', region_name='<region>')
table = dynamodb.Table('<table-name>')
table.put_item(Item={'<clave-particion>': '<valor>', '<atributo>': '<valor>'})

# Escanear toda la tabla
response = table.scan()
for item in response.get('Items', []):
    print(item)

# Borrar la tabla (vía resource o vía client)
table.delete()
# client_dynamodb.delete_table(TableName='<table-name>')
```
- Todo atributo que forme parte del `KeySchema` debe declararse también en `AttributeDefinitions` — el resto de atributos del ítem no necesitan declararse de antemano (DynamoDB no tiene esquema fijo fuera de la clave).
- En modo aprovisionado, `create_table` exige `ProvisionedThroughput` (RCU/WCU) — en modo bajo demanda no hace falta, ver [Módulo 30](../modulo-30-bases-de-datos-no-relacionales/README.md) para las fórmulas de capacidad.

### Ejercicio práctico: dataset de Netflix en DynamoDB
Ejercicio del curso: cargar el CSV público de [títulos de Netflix](https://www.kaggle.com/datasets/shivamb/netflix-shows) en una tabla con clave compuesta y un GSI, y resolver varias consultas.

```python
# Tabla con clave compuesta: show_id (partición) + release_year (ordenación)
attributes = [
    {'AttributeName': '<clave-particion>', 'AttributeType': 'S'},
    {'AttributeName': '<clave-ordenacion>', 'AttributeType': 'N'},
    {'AttributeName': '<atributo-gsi>', 'AttributeType': 'S'},
]
key_schema = [
    {'AttributeName': '<clave-particion>', 'KeyType': 'HASH'},
    {'AttributeName': '<clave-ordenacion>', 'KeyType': 'RANGE'},
]

client.create_table(
    TableName='<table-name>',
    AttributeDefinitions=attributes,
    KeySchema=key_schema,
    ProvisionedThroughput={'ReadCapacityUnits': 10, 'WriteCapacityUnits': 10},
    GlobalSecondaryIndexes=[{
        'IndexName': '<index-name>',
        'KeySchema': [{'AttributeName': '<atributo-gsi>', 'KeyType': 'HASH'}],
        'Projection': {'ProjectionType': 'ALL'},
        'ProvisionedThroughput': {'ReadCapacityUnits': 10, 'WriteCapacityUnits': 10}
    }]
)

# Carga masiva por lotes de 25 elementos (límite de batch_write_item)
items_to_upload = []
for item in data_list:
    put_request = {'PutRequest': {'Item': {...}}}  # tipar cada valor como {'S': ...} o {'N': ...}
    items_to_upload.append(put_request)
    if len(items_to_upload) == 25:
        client.batch_write_item(RequestItems={'<table-name>': items_to_upload})
        items_to_upload = []

# Consultar por clave de partición
client.query(
    TableName='<table-name>',
    KeyConditionExpression='<clave-particion> = :id',
    ExpressionAttributeValues={':id': {'S': '<valor>'}}
)

# Consultar por el GSI (solo admite igualdad sobre su clave hash)
client.query(
    TableName='<table-name>', IndexName='<index-name>',
    KeyConditionExpression='<atributo-gsi> = :valor',
    ExpressionAttributeValues={':valor': {'S': '<valor>'}}
)

# Filtrar por dos condiciones (partición del GSI + rango) — un scan, con paginación
items = []
response = client.scan(
    TableName='<table-name>',
    FilterExpression='<clave-ordenacion> >= :anio AND <atributo-gsi> = :pais',
    ExpressionAttributeValues={':anio': {'N': '2021'}, ':pais': {'S': '<valor>'}}
)
items.extend(response['Items'])
while 'LastEvaluatedKey' in response:
    response = client.scan(
        TableName='<table-name>',
        FilterExpression='<clave-ordenacion> >= :anio AND <atributo-gsi> = :pais',
        ExpressionAttributeValues={':anio': {'N': '2021'}, ':pais': {'S': '<valor>'}},
        ExclusiveStartKey=response['LastEvaluatedKey']
    )
    items.extend(response['Items'])
```
- Un `Query` sobre un GSI **no admite combinar** su clave hash con una condición de rango sobre otro atributo (ej. "país X **y** año ≥ 2021") — para eso hace falta un `Scan` con `FilterExpression`, aceptando que es menos eficiente. Es la limitación real detrás de "modela la tabla según tus consultas" del [Módulo 30](../modulo-30-bases-de-datos-no-relacionales/README.md).
- `Scan` pagina los resultados — mientras la respuesta incluya `LastEvaluatedKey`, hay que repetir la llamada pasando ese valor como `ExclusiveStartKey` para no perder elementos.
- DynamoDB no acepta campos vacíos en tipos que lo requieren — el ejercicio sustituye valores vacíos del CSV por un string `"None"` antes de subirlos, en vez de omitir el atributo.

### Lambda con Boto3
El código de la función es un archivo `.py` normal, cargado y empaquetado en el `.zip` que se envía a `create_function` — nada especial más allá del handler:
```python
# hello.py — la función "Hola Mundo" del ejercicio
def lambda_handler(event, context):
    return "Hola Mundo"
```
```python
import json, io, zipfile

iam_client = boto3.client('iam', region_name='<region>')
lambda_client = boto3.client('lambda', region_name='<region>')

# Rol de ejecución mínimo (solo logs) — el trust policy debe permitir que lambda.amazonaws.com lo asuma
role_response = iam_client.create_role(
    RoleName='<role-name>',
    AssumeRolePolicyDocument=json.dumps({
        "Version": "2012-10-17",
        "Statement": [{"Effect": "Allow", "Principal": {"Service": "lambda.amazonaws.com"}, "Action": "sts:AssumeRole"}]
    })
)
iam_client.put_role_policy(
    RoleName='<role-name>', PolicyName='<policy-name>',
    PolicyDocument=json.dumps({
        "Version": "2012-10-17",
        "Statement": [{"Effect": "Allow", "Action": ["logs:CreateLogGroup", "logs:CreateLogStream", "logs:PutLogEvents"], "Resource": "arn:aws:logs:*:*:*"}]
    })
)
role_arn = role_response['Role']['Arn']

# Empaquetar el código en un .zip en memoria y crear la función
with open('<archivo.py>', 'r') as f:
    function_code = f.read()

with io.BytesIO() as deployment_package:
    with zipfile.ZipFile(deployment_package, 'w') as zipf:
        zipf.writestr('lambda_function.py', function_code)
    create_function_response = lambda_client.create_function(
        FunctionName='<function-name>', Runtime='python3.12',
        Role=role_arn, Handler='lambda_function.<nombre-funcion-handler>',
        Code={'ZipFile': deployment_package.getvalue()}
    )

# Invocar la función y leer el resultado
invoke_response = lambda_client.invoke(FunctionName='<function-name>')
print(invoke_response['Payload'].read().decode('utf-8'))

# Borrar la función
lambda_client.delete_function(FunctionName='<function-name>')
```
- `Handler` sigue el formato `<archivo-sin-extension>.<nombre-de-la-función>` — el nombre de la función Python es libre, no tiene que llamarse `lambda_handler`.
- El código se sube como bytes de un `.zip` (`ZipFile`), o alternativamente referenciando un objeto en S3 (`S3Bucket`/`S3Key`) o una imagen de contenedor (`ImageUri`).

#### Disparador S3 → Lambda
El código de la función que procesa el evento de S3 lee la key del objeto subido desde el propio evento:
```python
# trigger.py — handler invocado al subir un objeto al bucket
def lambda_trigger(event, context):
    s3_object_key = event['Records'][0]['s3']['object']['key']
    print(f"File uploaded: {s3_object_key}")
    return s3_object_key
```
```python
s3_client = boto3.client('s3', region_name='<region>')

# El rol de ejecución de esta Lambda necesita además permiso de lectura sobre el bucket (s3:GetObject)

# 1) Dar permiso a S3 para invocar la función (resource policy de Lambda)
lambda_client.add_permission(
    FunctionName='<function-name>', StatementId='<statement-id>',
    Action='lambda:InvokeFunction', Principal='s3.amazonaws.com',
    SourceArn='arn:aws:s3:::<bucket-name>'
)

# 2) Configurar la notificación de eventos del bucket hacia la función
s3_client.put_bucket_notification_configuration(
    Bucket='<bucket-name>',
    NotificationConfiguration={
        'LambdaFunctionConfigurations': [{
            'LambdaFunctionArn': create_function_response['FunctionArn'],
            'Events': ['s3:ObjectCreated:*'],
        }]
    }
)
```
- Es el mismo patrón de dos pasos visto en el [Módulo 15](../modulo-15-lambda/README.md) para invocaciones cross-service: primero la resource policy de Lambda (quién puede invocar), luego la configuración del origen del evento (S3, en este caso) — sin el primer paso, S3 no tiene permiso aunque la notificación esté bien configurada.

#### Leer los logs de una función en CloudWatch
```python
logs_client = boto3.client('logs', region_name='<region>')

log_streams = logs_client.describe_log_streams(
    logGroupName=f'/aws/lambda/<function-name>',
    orderBy='LastEventTime', descending=True
)['logStreams']

logs_client.get_log_events(
    logGroupName=f'/aws/lambda/<function-name>',
    logStreamName=log_streams[0]['logStreamName']
)
```
- El log group de una función Lambda sigue siempre el mismo patrón de nombre: `/aws/lambda/<nombre-de-la-función>`.

### Amazon Textract con Boto3
Extrae texto, formularios y tablas de documentos escaneados (imagen o PDF).
```python
textract_client = boto3.client('textract', region_name='<region>')
document_bytes = open('<documento.pdf>', 'rb').read()

# Detectar texto plano (líneas)
response = textract_client.detect_document_text(Document={'Bytes': document_bytes})
for item in response['Blocks']:
    if item['BlockType'] == 'LINE':
        print(item['Text'])

# Analizar tablas del documento
response = textract_client.analyze_document(
    Document={'Bytes': document_bytes}, FeatureTypes=['TABLES']
)
block_id_map = {block['Id']: block for block in response['Blocks']}
for item in response['Blocks']:
    if item['BlockType'] == 'CELL':
        line = ""
        for relationship in item.get('Relationships', []):
            for related_id in relationship['Ids']:
                related_block = block_id_map.get(related_id)
                if related_block:
                    line += " " + related_block.get('Text', '')
        print(item['RowIndex'], item['ColumnIndex'], line)

# Preguntar directamente sobre el contenido del documento
response = textract_client.analyze_document(
    Document={'Bytes': document_bytes},
    FeatureTypes=['QUERIES'],
    QueriesConfig={'Queries': [{'Text': '<pregunta-en-lenguaje-natural>'}]}
)
for block in response['Blocks']:
    if block['BlockType'] == 'QUERY_RESULT':
        print(block)
```
- `Document` acepta `Bytes` (el archivo cargado en memoria) o una referencia `S3Object` — no hace falta subir el archivo primero si ya lo tienes en local.
- `FeatureTypes` admite combinar `TABLES`, `FORMS`, `QUERIES` y `SIGNATURES` en la misma llamada a `analyze_document`.
- Para reconstruir una tabla hay que cruzar manualmente los bloques `CELL` con sus `Relationships` — Textract no devuelve la tabla ya montada como texto, da bloques con relaciones que hay que recorrer (`RowIndex`/`ColumnIndex` + los `Id` de las celdas hijas).
- Las consultas (`QUERIES`) no aceptan caracteres especiales en el texto de la pregunta.

## Comandos clave

```python
import boto3

# Cliente de bajo nivel: llamadas 1:1 con la API de AWS
s3 = boto3.client('s3', region_name='<region>')
response = s3.list_buckets()
for bucket in response['Buckets']:
    print(bucket['Name'])

# Recurso de alto nivel: interfaz orientada a objetos sobre el mismo servicio
s3_resource = boto3.resource('s3')
for bucket in s3_resource.buckets.all():
    print(bucket.name)

# Ejemplo: lanzar una instancia EC2
ec2 = boto3.client('ec2', region_name='<region>')
ec2.run_instances(
    ImageId='<ami-id>',
    InstanceType='t2.micro',
    MinCount=1,
    MaxCount=1
)

# Ejemplo: leer un elemento de DynamoDB
dynamodb = boto3.resource('dynamodb', region_name='<region>')
table = dynamodb.Table('<table-name>')
item = table.get_item(Key={'<clave-particion>': '<valor>'})
print(item.get('Item'))
```

## Notas y gotchas

- Boto3 ofrece dos formas de hablar con un servicio: `client` (llamadas de bajo nivel, calcadas de la API REST) y `resource` (envoltorio orientado a objetos, más cómodo pero no disponible para todos los servicios) — no son intercambiables 1:1, hay que elegir según el servicio y la preferencia de estilo.
- Las credenciales no se hardcodean en el código: Boto3 las resuelve automáticamente (variables de entorno, `~/.aws/credentials`, rol IAM de la instancia/Lambda...) siguiendo la misma cadena de resolución que usa la CLI.
- Como la CLI usa el mismo SDK por debajo, cualquier comportamiento raro reproducible con `aws <servicio> <accion>` normalmente también aparece igual desde Boto3 — es útil para depurar: si el comando CLI falla igual, el problema no es del SDK ni del lenguaje.

## Recursos

- https://docs.aws.amazon.com/boto3/latest/index.html
- https://docs.aws.amazon.com/sdkref/latest/guide/overview.html
