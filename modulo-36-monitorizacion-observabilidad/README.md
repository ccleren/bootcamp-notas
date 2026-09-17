# Módulo 36 — Monitorización y observabilidad (Prometheus, Grafana, Loki)

## Resumen

### Por qué monitorizar y observar
- Permite conocer el estado/rendimiento de los sistemas en tiempo real y detectar fallos antes de que impacten al usuario.
- Clave en entornos cloud/DevOps/microservicios, donde hay muchas capas y componentes cambiando constantemente.
- Sin monitoreo, ante un error el equipo pierde tiempo revisando servicio por servicio para saber si el problema es de red, base de datos o servidor — con monitoreo, se localiza el fallo directamente.

### Herramientas del ecosistema
| Herramienta | Para qué |
|---|---|
| **Prometheus** | Métricas y alertas basadas en series temporales |
| **Grafana** | Visualización de datos y dashboards (de Prometheus, Loki, CloudWatch...) |
| **Loki** | Gestión de logs, integrada nativamente con Grafana |
| **Alertmanager** | Gestión y envío de alertas generadas por Prometheus |
| **Elasticsearch + Kibana** | Búsqueda/análisis de grandes volúmenes de logs + su visualización |
| **Datadog** | Plataforma comercial de observabilidad todo-en-uno |

### Prometheus
- Herramienta open source de monitoreo y alertas: recoge y almacena métricas consultando endpoints HTTP (scraping), y dispara alertas automáticas según condiciones.
- **Prometheus Server**: recolecta métricas de apps/servicios, las almacena como series temporales, y responde consultas (desde su propia UI o desde Grafana).
- Descubre objetivos (targets) vía integración con Kubernetes/Consul, o listas manuales.
- Qué monitorea (targets): servidores, apps web, bases de datos, servidores web. Qué mide (metrics): CPU, memoria, disco, nº/duración de peticiones, errores/excepciones.

### Configuración básica (`prometheus.yml`)
```yaml
global:
  scrape_interval: 20s #Intervalo de recolección de métricas
  evaluation_interval: 20s #Intervalo de evaluación de reglas

rule_files:
  - "alerts.rules"
  - "metrics.rules"

scrape_configs:
  - job_name: "api_service"
    static_configs:
      - targets: ["localhost:8080"]
  - job_name: "db_exporter"
    scrape_interval: 45s
    scrape_timeout: 10s
    static_configs:
      - targets: ["localhost:9200"]
```
- Un **job** es una tarea que obtiene métricas de un servicio concreto; cada uno puede sobrescribir `scrape_interval`, `metrics_path` (por defecto `/metrics`) y `scheme` (por defecto `http`).

### Desplegar Prometheus con Docker Compose (ejemplo)
Una configuración mínima que hace que Prometheus se monitorice a sí mismo:
```yaml
# prometheus.yml
# Configuración global de Prometheus
# Define cada cuánto tiempo se recolectan y evalúan las métricas
global:
  scrape_interval: 15s      # Intervalo de recolección de métricas
  evaluation_interval: 15s  # Intervalo de evaluación de reglas

scrape_configs:
  # Job para monitorizar Prometheus a sí mismo
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']
```
```yaml
# docker-compose-prometheus.yml
# Versión básica sin Grafana ni node-exporter
services:
  prometheus:
    # Imagen oficial de Prometheus
    image: prom/prometheus:latest
    container_name: prometheus
    ports:
      - "9090:9090"  # Puerto web de Prometheus
    volumes:
      - ./prometheus:/etc/prometheus  # Configuración personalizada
      - prometheus_data:/prometheus   # Datos persistentes
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'  # Ruta del archivo de configuración
      - '--storage.tsdb.path=/prometheus'               # Ruta de almacenamiento de datos
      - '--web.console.libraries=/usr/share/prometheus/console_libraries'
      - '--web.console.templates=/usr/share/prometheus/consoles'
    restart: unless-stopped  # Reinicia el contenedor si se detiene

# Definición de volúmenes persistentes
volumes:
  prometheus_data:
```
```bash
docker-compose -f docker-compose-prometheus.yml up
```
- `./prometheus:/etc/prometheus` monta la carpeta local (donde vive `prometheus.yml`) dentro del contenedor — así Prometheus arranca con la configuración propia en vez de la que trae la imagen por defecto.
- `prometheus_data` es un volumen con nombre (no una carpeta local) — persiste las series temporales aunque se recree el contenedor, sin mezclarlas con el resto del proyecto en disco.
- Con esta config mínima, Prometheus ya tiene algo que scrapear desde el minuto uno (a sí mismo, en `localhost:9090`) — útil para verificar que el contenedor funciona antes de añadir jobs reales.

