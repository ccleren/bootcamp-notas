# Bootcamp Cloud/DevOps — Notas y Repaso

Repositorio personal de apuntes, resúmenes y comandos del bootcamp de Cloud/DevOps (AWS). El objetivo es tener un lugar de repaso rápido y evitar tener que repasar el material completo del curso cada vez que necesito recordar algo.

No contiene el material oficial del curso (PDFs, código de los profesores) — solo mis propias notas.

## Cómo se usa

Cada módulo tiene su propia carpeta con un `README.md` (basado en `_plantilla.md`). Lo actualizo mientras avanzo:

- **Resumen**: qué es el tema y para qué sirve, en mis palabras.
- **Comandos clave**: los comandos/CLI que uso o que se me olvidan fácil.
- **Notas y gotchas**: cosas que me sorprendieron, errores típicos, decisiones de diseño.
- **Recursos**: enlaces o referencias que quiero volver a consultar.

Cuando empiece un módulo nuevo, copio `_plantilla.md` a `modulo-XX-nombre/README.md` y lo relleno según avanzo (no hace falta esperar a terminar el módulo).

## Progreso

| Módulo | Tema | Estado |
|---|---|---|
| 01 | Introducción a Cloud Computing y DevOps | ✅ Completo |
| 02 | Infraestructura global del Cloud de AWS | ✅ Completo |
| 03 | AWS IAM | ✅ Completo |
| 04 | Crear una organización en el Cloud | ✅ Completo |
| 05 | VPC — conceptos fundamentales | ✅ Completo |
| 06 | EC2 - Básico | ✅ Completo |
| 07 | EC2 - Avanzado (Spot, CloudWatch Agent) | ✅ Completo |
| 08 | AWS Systems Manager | ✅ Completo |
| 09 | Almacenamiento de EC2 (AMI, EBS, Instance Store, EFS) | ✅ Completo |
| 10 | Elastic Load Balancing (ELB) | ✅ Completo |
| 11 | Auto Scaling Groups (ASG) | 🟡 Falta sección de comandos de la demo |

Leyenda: 🟢 en curso · 🟡 con notas parciales · ✅ completo

Los módulos 1-10 se redactaron a partir de `Bootcamp-v4.pdf` (slides del curso) combinado con mis propios scripts/comandos de las demos prácticas. El módulo 11 en adelante se actualiza en tiempo real conforme avanzo.

## Estructura

```
bootcamp-notas/
├── _plantilla.md                        # plantilla para nuevos módulos
├── modulo-01-introduccion-cloud-devops/
├── modulo-02-infraestructura-global/
├── modulo-03-iam/
├── modulo-04-organizacion-cloud/
├── modulo-05-vpc/
├── modulo-06-ec2-basico/
├── modulo-07-ec2-avanzado/
├── modulo-08-aws-systems-manager/
├── modulo-09-almacenamiento-ec2/
├── modulo-10-elb/
└── modulo-11-auto-scaling-groups/
```
