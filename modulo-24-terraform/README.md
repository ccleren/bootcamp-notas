# Módulo 24 — Terraform

## Resumen

### Qué es Terraform
- Herramienta de **Infraestructura como Código (IaC)**: define y aprovisiona infraestructura mediante archivos de configuración, en vez de crearla a mano en la consola.
- **Independiente de plataforma**: soporta AWS, Azure, GCP y también infraestructura on-premise o software como VMware/Docker — el mismo lenguaje de configuración sirve para distintos proveedores.
- Soporta **módulos reutilizables** para organizar componentes comunes de infraestructura.

### Flujo de trabajo (etapas)
1. **Escribir**: definir la infraestructura deseada en los archivos de configuración.
2. **Planear**: revisar qué cambios va a hacer Terraform antes de aplicarlos.
3. **Aplicar**: Terraform aprovisiona los recursos y actualiza su archivo de estado.

### Componentes principales
| Componente | Qué es |
|---|---|
| **Archivos de configuración (`.tf`)** | Definen, en el lenguaje de Terraform, qué recursos se necesitan y cómo configurarlos |
| **Terraform Core** | El motor: interpreta la configuración y calcula las acciones concretas para llegar al estado deseado |
| **Archivo de estado (`.tfstate`)** | Registra el estado actual de los recursos ya gestionados por Terraform |
| **Providers** | Plugins que permiten a Terraform hablar con un proveedor concreto (AWS, GCP, Azure...) |
| **Provisioners** | Ejecutan comandos/scripts en una máquina local o remota justo al crearse un recurso (`remote-exec`, `local-exec`) |
| **Client Library** | SDK del proveedor (ej. `aws-sdk-go`) usado por el provider para comunicarse con la nube vía su API |

### Módulos
- Configuraciones reutilizables que agrupan varios recursos relacionados (ej. un módulo "MiVPC" que crea VPC + subredes públicas/privadas + NAT Gateway).
- Un módulo puede llamar a otros módulos ("módulos hijo"), y un mismo módulo puede reutilizarse varias veces (en la misma configuración o en configuraciones distintas).
- Ventajas: **reutilización** (no repetir la misma definición de infraestructura), **abstracción** (ocultar la complejidad detrás de una interfaz simple), **escalabilidad** (escalar componentes sueltos en vez de todo el monolito de infraestructura), **gestión de cambios controlada** (aislar y probar un componente antes de aplicarlo al resto).

### Estructura mínima de un módulo
```
minimal-module/
├── main.tf         # los recursos del módulo
├── variables.tf    # las variables de entrada del módulo
└── outputs.tf      # las salidas del módulo, para que otros módulos las consuman
```

### Ejemplo: módulo que despliega una EC2 con un Security Group

`variables.tf` — variables de entrada del módulo:
```hcl
variable "region" {
  description = "Región de AWS donde se desplegarán los recursos"
  default     = "us-east-1"
}
```

`main.tf` — recursos del módulo:
```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 4.16"
    }
  }
  required_version = ">= 1.2.0"
}

provider "aws" {
  region = var.region
}

# Security Group para SSH y HTTP
resource "aws_security_group" "web" {
  name        = "web_sg"
  description = "Permite trafico SSH y HTTP"

  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

# Instancia EC2 con un servidor web instalado vía user_data
resource "aws_instance" "web" {
  ami             = "<ami-id>"
  instance_type   = "t2.micro"
  security_groups = [aws_security_group.web.name]

  tags = {
    Name = "WebServerInstance"
  }

  user_data = <<-EOF
              #!/bin/bash
              yum update -y
              yum install -y httpd
              systemctl start httpd
              systemctl enable httpd
              echo "<h1>Hola desde $(hostname -f)</h1>" > /var/www/html/index.html
              EOF
}
```

`outputs.tf` — salidas del módulo:
```hcl
output "instance_id" {
  description = "ID de la instancia EC2"
  value       = aws_instance.web.id
}

output "instance_public_ip" {
  description = "IP publica de la instancia EC2"
  value       = aws_instance.web.public_ip
}
```

## Comandos clave

### Inicializar

