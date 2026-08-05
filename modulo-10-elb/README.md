# Módulo 10 — Elastic Load Balancing (ELB)

## Resumen

### Escalabilidad, elasticidad y alta disponibilidad, sin mezclarlos
Son tres conceptos que se suelen confundir: **escalabilidad** es la capacidad de aguantar más carga (subiendo el hardware de una máquina, o añadiendo más máquinas); **elasticidad** es que, una vez el sistema sabe escalar, lo haga automáticamente según la demanda real, sin que nadie tenga que intervenir a mano; y **alta disponibilidad** es un objetivo distinto — sobrevivir a que se caiga un centro de datos entero, lo cual solo se consigue repartiendo la aplicación en al menos dos Zonas de Disponibilidad. El escalado horizontal (más nodos) necesita piezas extra para funcionar de forma coordinada: un Auto Scaling Group y un Load Balancer, ambos desplegados en varias AZ.

### Para qué sirve un balanceador de carga
Es, en esencia, la puerta de entrada única de tu aplicación: recibe el tráfico y lo reparte entre varias instancias backend, enviándolo solo a las que están respondiendo correctamente. Los motivos habituales para meter uno delante de tu app:
- Repartir la carga entre varios servidores en vez de saturar uno solo.
- Dar a los usuarios un único punto de acceso (una URL/DNS), aunque por debajo haya muchas instancias cambiando constantemente.
- Poder perder una instancia sin que el servicio se caiga.
- Vigilar continuamente qué instancias están sanas.
- Centralizar el cifrado HTTPS en un solo sitio en vez de configurarlo en cada instancia.
- Repartir tráfico entre varias zonas de disponibilidad.

### El ELB es la versión gestionada por AWS
No tienes que instalar ni mantener el software del balanceador — AWS se encarga de que funcione, de actualizarlo y de su propia alta disponibilidad. Hay tres tipos activos hoy (el cuarto, el Classic Load Balancer, se retiró en 2023):

| Tipo | En qué capa de red opera | Protocolos | Cuándo usarlo |
|---|---|---|---|
| **Application LB (ALB)** | Capa 7 (aplicación) | HTTP/HTTPS | Enrutamiento inteligente basado en el contenido de la petición, microservicios, contenedores |
| **Network LB (NLB)** | Capa 4 (transporte) | TCP/UDP | Cuando necesitas rendimiento extremo y la mínima latencia posible, con IP fija |
| **Gateway LB (GWLB)** | Capa 3 (red) | GENEVE | Insertar dispositivos de seguridad de terceros en el camino del tráfico |

### Cómo decide el ALB a dónde mandar cada petición
El ALB puede enrutar tráfico fijándose en distintas partes de la petición HTTP: la ruta de la URL (para separar, por ejemplo, `/clientes` de `/articulos`), el subdominio usado, o incluso parámetros de la query string o cabeceras concretas (como distinguir tráfico móvil de escritorio por una cabecera `Platform`). Cada una de esas reglas termina apuntando a un **Target Group** distinto, que puede vivir dentro de AWS o incluso fuera (on-premises).

### NLB: pensado para volumen y velocidad
Sus Target Groups pueden apuntar a instancias EC2, direcciones IP privadas o incluso a otro ALB por detrás. Soporta health checks por TCP, HTTP o HTTPS. Es el único de los tres que no entra en la capa gratuita de AWS.

### GWLB: seguridad transparente
Se coloca en medio del flujo de tráfico de tu VPC para que pase por dispositivos de terceros (firewalls, sistemas de detección de intrusos) sin que tengas que reconfigurar tus tablas de rutas cada vez — centraliza toda la inspección de tráfico en un solo punto.

### Sticky sessions: cuando un usuario necesita volver siempre al mismo servidor
Solo disponible en el ALB: usando una cookie propia (`AWSALB`, con caducidad de 7 días) el balanceador recuerda a qué instancia mandó a un cliente la primera vez, y lo sigue mandando ahí en peticiones sucesivas. Es útil si tu app guarda estado en memoria local, pero tiene una contrapartida: puede desequilibrar la carga entre instancias, porque unas pueden acabar acumulando más clientes "pegados" que otras.