### Añadir Node Exporter al stack
Se amplía `prometheus.yml` con un nuevo job para el Node Exporter (métricas del host: CPU, RAM, disco):
```yaml
scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  # Job para monitorizar node-exporter (puedes añadirlo más adelante)
  - job_name: 'node'
    static_configs:
      - targets: ['node-exporter:9100']
```
- El target es `node-exporter:9100`, no `localhost:9100` — dentro de la red de Docker Compose, cada servicio resuelve por su **nombre de servicio**, no por `localhost`.

Y un `docker-compose` que añade el contenedor de Node Exporter junto a Prometheus:
```yaml
# docker-compose-prometheus-node-exporter.yml
# Archivo docker-compose solo con Prometheus y node-exporter
# Versión básica para monitorización del sistema

services:
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus:/etc/prometheus
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--web.console.libraries=/usr/share/prometheus/console_libraries'
      - '--web.console.templates=/usr/share/prometheus/consoles'
    restart: unless-stopped
    depends_on:
      - node-exporter  # Espera a que node-exporter esté listo

  node-exporter:
    # Imagen oficial de node-exporter para métricas del sistema
    image: prom/node-exporter:latest
    container_name: node-exporter
    ports:
      - "9100:9100"  # Puerto para métricas del sistema
    volumes:
      - /proc:/host/proc:ro  # Métricas de procesos
      - /sys:/host/sys:ro    # Métricas del sistema
      - /:/rootfs:ro         # Métricas del sistema de archivos
    command:
      - '--path.procfs=/host/proc'
      - '--path.sysfs=/host/sys'
      - '--path.rootfs=/rootfs'
      - '--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)'
    restart: unless-stopped

volumes:
  prometheus_data:
```
- Node Exporter necesita ver el sistema de archivos **real del host** para medirlo — de ahí montar `/proc`, `/sys` y `/` (como `/rootfs`) en modo solo lectura (`:ro`) dentro del contenedor, en vez de usar sus propios `/proc`/`/sys` de contenedor.
- `$$` en `mount-points-exclude` escapa el `$` para Docker Compose (que si no, interpretaría `$(` como una variable) — el patrón real que recibe Node Exporter es `^/(sys|proc|dev|host|etc)($|/)`.
- `depends_on` solo controla el **orden de arranque** de los contenedores, no espera a que Node Exporter esté realmente listo para servir métricas — si Prometheus falla el primer scrape por estar node-exporter aún iniciando, simplemente lo reintentará en el siguiente `scrape_interval`.

Para levantarlo en segundo plano, sin quedarse mostrando los logs en la terminal:
```bash
docker-compose -f docker-compose-prometheus-node-exporter.yml up -d
```
- `-d` (detached) arranca los contenedores en background y devuelve el control de la terminal — sin él, `docker-compose up` se queda "enganchado" mostrando los logs de todos los servicios en tiempo real hasta que se interrumpe con Ctrl+C (lo cual además detiene los contenedores).

