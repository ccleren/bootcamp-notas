# Módulo 08 — AWS Systems Manager (SSM)

## Resumen

**AWS Systems Manager** permite visualizar, administrar y operar nodos (instancias EC2, servidores on-prem, multi-cloud) de forma **centralizada**, sin necesitar SSH/RDP directo.

### Cómo funciona
- Basado en **agente** (SSM Agent), preinstalado en las AMIs oficiales de AWS (Windows y Linux) o instalable manualmente en servidores on-prem.
- Para servidores fuera de AWS: se usa un **código/ID de activación** + un **rol IAM** para conectarlos de forma segura a Systems Manager (concepto de "activación híbrida").
- Requiere: agente instalado, conectividad al endpoint público de SSM (o VPC Endpoint), y un **rol IAM** con permisos.

### Casos de uso principales
- Gestionar inventario y aplicar parches de seguridad a escala.
- Ejecutar comandos remotos y mantener el "estado deseado" de las instancias.
- **Conexión segura a instancias EC2 incluso en VPCs privadas** (sin abrir el puerto 22/3389 a internet).

### Run Command
- Ejecuta comandos remotos a escala en instancias administradas **sin necesitar acceso SSH/RDP**.
- Usa **documentos de comandos** (Command Documents) reutilizables, en JSON/YAML.
- Se puede dirigir a instancias específicas, por **tags**, o por grupos de recursos.
- Controles de ejecución: **concurrencia** (cuántas instancias a la vez) y **umbral de error** (a partir de cuántos fallos se considera la tarea fallida).
- Salidas configurables a S3 (logs) o SNS (notificaciones); se puede disparar desde EventBridge.
- **Auditable con CloudTrail**: queda registro de quién ejecutó qué comando y cuándo.
- Seguro: basado en IAM (controla qué usuarios/roles pueden ejecutar comandos sobre qué recursos).

### Documentos SSM (SSM Documents)
- JSON/YAML con pasos secuenciales (steps) y parámetros dinámicos, versionables.
- Se guardan en el **Document Store**; los usa Run Command, Automation, State Manager, Maintenance Windows, Distributor.

### Patch Manager
- Automatiza el parcheo de seguridad de instancias EC2 (y on-prem).
- Usa **Patch Baselines**: predefinidas por AWS (ej. `AWS-AmazonLinux2DefaultPatchBaseline`, `AWS-UbuntuDefaultPatchBaseline`, `AWS-DefaultPatchBaseline` para Windows) o personalizadas.
- Casos de uso: actualización automática en producción, auditoría/reportes de cumplimiento, y respuesta urgente con "Patch Now" ante una vulnerabilidad crítica.

### Flujo SSM Inventory + Patching
1. Se agrupan instancias en **grupos de parches** (Grupo A, Grupo B...), cada uno con su propia patch baseline.
2. **Maintenance Windows** definen horario/duración/objetivos/tareas del parcheo.
3. Run Command aplica los parches sobre cada grupo.
4. Los resultados se envían a **SSM Inventory**, que centraliza el historial de parches para auditoría/reportes.

## Comandos clave

*(No hay comandos de CLI en las slides — la práctica es vía consola: activar Session Manager / Run Command sobre una instancia con rol IAM `AmazonSSMManagedInstanceCore`.)*

## Notas y gotchas

- El caso de uso más útil en el día a día: **conectarte a una instancia EC2 en subred privada sin SSH ni bastion host**, vía Session Manager (parte de SSM) — solo necesitas el agente y el rol IAM correcto.
- Requisito fácil de olvidar: la instancia necesita un **rol IAM** adjunto con permisos de SSM (política gestionada `AmazonSSMManagedInstanceCore`) — sin eso, el agente no puede registrarse aunque esté instalado.
- Relaciona con [[modulo-03-iam]]: Systems Manager es otro ejemplo de un servicio que necesita un rol IAM asignado a la instancia, no credenciales de usuario.

## Recursos

-
