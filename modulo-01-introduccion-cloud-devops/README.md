# Módulo 01 — Introducción a Cloud Computing y DevOps

## Resumen

### Problema que resuelve el Cloud
Con infraestructura tradicional (centro de datos propio) hay que pagar alquiler, electricidad, refrigeración, mantenimiento; añadir/sustituir hardware lleva tiempo; el escalado es limitado; hay que tener equipo 24/7; y hay que planificar para catástrofes (terremoto, apagón...). El Cloud externaliza todo esto.

### ¿Qué es el Cloud Computing?
Suministro **bajo demanda** de potencia de cálculo, almacenamiento, bases de datos y otros recursos IT, a través de una plataforma con **precio de pago por uso**. Provisionas el tipo y tamaño exacto que necesitas, casi al instante.

### Las 5 características del Cloud Computing
1. **Autoservicio bajo demanda**: provisionas recursos sin interacción humana del proveedor.
2. **Amplio acceso a la red**: accesible desde diversas plataformas cliente.
3. **Alquiler múltiple / pooling de recursos** (multi-tenancy): varios clientes comparten la misma infraestructura física con seguridad y privacidad.
4. **Rápida elasticidad**: se adquieren y liberan recursos automáticamente según demanda.
5. **Servicio medido**: pagas exactamente por lo que usas.

### Modelos de despliegue
- **Privado**: uso exclusivo de una organización, control total, para datos sensibles.
- **Público**: recursos de un proveedor, accesibles vía internet a cualquiera.
- **Híbrido**: parte on-premise + parte cloud.

### Modelos de servicio (IaaS / PaaS / SaaS)
| Modelo | Qué gestiona el proveedor | Ejemplo AWS | Otros |
|---|---|---|---|
| **IaaS** | Redes, servidores, almacenamiento — tú gestionas el resto | Amazon EC2 | Azure VMs, GCP, DigitalOcean |
| **PaaS** | + runtime, gestión de hardware/software subyacente | AWS Elastic Beanstalk | Heroku, Azure App Service, Google App Engine |
| **SaaS** | Todo — producto completo ya ejecutado y mantenido | Amazon QuickSight | Gmail, Dropbox, Zoom |

### Modelo de precios de AWS
Tres pilares de pago por uso: **computación** (tiempo de cómputo), **almacenamiento** (datos guardados) y **transferencia de datos saliente** (la entrada de datos es gratis).

### Modelo de responsabilidad compartida
- **AWS** es responsable de la seguridad **DEL** Cloud (infraestructura física, hardware, red global).
- **Tú (cliente)** eres responsable de la seguridad **DENTRO** del Cloud (configuración de tus recursos, IAM, datos, cifrado, parches del SO en tus instancias).

### Introducción a DevOps y CI/CD
- **DevOps**: metodología que integra desarrollo (Dev) y operaciones (Ops) para acelerar la entrega de software, con mejor colaboración, mayor confiabilidad y escalado más rápido.
- **CI/CD**: enfoque automatizado de desarrollar, probar y desplegar software con frecuencia y de forma segura.
- **Fases del pipeline CI/CD**: `PLAN → CODE → BUILD → TEST → RELEASE → DEPLOY → OPERATE → MONITOR`.

## Comandos clave

*(No aplica — módulo puramente conceptual, sin práctica de CLI.)*

## Notas y gotchas

- AWS es el proveedor líder del mercado (Gartner Magic Quadrant), con 200+ servicios y más de 1M de usuarios activos.
- El **modelo de responsabilidad compartida** es un tema recurrente en el resto del curso (y típico de examen): recuerda siempre separar "seguridad DEL cloud" (AWS) vs "seguridad EN el cloud" (tú).

## Recursos

- https://aws.amazon.com/es/compliance/shared-responsibility-model/
- https://aws.amazon.com/aup/ (política de uso aceptable)
