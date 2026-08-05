# Módulo 08 — AWS Systems Manager (SSM)

## Resumen

Gestiona muchos servidores (EC2, on-prem, multi-cloud) de forma centralizada, sin SSH/RDP máquina a máquina.

### Cómo funciona
- Basado en **agente** (SSM Agent), preinstalado en AMIs oficiales o instalable a mano.
- Servidores fuera de AWS: código de activación + rol IAM ("activación híbrida").
- Requiere: agente instalado + conectividad al endpoint de SSM + **rol IAM** con permisos. Sin el rol, el agente no se registra aunque esté instalado.

### Casos de uso
- Inventario y parches de seguridad a escala.
- Ejecutar comandos remotos, mantener "estado deseado".
- **Conexión segura a EC2 en subred privada, sin SSH ni bastion host.**

### Run Command
- Ejecuta comandos remotos sin SSH/RDP, usando **documentos de comando** (JSON/YAML) reutilizables.
- Se dirige por instancia, tags o grupo de recursos.
- Controles: **concurrencia** (cuántas a la vez) y **umbral de error** (a partir de cuántos fallos se da por perdida la tarea).
- Salidas a S3/SNS, disparable desde EventBridge.
- Auditable con CloudTrail (quién ejecutó qué y cuándo). Basado en IAM.

### Documentos SSM
JSON/YAML con pasos secuenciales y parámetros dinámicos, versionables. Usados por Run Command, Automation, State Manager, Maintenance Windows, Distributor.

### Patch Manager
- Automatiza parches de seguridad usando **Patch Baselines** (predefinidas por SO — Amazon Linux 2, Ubuntu, Windows — o personalizadas).
- Casos de uso: actualización continua en producción, auditoría de cumplimiento, "Patch Now" para vulnerabilidades críticas.

### Flujo SSM Inventory + Patching
1. Instancias agrupadas en **grupos de parches**, cada uno con su baseline.
2. **Maintenance Windows** definen horario/duración/objetivo.
3. Run Command aplica los parches.
4. Resultados centralizados en **SSM Inventory** para auditoría.

## Comandos clave

```bash
# Ver qué instancias están registradas en SSM (agente + rol correctos)
aws ssm describe-instance-information

# Conectarte a una instancia sin SSH, vía Session Manager
aws ssm start-session --target <instance-id>

# Ejecutar un comando remoto en varias instancias por tag
aws ssm send-command \
  --document-name "AWS-RunShellScript" \
  --targets "Key=tag:Environment,Values=Prod" \
  --parameters commands="sudo yum update -y"

# Ver el resultado de un comando lanzado con Run Command
aws ssm get-command-invocation \
  --command-id <command-id> --instance-id <instance-id>

# Aplicar parches manualmente a una instancia (Patch Manager)
aws ssm send-command --document-name "AWS-RunPatchBaseline" \
  --targets "Key=instanceIds,Values=<instance-id>" \
  --parameters "Operation=Install"
```

## Notas y gotchas

- Requisito fácil de olvidar: sin el rol `AmazonSSMManagedInstanceCore`, el agente no se registra aunque esté instalado.
- Conecta con [[modulo-03-iam]]: otro servicio más que necesita un rol asignado a la instancia, no credenciales de usuario.

## Recursos

-