### PromQL (lenguaje de consulta)
- Permite consultar y procesar las métricas en tiempo real, desde el propio objetivo, la UI de Prometheus o Grafana.
```promql
# Todas las peticiones HTTP excepto errores 4xx
http_requests_total{status!~"4.."}

# Tasa promedio de peticiones en ventanas de 5 min, durante los últimos 30 min
rate(http_requests_total[5m])[30m:]

# % de memoria usada del host (métrica de Node Exporter)
(1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100

# Tráfico de red recibido, en bytes/segundo (por interfaz)
rate(node_network_receive_bytes_total[5m])

# Tráfico de red transmitido, en bytes/segundo (por interfaz)
rate(node_network_transmit_bytes_total[5m])
```
- `node_memory_MemAvailable_bytes` / `node_memory_MemTotal_bytes` son métricas que expone **Node Exporter** — solo aparecen si el job `node` del `scrape_configs` está activo y accesible.
- `node_network_receive_bytes_total` / `node_network_transmit_bytes_total` son contadores que **siempre crecen**; `rate(...[5m])` los convierte en bytes/segundo promediados sobre los últimos 5 minutos — consultarlos sin `rate()` solo daría el total acumulado, no una velocidad útil para un gráfico. Ambas métricas se devuelven por interfaz de red (`device`), así que un dashboard típico las filtra o suma según convenga.

### Exporters
- Un **exporter** obtiene métricas de un sistema que no las expone nativamente en formato Prometheus, las transforma y las expone vía HTTP en `/metrics`. Hay exporters oficiales y de terceros.
- **Servidor Linux**: instalar el **Node Exporter** (puerto 9100 por defecto) — convierte CPU/RAM/disco a métricas, Prometheus las scrapea desde `/metrics`.
- **App propia**: usar una librería cliente de Prometheus (`prom-client` en Node.js, `prometheus-net` en .NET...), definir métricas propias (peticiones, errores, latencia...) y exponerlas en `/metrics`.
```javascript
const client = require('prom-client');
const counter = new client.Counter({
  name: 'app_requests_total',
  help: 'Número total de peticiones'
});
```

### Alertmanager
- Recibe las alertas que dispara Prometheus (según reglas) y las envía a los canales configurados (Email, Slack, webhooks...).
- Prometheus necesita saber dónde está Alertmanager, y dónde cargar los ficheros de reglas — se añade a `prometheus.yml`:
```yaml
# Configuración de Alertmanager
alerting:
  alertmanagers:
    - static_configs:
        - targets:
          - alertmanager:9093

# Reglas de alertas
rule_files:
  - "rules/*.yml"
```
- Las reglas viven en archivos aparte (carpeta `rules/`), no en el propio `prometheus.yml` — así se pueden versionar/organizar por separado y cargar varios ficheros con un glob (`rules/*.yml`).
```yaml
# rules/alerts.yml
groups:
- name: alertas
  rules:
  # Alerta de uso de CPU alto (forzada para pruebas)
  - alert: HighCPUUsage
    expr: 100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 0
    for: 1m
    labels:
      severity: warning
    annotations:
      summary: "Alto uso de CPU (prueba)"
      description: "El uso de CPU está por encima del 0% durante 1 minuto (esto es solo una prueba)"

  # Alerta de uso de memoria alto
  - alert: HighMemoryUsage
    expr: (1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100 > 85
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "Alto uso de memoria"
      description: "El uso de memoria está por encima del 85% durante 5 minutos"

  # Alerta de espacio en disco bajo
  - alert: LowDiskSpace
    expr: (1 - node_filesystem_free_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"}) * 100 > 90
    for: 5m
    labels:
      severity: warning
    annotations:
      summary: "Espacio en disco bajo"
      description: "El espacio en disco está por debajo del 10% durante 5 minutos"
```
- `HighCPUUsage` está deliberadamente forzada con `> 0` (siempre se cumple) para comprobar que el flujo Prometheus → Alertmanager → notificación funciona de punta a punta antes de afinar umbrales reales.
- El resto de reglas siguen el mismo patrón: una expresión PromQL que calcula un porcentaje, un umbral, y un `for` que exige que la condición se mantenga un tiempo mínimo antes de disparar — evita alertas por picos momentáneos que se resuelven solos.
```yaml
# alertmanager.yml
global:
  resolve_timeout: 3m
route:
  receiver: "default-notifications"
  group_by: ['alertname', 'job']
  group_wait: 10s
  group_interval: 2m
  repeat_interval: 4h
  routes:
    - receiver: "alerts"
      match: {severity: "critical", team: "infrastructure"}
      continue: true
    - receiver: "app-alerts"
      match: {service: "api-service", severity: "warning"}
receivers:
  - name: "default-notifications"
    email_configs:
      - to: "example@example.com"
  - name: "alerts"
    slack_configs:
      - channel: "#aws-alerts"
        send_resolved: true
  - name: "app-alerts"
    webhook_configs:
      - url: "http://app-alert-handler.local/notify"
        send_resolved: false
```
- `route` define cómo se enrutan las alertas según sus etiquetas (`match`); `receiver` en cada bloque define a dónde van. `continue: true` permite que una alerta siga evaluándose contra rutas posteriores además de la que ya coincidió.