```bash
# Ver la versión instalada de Terraform
terraform version

# Inicializar el directorio de trabajo: descarga los providers necesarios y prepara el estado
terraform init

# Mostrar los providers que requiere la configuración actual
terraform providers

# Instalar/actualizar los módulos remotos referenciados en la configuración
terraform get
```

### Planificar, aplicar y destruir

```bash
# Generar un plan de ejecución (qué va a crear/cambiar/borrar, sin aplicarlo aún)
terraform plan

# Guardar el plan generado en un archivo, para aplicarlo tal cual más tarde
terraform plan -out <archivo-plan.out>

# Generar un plan de destrucción (previsualizar qué borraría un destroy)
terraform plan -destroy

# Fijar el valor de una variable de entrada al planificar/aplicar
terraform plan -var "<nombre-variable>=<valor>"

# Aplicar los cambios del plan a la infraestructura real
terraform apply

# Aplicar sin pedir confirmación interactiva
terraform apply --auto-approve

# Aplicar exactamente el plan guardado previamente en un archivo
terraform apply <archivo-plan.out>

# Actualizar el estado para que refleje la infraestructura real (sin aplicar cambios)
terraform refresh

# Destruir todos los recursos gestionados por esta configuración
terraform destroy
```

### Salidas (outputs)

```bash
# Listar todas las salidas definidas en la configuración
terraform output

# Listar las salidas en formato JSON
terraform output -json

# Consultar una salida concreta
terraform output <nombre-output>
```

### Gestión del estado

```bash
# Listar todos los recursos que hay en el archivo de estado actual
terraform state list

# Ver el detalle de un recurso concreto en el estado
terraform state show <recurso>

# Descargar el estado remoto a un archivo local
terraform state pull > terraform.tfstate

# Dejar de gestionar un recurso (lo quita del estado, no lo borra de la infraestructura real)
terraform state rm <recurso>
```

### Marcar un recurso para recrear (taint)

```bash
# Marcar un recurso para que se recree en el próximo apply
terraform taint <recurso>

# Quitar esa marca
terraform untaint <recurso>
```

### Formatear y validar

```bash
# Reformatear el código a la convención estándar de Terraform
terraform fmt

# Formatear también las subcarpetas
terraform fmt -recursive

# Validar la sintaxis de la configuración
terraform validate
```

### Espacios de trabajo (workspaces)

```bash
# Listar los workspaces existentes
terraform workspace list

# Cambiar de workspace
terraform workspace select <workspace-name>

# Crear un workspace nuevo
terraform workspace new <workspace-name>
```

## Notas y gotchas

- `terraform plan` no toca nada — es puramente informativo; el cambio real solo ocurre con `terraform apply`. Revisar el plan antes de aplicar es la forma de detectar un cambio no intencionado (ej. un recurso que se va a recrear en vez de actualizar).
- El archivo `.tfstate` es el que le dice a Terraform qué existe ya y qué no — si se pierde o se desincroniza de la infraestructura real, Terraform puede intentar recrear recursos que ya existen o borrar los que no debería.
- Un módulo no es solo "una carpeta con recursos": la convención `main.tf` / `variables.tf` / `outputs.tf` es lo que permite que otros módulos lo reutilicen sin mirar por dentro, pasando variables de entrada y leyendo sus outputs.
- `terraform destroy` borra **todo** lo que la configuración gestiona — no hay una versión parcial "borra solo esto" salvo apuntando a un recurso concreto explícitamente; hay que tener claro el alcance antes de ejecutarlo.
- `terraform state rm` **no borra nada en la nube** — solo hace que Terraform deje de rastrear ese recurso; el recurso real sigue existiendo, simplemente queda fuera del control de esa configuración.
- `terraform taint`/`untaint` es el mecanismo clásico para forzar la recreación de un recurso; en versiones recientes de Terraform se recomienda `terraform apply -replace=<recurso>` en su lugar, que hace lo mismo en un solo paso.

## Recursos

- https://developer.hashicorp.com/terraform/intro
- https://developer.hashicorp.com/terraform/language/modules
- https://developer.hashicorp.com/terraform/cli/commands
- https://developer.hashicorp.com/terraform/language/providers
