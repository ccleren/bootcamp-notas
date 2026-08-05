# Módulo 02 — Infraestructura global del Cloud de AWS

## Resumen

AWS organiza su infraestructura física en varios niveles geográficos: **Regiones, Zonas de Disponibilidad (AZ), Edge Locations y Local Zones**. Entender esta jerarquía es la base para diseñar arquitecturas resistentes a fallos.

### Regiones: el nivel más alto
Una región es, básicamente, "un área geográfica de AWS" (ej. Irlanda, Frankfurt, N. Virginia) — AWS tiene presencia en más de 40 países y sigue creciendo. Cada recurso que creas (una instancia, una base de datos...) vive dentro de una región concreta.

A la hora de elegir en qué región desplegar algo, entran en juego cuatro factores:
- **Dónde tiene que quedarse legalmente tu dato** — hay normativas que exigen que ciertos datos no salgan de un país o bloque concreto.
- **Qué tan cerca está tu región de tus usuarios reales** — menos distancia física suele traducirse en menos latencia.
- **Si el servicio que necesitas está disponible ahí** — AWS no lanza cada función nueva simultáneamente en todas las regiones; algunas tardan meses en llegar a ciertas zonas.
- **El precio**, que varía de una región a otra para el mismo servicio.

### Zonas de Disponibilidad (AZ): la resiliencia dentro de una región
Dentro de cada región hay varias Zonas de Disponibilidad — normalmente entre 3 y 6 (por ejemplo, en la región de Londres serían `eu-west-2a`, `eu-west-2b`, `eu-west-2c`). Cada AZ es, en la práctica, uno o más centros de datos con su propia electricidad, refrigeración y conexión de red, físicamente separados de las demás AZ de la misma región para que un incendio, apagón o corte de fibra en una no arrastre a las otras. Eso sí, están unidas entre ellas por enlaces de muy baja latencia, así que replicar datos entre AZ no penaliza casi nada el rendimiento.

### Por qué no basta con una sola AZ (ni siquiera con una sola región)
Piensa en una aplicación desplegada en un único centro de datos: si ese centro se cae, tu app se cae con él, sin excusas. El primer paso lógico es repartir esa misma app entre varias AZ de la misma región — así sobrevive a que falle una de ellas. Pero eso no arregla el problema de la distancia: un usuario que está al otro lado del mundo va a seguir notando latencia alta, y si toda la región tuviera un problema grave (algo raro, pero no imposible), la app entera quedaría offline. La solución final es replicar la aplicación en **varias regiones distintas**, normalmente eligiendo las más cercanas a tus grupos de usuarios — así ganas tanto en velocidad percibida como en tolerancia a que una región entera falle.

### Edge Locations
Son puntos de presencia de AWS mucho más numerosos y repartidos que las regiones, pensados para servir contenido (imágenes, vídeo, páginas cacheadas) lo más cerca físicamente posible del usuario final. CloudFront, la CDN de AWS, se apoya en esta red.

### AWS Local Zones
Son como una "sucursal" de una región concreta, colocada físicamente cerca de una gran ciudad para bajar aún más la latencia que la propia región. Por ejemplo, la región de N. Virginia (`us-east-1`) tiene Local Zones en ciudades como Miami, Chicago o Dallas. Se usan sobre todo para aplicaciones muy sensibles a la latencia, migraciones híbridas cloud/on-prem, y casos donde el dato debe quedarse cerca de un sitio concreto.

### ¿Servicio regional o global?
Algunos servicios de AWS existen "por región" (tienes que elegir en cuál desplegarlos) y otros son globales (uno solo, vale para toda la cuenta):

| Regionales (eliges dónde) | Globales (uno para toda la cuenta) |
|---|---|
| Amazon EC2 | AWS IAM |
| AWS Elastic Beanstalk | Amazon Route 53 |
| AWS Lambda | AWS CloudFront |
| Amazon Rekognition | AWS WAF |

### Organizando varias cuentas de AWS
Un patrón habitual es tener una **cuenta de gestión** (con MFA activo y presupuesto configurado) desde la que un usuario administrador gestiona cuentas separadas por entorno — por ejemplo una de uso general y otra dedicada solo a producción.

Un truco práctico si quieres practicar esto sin tener varios correos: Gmail ignora todo lo que pongas tras un `+` en la dirección, así que `tucorreo+cuenta1@gmail.com` y `tucorreo+cuenta2@gmail.com` llegan igualmente a tu bandeja principal, pero AWS los trata como direcciones distintas — perfecto para dar de alta varias cuentas AWS sin crear buzones nuevos.

## Comandos clave

*(No aplica — módulo conceptual.)*

## Notas y gotchas

- El truco del `+` en Gmail es especialmente útil para el módulo de [[modulo-04-organizacion-cloud]], donde se practica con varias cuentas.
- No asumas que un servicio nuevo estará disponible en tu región habitual desde el día uno — conviene comprobarlo antes de diseñar sobre él.

## Recursos

- https://aws.amazon.com/es/about-aws/global-infrastructure/
- https://aws.amazon.com/es/about-aws/global-infrastructure/localzones/locations/
- https://aws.amazon.com/about-aws/global-infrastructure/regional-product-services/