El stack completo (Prometheus + Node Exporter + Alertmanager) en un único `docker-compose`:
```yaml
# docker-compose-with-alertmanager.yml
# Versión completa con sistema de alertas
services:
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus:/etc/prometheus
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--web.console.libraries=/usr/share/prometheus/console_libraries'
      - '--web.console.templates=/usr/share/prometheus/consoles'
    restart: unless-stopped
    depends_on:
      - node-exporter
      - alertmanager

  node-exporter:
    image: prom/node-exporter:latest
    container_name: node-exporter
    ports:
      - "9100:9100"
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - '--path.procfs=/host/proc'
      - '--path.sysfs=/host/sys'
      - '--path.rootfs=/rootfs'
      - '--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)'
    restart: unless-stopped

  alertmanager:
    # Imagen oficial de Alertmanager
    image: prom/alertmanager:latest
    container_name: alertmanager
    ports:
      - "9093:9093"  # Puerto web de Alertmanager
    volumes:
      - ./alertmanager:/etc/alertmanager  # Configuración personalizada
      - alertmanager_data:/alertmanager   # Datos persistentes
    command:
      - '--config.file=/etc/alertmanager/alertmanager.yml'
      - '--storage.path=/alertmanager'
    restart: unless-stopped

volumes:
  prometheus_data:
  alertmanager_data:
```
- Tres servicios, cada uno con su propio volumen de configuración montado desde una carpeta local (`./prometheus`, `./alertmanager`) y su propio volumen con nombre para persistir datos — el mismo patrón repetido de forma consistente en los tres.
- El puerto `9093` es la UI web de Alertmanager, donde se pueden ver las alertas activas/silenciadas — útil para depurar sin esperar a que llegue la notificación por Slack/email.

Configuración real de `alertmanager/alertmanager.yml`, notificando por email vía SMTP (Gmail):
```yaml
global:
  resolve_timeout: 5m
  # Configuración del servidor SMTP
  smtp_smarthost: 'smtp.gmail.com:587'
  smtp_from: '<tu-email>@gmail.com'
  smtp_auth_username: '<tu-email>@gmail.com'
  smtp_auth_password: '<contraseña-de-aplicacion>'
  smtp_require_tls: true

route:
  group_by: ['alertname']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 1h
  receiver: 'email-notifications'

receivers:
- name: 'email-notifications'
  email_configs:
  - to: '<tu-email>@gmail.com'
    send_resolved: true
    headers:
      subject: '{{ template "email.subject" . }}'
    html: '{{ template "email.html" . }}'

templates:
- '/etc/alertmanager/templates/*.tmpl'

inhibit_rules:
  - source_match:
      severity: 'critical'
    target_match:
      severity: 'warning'
    equal: ['alertname', 'dev', 'instance']
```
- `smtp_auth_password` no es la contraseña normal de la cuenta de Gmail — Google exige una **contraseña de aplicación** específica cuando la cuenta tiene 2FA activado; la contraseña habitual no funciona aquí.
- `templates` carga plantillas personalizadas (`.tmpl`) para dar formato al asunto/cuerpo del email — sin definirlas, Alertmanager usa una plantilla por defecto más básica.
- **`inhibit_rules`**: si ya hay una alerta activa que cumple `source_match` (aquí, `severity: critical`), se **silencian** las alertas que cumplan `target_match` (`severity: warning`) para el mismo `alertname`/`dev`/`instance` (`equal`) — evita recibir un aviso de "warning" cuando ya llegó el de "critical" para lo mismo, que sería ruido redundante.