### Balanceo entre zonas (cross-zone)
Por defecto, cada nodo del balanceador solo reparte tráfico entre las instancias de su propia AZ — activar el balanceo cruzado hace que reparta entre **todas** las instancias registradas, sin importar en qué AZ estén.

| Tipo de balanceador | ¿Viene activado por defecto? | ¿Se paga el tráfico entre AZ? |
|---|---|---|
| ALB | Sí | No |
| NLB / GWLB | No | Sí, si lo activas |

### Cifrado en tránsito: SSL/TLS y ACM
El tráfico entre el usuario y tu Load Balancer se puede cifrar con un certificado TLS (técnicamente TLS ha sustituido a SSL, pero el nombre "SSL" se sigue usando por costumbre). **AWS Certificate Manager (ACM)** se encarga de emitir, renovar y gestionar esos certificados automáticamente — los certificados públicos no tienen coste, y se integran directamente con ELB, API Gateway y CloudFront sin que tengas que subir nada a mano.

Si necesitas servir varios dominios distintos con certificados diferentes desde el mismo listener HTTPS, entra en juego **SNI (Server Name Indication)**: el propio cliente indica qué hostname quiere alcanzar durante el saludo inicial TLS, y el servidor le responde con el certificado correcto para ese dominio. Esto solo funciona en ALB, NLB y CloudFront.

### Health Checks: cómo sabe el balanceador quién está sano
Cada destino (target) pasa por distintos estados a lo largo de su vida:

| Estado | Qué significa |
|---|---|
| `Initial` | Se está registrando todavía |
| `Healthy` | Responde bien, recibe tráfico normal |
| `Unhealthy` | Falló varios checks seguidos, deja de recibir tráfico |
| `Unused` | No está registrado en ningún grupo |
| `Draining` | Se está retirando progresivamente del grupo |
| `Unavailable` | No se le están ejecutando checks |

Los valores por defecto de un health check son: protocolo HTTP, puerto 80, ruta `/`, tiempo de espera de 5 segundos, comprobación cada 30 segundos, 3 respuestas correctas seguidas para pasar a sano y 5 fallos seguidos para pasar a no sano, esperando un código 200 como respuesta válida.

Un detalle importante que conviene recordar: **si absolutamente todas las instancias detrás del balanceador están "unhealthy", el balanceador les sigue mandando tráfico igualmente**, porque no tiene ninguna instancia sana a la que redirigir en su lugar — no hay una opción de "cortar todo el tráfico" automáticamente, así que hay que reaccionar rápido para recuperar al menos una instancia.

### Qué se puede monitorizar
CloudWatch recibe automáticamente, sin configuración adicional, métricas como los códigos de respuesta HTTP (2XX a 5XX), el número de hosts sanos/no sanos, la latencia y el volumen de peticiones.

### Errores comunes y cómo diagnosticarlos
| Error | Qué suele indicar |
|---|---|
| **400 Bad Request** | La petición del cliente está mal formada, el problema viene de fuera |
| **503 Service Unavailable** | No queda ninguna instancia disponible — revisa el número de hosts sanos y si tienes instancias funcionando en todas las AZ |
| **504 Gateway Timeout** | La instancia backend tardó demasiado en responder — revisa la configuración de keep-alive y que el timeout del balanceador sea mayor que el del propio servidor |

## Comandos clave

*(No aplica — configuración vía consola: creación de ALB/NLB, target groups, listeners y health checks.)*

## Notas y gotchas

- Las capas OSI de cada tipo de balanceador es materia clásica de examen: ALB = capa 7, NLB = capa 4, GWLB = capa 3.
- Las sticky sessions solo existen en el ALB, no en el NLB.
- El balanceo cruzado entre AZ viene gratis y activado por defecto en el ALB, pero en NLB/GWLB hay que activarlo a mano y tiene coste.
- Conecta con [[modulo-06-ec2-basico]]: los Security Groups del balanceador y de las instancias EC2 son capas distintas — el SG de las instancias debería permitir tráfico únicamente desde el SG del balanceador, nunca abrirlo directamente a internet.

## Recursos

- https://docs.aws.amazon.com/es_es/elasticloadbalancing/latest/application/target-group-health-checks.html
- https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-cloudwatch-metrics.html
- https://docs.aws.amazon.com/es_es/elasticloadbalancing/latest/classic/ts-elb-error-message.html
