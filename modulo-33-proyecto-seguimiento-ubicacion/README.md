# Módulo 33 — Proyecto práctico: Seguimiento de ubicación (IoT)

## Resumen

Proyecto práctico de IoT en tiempo real: un dispositivo físico (Raspberry Pi + módulo GPS) envía su posición a AWS, y un cliente recibe las actualizaciones de ubicación en tiempo real vía WebSocket.

### Hardware
- **Raspberry Pi 3B+**: Wi-Fi doble banda, Bluetooth, Ethernet, 40 pines GPIO. Necesita al menos 5V de alimentación estable.
- **Módulo GPS NEO 6M**: conectado a la Raspberry por los pines VCC/GND/TX↔RX. Necesita cielo despejado (exterior) para funcionar bien — la lluvia y los edificios/montañas degradan la señal.
- Para precisión completa (lat/lon/altitud) hace falta al menos 4 satélites; con 3 se puede calcular lat/lon pero con menos precisión.
- Estados de arranque del GPS: **frío** (primera vez en una ubicación nueva, puede tardar minutos), **tibio** (apagado breve, datos recientes, segundos-minutos), **caliente** (apagado/encendido en el mismo sitio en pocos minutos, reconexión casi inmediata).

### Arquitectura en AWS
1. El dispositivo IoT publica su posición (lat/lon) al topic MQTT **`fleet/location`** en **AWS IoT Core**.
2. Una **regla de IoT Core** (`SELECT * FROM 'fleet/location'`) enruta esos mensajes a la función Lambda **`GuardarUbicacionFlota`** (Python 3.12), que procesa y almacena el dato.
3. **Amazon Location Service** (un **Tracker** con `EventBridgeEnabled: true`) registra la posición — al tener EventBridge habilitado en el propio tracker, cada actualización de posición emite un evento automáticamente, sin código adicional para publicarlo.
4. Ese evento en **Amazon EventBridge** dispara la función Lambda **`EnviarClienteLocalizacionFlota`**, que envía la actualización al cliente conectado vía **WebSocket** (Amazon API Gateway).
5. Dos Lambdas más gestionan el ciclo de vida de la conexión WebSocket: **`GuardarFlotaWebSocketConnectionId`** (al conectar) y **`BorrarFlotaConexionWebsocket`** (al desconectar), sobre una tabla **DynamoDB** de conexiones activas.
6. **Amazon Cognito** gestiona la identidad/autorización de los clientes que consultan la API.
7. Toda la infraestructura se despliega con **AWS CloudFormation** como un stack anidado: una plantilla raíz (`main-stack.yaml`) que referencia plantillas hijas por componente (`iot-core-stack.yaml`, `lambda-stack.yaml`, `location-service-stack.yaml`, `dynamodb-stack.yaml`, `api-gateway-stack.yaml`, `cognito-identity-pool.yaml`) — el mismo patrón de stacks anidados visto en el [Módulo 26](../modulo-26-cloudformation/README.md).
8. **CloudWatch** monitoriza el conjunto.

### El script de la Raspberry Pi (`ubicacion.py`)
- Lee el módulo GPS por el puerto serie (`/dev/ttyAMA0`, 9600 baudios) y parsea las sentencias NMEA con `pynmea2`, quedándose con las de tipo `$GPRMC` (posición + hora).
- Publica cada posición válida (lat/lon distintos de 0) como JSON al topic `fleet/location`, vía `paho-mqtt`, en un hilo separado del bucle principal del cliente MQTT (`client.loop_forever()`).
- La conexión a IoT Core usa **TLS mutuo** (`client.tls_set` con el CA de Amazon, el certificado y la clave privada del dispositivo) sobre el puerto **8883** — no hay usuario/contraseña, la identidad la da el certificado.

### Interfaz de cliente
- El proyecto incluye una UI web simple (HTML/CSS/JS) que se conecta al WebSocket de API Gateway para pintar la posición en tiempo real, más una página `simulador.html` para simular movimiento sin depender del hardware real — útil para probar el flujo completo sin tener la Raspberry Pi a mano.

### Seguridad del dispositivo
- La Raspberry Pi se autentica frente a AWS IoT Core con **certificados X.509** (par de claves pública/privada).
- Flujo: generar/extraer certificado y claves desde IoT Core → cargarlos en la Raspberry Pi → el dispositivo los usa para autenticar cada conexión MQTT.
- Cada certificado se adjunta a una **IoT Policy** que define qué acciones (`iot:Publish`, `iot:Subscribe`...) puede hacer ese dispositivo concreto — principio de mínimo privilegio también a nivel de dispositivo.