### Grafana
- Herramienta open source para crear dashboards dinámicos y reutilizables, conectándose a Prometheus (y también Loki, CloudWatch, etc.) para mostrar tendencias, alertas y comportamiento en tiempo real.
- Preguntas típicas que responde un dashboard de app: usuarios conectados, tasa de errores, latencia de respuesta, uso de CPU/memoria, peticiones por segundo, disponibilidad/caídas.
- Flujo típico: instalar una librería cliente de Prometheus en la app → definir métricas en el código → exponer `/metrics` → Prometheus la scrapea → Grafana la visualiza.
- Un panel de Grafana se construye con la misma sintaxis PromQL vista antes — se añade Prometheus como datasource, y una consulta como esta sirve directamente como query de un panel de "uso de CPU por instancia":
```promql
rate(node_cpu_seconds_total{mode!="idle"}[5m])
```
- `node_cpu_seconds_total` acumula segundos de CPU por `mode` (`idle`, `user`, `system`...) y por núcleo; filtrar `mode!="idle"` y aplicar `rate()` da el % de tiempo que la CPU ha estado ocupada (no en reposo) en los últimos 5 minutos, desglosado por instancia y núcleo.

Otro panel típico, "ratio de memoria libre":
```promql
node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes
```
- A diferencia de la query de "% memoria usada" (que resta a 1 y multiplica por 100), esta deja el resultado como una fracción entre 0 y 1 de memoria **disponible** — útil si el panel de Grafana ya formatea el valor como porcentaje por su cuenta (unidad `Percent (0.0-1.0)`), sin necesidad de hacer la transformación en la propia consulta.

Panel de "tiempo de actividad del sistema" (uptime):
```promql
node_time_seconds - node_boot_time_seconds
```
- `node_time_seconds` es la hora actual del sistema (en segundos, formato Unix); `node_boot_time_seconds` es la hora en que arrancó. La resta da directamente los segundos que lleva el sistema encendido — Grafana puede formatear ese número como una duración legible (días/horas/minutos) sin más cálculo.
- Un mismo panel (o un mismo dashboard) puede tener **varias queries a la vez** — por ejemplo, un panel con una query por núcleo de CPU, o un dashboard con un panel por cada una de las consultas anteriores (CPU, memoria, red, uptime) — no hace falta un dashboard distinto por métrica.
- **Grafana Drilldown**: conjunto de apps (Metrics, Logs, Traces, Profiles Drilldown) para explorar métricas/logs/trazas/perfiles **sin escribir PromQL/LogQL a mano** — navegación guiada por clics en vez de construir la consulta desde cero, pensada para simplificar la exploración inicial antes de crear un dashboard/panel definitivo.
- **Alertas en Grafana**: Grafana tiene su propio apartado de alertas, con dos usos posibles — crear reglas de alerta propias desde Grafana (independientes de las de Prometheus), o visualizar/gestionar desde ahí las alertas que ya vienen de Prometheus + Alertmanager (añadiendo Alertmanager como datasource). No sustituye a Alertmanager, sino que centraliza la vista de alertas de varias fuentes en un mismo sitio.

Stack completo con Grafana añadido (`docker-compose-with-grafana.yml`):
```yaml
# Archivo docker-compose básico para Prometheus y Grafana
services:
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus:/etc/prometheus
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--web.console.libraries=/usr/share/prometheus/console_libraries'
      - '--web.console.templates=/usr/share/prometheus/consoles'
    depends_on:
      - node-exporter
      - alertmanager
    restart: unless-stopped

  alertmanager:
    image: prom/alertmanager:latest
    container_name: alertmanager
    ports:
      - "9093:9093"
    volumes:
      - ./alertmanager:/etc/alertmanager
      - alertmanager_data:/alertmanager
    command:
      - '--config.file=/etc/alertmanager/alertmanager.yml'
      - '--storage.path=/alertmanager'
    restart: unless-stopped

  node-exporter:
    image: prom/node-exporter:latest
    container_name: node-exporter
    ports:
      - "9100:9100"
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - '--path.procfs=/host/proc'
      - '--path.sysfs=/host/sys'
      - '--path.rootfs=/rootfs'
      - '--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)'
    restart: unless-stopped

  grafana:
    # Imagen oficial de Grafana
    image: grafana/grafana:latest
    container_name: grafana
    ports:
      - "3000:3000"  # Puerto web de Grafana
    volumes:
      - grafana_data:/var/lib/grafana  # Datos persistentes
    environment:
      - GF_SECURITY_ADMIN_USER=admin      # Usuario por defecto
      - GF_SECURITY_ADMIN_PASSWORD=admin  # Contraseña por defecto
    depends_on:
      - prometheus
    restart: unless-stopped

volumes:
  prometheus_data:
  grafana_data:
  alertmanager_data:
```
- Grafana solo necesita un volumen persistente (`grafana_data`) — a diferencia de Prometheus/Alertmanager, no monta configuración desde una carpeta local: los dashboards, datasources y usuarios se gestionan desde su propia UI (o vía provisioning, si se añade más adelante).
- `GF_SECURITY_ADMIN_USER`/`GF_SECURITY_ADMIN_PASSWORD` fijan las credenciales iniciales de `admin` vía variables de entorno — cómodo para levantar el entorno rápido, pero en un despliegue real conviene cambiarlas o inyectarlas desde un secreto, no dejarlas como `admin`/`admin`.
- Grafana escucha en el puerto **3000**, distinto del 9090 de Prometheus y el 9093 de Alertmanager — cada servicio del stack tiene su propia UI web independiente.
- En vez de construir un dashboard desde cero, el directorio oficial de Grafana (https://grafana.com/grafana/dashboards/) tiene dashboards ya hechos e importables por ID para fuentes comunes (Node Exporter, Docker, Kubernetes...) — punto de partida habitual antes de personalizar.
- **Autenticación**: en producción no se deja el login por defecto (`admin`/`admin`) — desde *Administration → Authentication*, Grafana permite configurar de forma sencilla el login vía proveedores externos (GitHub, GitLab, Google, Azure AD...), en vez de gestionar usuarios y contraseñas manualmente dentro de la propia herramienta.
- **Permisos por rol**: a cada usuario se le puede asignar un nivel de acceso — por ejemplo, solo lectura (ver dashboards existentes) o edición (crear/modificar dashboards y consultas) — en vez de dar acceso total de administrador a todo el mundo.

### Grafana Loki
- Sistema de almacenamiento de logs de Grafana Labs: **no indexa el contenido** de los logs, solo las etiquetas — más ligero que Elasticsearch, pensado para integrarse con Prometheus/Grafana y escalar bien en Kubernetes.
- Se consulta con **LogQL** desde Grafana o `LogCLI`.

### Promtail (en desuso) → Grafana Alloy
- **Promtail**: agente que recolectaba logs de servidores/pods (ej. `/var/log/containers/*.log`), los etiquetaba automáticamente (job, namespace, pod, container) y los enviaba a Loki.
- ⚠️ Promtail queda **oficialmente en desuso** desde el 13/02/2025 (solo parches críticos), soporte comercial hasta el 28/02/2026, EOL el 02/03/2026.
- **Grafana Alloy** es su sustituto: agente unificado de observabilidad que reemplaza a Promtail, Node Exporter y otros agentes específicos — recoge logs, métricas y trazas desde un único agente, compatible con Prometheus y Loki.

### Desplegar Loki + Alloy: ejemplo completo con logs de Docker
Configuración de Loki (`loki/loki-config.yaml`) — almacenamiento local en filesystem, sin autenticación (solo para pruebas):
```yaml
# auth_enabled: Desactiva autenticación (no recomendado en producción)
auth_enabled: false

server:
  http_listen_port: 3100

ingester:
  lifecycler:
    address: 127.0.0.1
    ring:
      kvstore:
        store: inmemory       # Almacenamiento en memoria para el anillo de ingesters
      replication_factor: 1   # Solo una réplica (ideal para pruebas)
    final_sleep: 0s
  chunk_idle_period: 5m
  chunk_retain_period: 30s
  wal:
    dir: /loki/wal

schema_config:
  configs:
    - from: 2020-05-15
      store: boltdb-shipper
      object_store: filesystem
      schema: v11
      index:
        prefix: index_
        period: 24h

storage_config:
  boltdb_shipper:
    active_index_directory: /loki/boltdb-shipper-active
    cache_location: /loki/boltdb-shipper-cache
    cache_ttl: 24h
  filesystem:
    directory: /loki/chunks

compactor:
  working_directory: /loki/compactor

limits_config:
  retention_period: 744h   # 31 días
  volume_enabled: true
  allow_structured_metadata: false

ruler:
  alertmanager_url: http://localhost:9093  # opcional: alertas basadas en logs
```

Configuración de Alloy (`alloy/alloy-config.yaml`) — recolecta logs de contenedores Docker y los envía a Loki:
```yaml
logs:
  configs:
    - name: default
      positions:
        filename: /tmp/positions.yaml
      clients:
        - url: http://loki:3100/loki/api/v1/push
      scrape_configs:
        - job_name: docker
          static_configs:
            - targets: [localhost]
              labels:
                job: docker-logs
                __path__: /var/lib/docker/containers/*/*log
          pipeline_stages:
            - json:
                expressions:
                  stream: stream
                  attrs: attrs
                  tag: attrs.tag
                  time: time
            - labels:
                stream:
                tag:
            - timestamp:
                source: time
                format: RFC3339Nano
            - output:
                source: log
```
- `__path__` apunta al patrón de logs de Docker en el host (`/var/lib/docker/containers/*/*log`) — Alloy los lee directamente del filesystem, no vía API de Docker.
- Docker escribe cada línea de log como JSON; el `pipeline_stages` la parsea (`json`), extrae etiquetas (`stream`, `tag`), fija el timestamp real del log (en vez de la hora de ingesta) y se queda solo con el campo `log` como contenido final.

Stack completo (app de ejemplo + Loki + Alloy + Grafana):
```yaml
# docker-compose-with-loki.yml
version: '3.8'

services:
  # Aplicación simple que genera logs
  app-simple:
    image: alpine:latest
    container_name: app-simple
    command: >
      sh -c "
        echo 'Iniciando aplicación simple...' &&
        while true; do
          echo '$(date): INFO - Aplicación funcionando correctamente' &&
          echo '$(date): WARNING - Uso de memoria: $(free -m | grep Mem | awk \"{print \\$3}\")MB' &&
          if [ $$((RANDOM % 10)) -eq 0 ]; then
            echo '$(date): ERROR - Error simulado en la aplicación'
          fi &&
          sleep 5
        done
      "
    networks:
      - monitoring
    restart: unless-stopped
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"

  loki:
    image: grafana/loki:latest
    container_name: loki
    ports:
      - "3100:3100"
    command: -config.file=/etc/loki/local-config.yaml
    volumes:
      - ./loki/loki-config.yaml:/etc/loki/local-config.yaml
      - loki_data:/loki
    networks:
      - monitoring
    restart: unless-stopped

  # Grafana Alloy para recopilar logs (reemplaza Promtail)
  alloy:
    image: grafana/agent:latest
    container_name: alloy
    ports:
      - "12345:12345"  # Puerto para métricas de Alloy
    volumes:
      - ./alloy/alloy-config.yaml:/etc/grafana-agent.yaml:ro
      - /var/lib/docker/containers:/var/lib/docker/containers:ro
    command: ["-config.file=/etc/grafana-agent.yaml"]
    networks:
      - monitoring
    restart: unless-stopped
    depends_on:
      - loki

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    ports:
      - "3000:3000"
    volumes:
      - grafana_data:/var/lib/grafana
      - ./grafana/datasources:/etc/grafana/provisioning/datasources
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=admin
      - GF_USERS_ALLOW_SIGN_UP=false
    networks:
      - monitoring
    restart: unless-stopped
    depends_on:
      - loki

networks:
  monitoring:
    driver: bridge

volumes:
  loki_data:
  grafana_data:
```
- `app-simple` es un contenedor de prueba que genera logs INFO/WARNING/ERROR (el ERROR sale aleatoriamente, ~10% de las veces) — sirve para tener algo real que ver en Loki sin depender de una app propia.
- Alloy monta `/var/lib/docker/containers` **del host** en modo solo lectura — necesita ver los logs de *todos* los contenedores Docker, no solo los suyos, para poder recolectarlos.
- `GF_USERS_ALLOW_SIGN_UP=false` desactiva el autorregistro de usuarios en Grafana — con autenticación por proveedor externo (GitHub/Google/...) es habitual desactivar esto para que solo entren cuentas invitadas explícitamente, no cualquiera con cuenta en el proveedor.
- El datasource de Loki en Grafana se añade vía `./grafana/datasources` (provisioning por archivo) — una forma de configurar Grafana como código, en vez de añadir el datasource a mano desde la UI cada vez que se levanta el entorno.

### Mini proyecto del módulo
Stack completo Prometheus + Grafana + Loki + Alloy: una app Flask expone métricas y logs, se recolectan y se visualizan en dashboards en tiempo real. Todo el código vive en este módulo:
- [`app/`](./app) — API Flask de ejemplo con métricas Prometheus y logging JSON.
- [`prometheus/`](./prometheus) — configuración y reglas de alerta.
- [`alloy/`](./alloy) y [`loki/`](./loki) — recolección y almacenamiento de logs.
- [`grafana/`](./grafana) — dashboards y datasources provisionados por archivo.
- [`monitor.sh`](./monitor.sh) — script que envuelve `docker-compose` (start/stop/logs/test/backup/urls...).

## Notas y gotchas

- Loki no indexa el contenido de los logs, solo etiquetas — buscar por texto libre es mucho más limitado que en Elasticsearch; el diseño prioriza coste y escalabilidad sobre búsqueda full-text potente.
- Provisionar datasources/dashboards por archivo (`grafana/provisioning/`) en vez de configurarlos a mano en la UI es lo que permite recrear el entorno completo con `docker-compose up` desde cero — sin esto, cada vez que se recrea el contenedor de Grafana habría que volver a añadir Prometheus/Loki como datasource e importar los dashboards manualmente.
- `exemplarTraceIdDestinations` en el datasource de Prometheus referencia un datasource `jaeger` que **no existe** en este stack — es una opción preparada para cuando se añada tracing (Jaeger/Tempo) más adelante; sin ese datasource, simplemente no tiene efecto, no rompe nada.
- La función `create_backup` de `monitor.sh` intenta copiar una carpeta `promtail/` que ya no existe en este proyecto (se sustituyó por `alloy/`) — no falla porque usa `|| true`, es simplemente un resto de una versión anterior del script que no se limpió del todo al migrar a Alloy.
- Si el proyecto todavía usa Promtail, hay que planificar la migración a Grafana Alloy antes de la fecha de EOL — no es opcional a medio plazo, Promtail deja de recibir hasta parches de seguridad.
- Un exporter no "inventa" métricas — traduce las que el sistema de origen ya tiene a un formato que Prometheus entiende; si el sistema no expone cierto dato, no hay exporter que lo saque de la nada.
- `continue: true` en una ruta de Alertmanager es fácil de olvidar — sin él, la primera ruta que hace match "se queda" con la alerta y no la evalúa contra las siguientes, aunque debería notificar a más de un receptor.

## Recursos

- https://prometheus.io/docs/introduction/overview/
- https://prometheus.io/docs/prometheus/latest/configuration/configuration/
- https://prometheus.io/docs/prometheus/latest/querying/basics/
- https://prometheus.io/docs/alerting/latest/alertmanager/
- https://grafana.com/docs/loki/latest/
- https://grafana.com/docs/alloy/latest/
- https://grafana.com/docs/grafana/latest/visualizations/simplified-exploration/
- https://grafana.com/grafana/dashboards/