### Coste (dentro de capa gratuita)
| Servicio | Capa gratuita |
|---|---|
| AWS Lambda | 1M solicitudes/mes, gratis para siempre |
| Amazon CloudWatch | 10 alarmas/métricas personalizadas, gratis para siempre |
| AWS IoT Core | 500.000 mensajes/mes, 12 meses gratis |
| Amazon API Gateway | 1M llamadas/mes, 12 meses gratis |
| Amazon Location Service | 200.000 escrituras/evaluaciones de posición, 3 meses gratis |
| Amazon EventBridge | 1M eventos/mes, gratis por defecto |
| Amazon DynamoDB | 25 GB + 25 RCU/WCU, gratis para siempre |
| AWS CloudFormation | 1000 operaciones/mes, gratis para siempre |
| Amazon Cognito | 50.000 MAU, gratis para siempre (con condiciones a los 12 meses) |

## Comandos clave

```bash
# Registrar el dispositivo como "Thing" en IoT Core
aws iot create-thing --thing-name <thing-name>

# Generar un certificado X.509 + par de claves, y activarlo
aws iot create-keys-and-certificate --set-as-active

# Asociar el certificado al Thing
aws iot attach-thing-principal --thing-name <thing-name> --principal <certificate-arn>

# Crear una política IoT (qué puede hacer el dispositivo)
aws iot create-policy \
  --policy-name <policy-name> \
  --policy-document '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Action":"iot:Publish","Resource":"*"}]}'

# Adjuntar la política al certificado del dispositivo
aws iot attach-policy --policy-name <policy-name> --target <certificate-arn>

# Crear una regla que enrute mensajes MQTT a una función Lambda
aws iot create-topic-rule \
  --rule-name <rule-name> \
  --topic-rule-payload '{"sql":"SELECT * FROM '\''<topic-mqtt>'\''","actions":[{"lambda":{"functionArn":"<lambda-arn>"}}]}'

# Crear un rastreador en Amazon Location Service
aws location create-tracker --tracker-name <tracker-name>

# Actualizar la posición de un dispositivo en el rastreador
aws location batch-update-device-position \
  --tracker-name <tracker-name> \
  --updates '[{"DeviceId":"<device-id>","Position":[<longitud>,<latitud>],"SampleTime":"<timestamp-iso8601>"}]'
```

## Notas y gotchas

- En `location batch-update-device-position`, la posición se especifica como `[longitud, latitud]` — el orden invertido respecto a como solemos decir "lat/lon" en voz alta es una fuente típica de errores.
- El GPS necesita **exterior real** o al menos estar muy cerca de una ventana — probarlo en interior sin línea de vista al cielo da resultados inconsistentes que parecen un fallo del hardware pero no lo son.
- El "arranque frío" del GPS (varios minutos sin señal) es esperable la primera vez o tras mucho tiempo apagado — no es un fallo del módulo, conviene tenerlo en cuenta al probar el proyecto por primera vez.
- Cada certificado X.509 va ligado a una policy — sin adjuntar la policy al certificado, el dispositivo se autentica pero no puede publicar/suscribirse a nada (error de autorización, no de autenticación).
- La combinación EventBridge + WebSocket (vía API Gateway) es el patrón estándar para "notificar cambios a un cliente en tiempo real sin que el cliente esté haciendo polling constante" — reutilizable fuera de este proyecto concreto.
- Habilitar `EventBridgeEnabled: true` directamente en el Tracker de Location Service evita tener que publicar el evento de posición a mano desde el código — es el propio servicio quien lo hace, así que si el flujo de notificación falla, conviene revisar primero esa propiedad antes de sospechar del código.
- El simulador web (`simulador.html`) permite probar todo el pipeline (IoT Core → Lambda → Location Service → EventBridge → WebSocket) sin depender de tener la Raspberry Pi y el GPS montados y con buena señal — útil para separar "problema de hardware/GPS" de "problema de arquitectura en AWS".

## Recursos

- https://docs.aws.amazon.com/iot/latest/developerguide/what-is-aws-iot.html
- https://docs.aws.amazon.com/iot/latest/developerguide/x509-client-certs.html
- https://docs.aws.amazon.com/iot/latest/developerguide/iot-rules.html
- https://docs.aws.amazon.com/location/latest/developerguide/what-is.html
