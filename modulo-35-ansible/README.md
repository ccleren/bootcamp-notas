# Módulo 35 — Ansible

## Resumen

### Qué es y qué problema resuelve
- Herramienta de automatización y gestión de configuraciones: crear hosts/VMs, aplicar configuraciones, migraciones, despliegues, parches en cientos de servidores a la vez.
- **Sin agentes**: no hay que instalar nada en los nodos remotos, funciona vía SSH (o WinRM en Windows).

### Script Bash vs. Ansible
| | Bash | Ansible |
|---|---|---|
| Ejecución | SSH manual, comandos imperativos | Define el **estado deseado**, sin SSH manual explícito |
| Escalabilidad | Hay que editar el script para más servidores | Inventario (estático o dinámico) |
| Idempotencia | Puede repetir comandos sin comprobar el estado actual | Aplica solo los cambios necesarios |
| Mantenimiento | Se complica en infraestructuras grandes | Estructurado y reutilizable |

```yaml
# instalar_nginx.yaml — el equivalente declarativo de un bucle SSH con install/enable/start en Bash
---
- name: Configuración de Nginx
  hosts: webservers
  become: true
  tasks:
    - name: Actualizar paquetes
      apt:
        update_cache: yes
    - name: Instalar Nginx
      apt:
        name: nginx
        state: present
    - name: Habilitar y arrancar Nginx
      service:
        name: nginx
        state: started
        enabled: yes
```

### Arquitectura de nodos
- **Nodo de control**: máquina donde corre Ansible y desde la que se envían las instrucciones.
- **Nodos gestionados**: servidores remotos administrados sin agente, solo por SSH/WinRM.
- En arquitecturas más avanzadas, el nodo de control puede recibir peticiones desde un sistema ITSM, consultar datos dinámicos (BBDD, infraestructura cloud) y aplicar configuraciones tanto en nodos cloud como on-premise.
- Requisito del nodo de control: tener **Python** instalado.

### Inventory (inventario)
- Lista de hosts que Ansible administra: archivo estático (típicamente `/etc/ansible/hosts`) o inventario dinámico generado por script/herramienta.
- Se pueden agrupar hosts por rol/función/ubicación.
```ini
[webservers]
web1.example.com
web2.example.com

[dbservers]
db1.example.com
db2.example.com
```
- Variables por host (conexión, usuario, etc.):
```ini
db  ansible_host=server1.company.com ansible_connection=ssh   ansible_user=root
web ansible_host=server2.company.com ansible_connection=winrm ansible_user=admin
```
- `ansible_connection`: `ssh` (Linux), `winrm` (Windows), `localhost` (tareas locales).
- ⚠️ Evitar contraseñas en texto plano en el inventario — usar **Ansible Vault** para secretos.

### Archivo de configuración (`ansible.cfg`)
- Personaliza comportamiento: ruta del inventario, logging, rutas de módulos/roles/plugins, timeout, nº de forks (concurrencia).
```ini
[defaults]
inventory = /etc/ansible/hosts
log_path  = /var/log/ansible.log
timeout   = 10
forks     = 5
```

### Playbooks
- Archivo YAML que define tareas — es el núcleo de la automatización en Ansible.
- Se tratan como **Infraestructura como Código**: se versionan en Git como cualquier otro código.
```yaml
- name: Playbook 1
  hosts: localhost
  tasks:
    - name: Mostrar la fecha
      command: date
    - name: Ejecutar un script
      script: test_script.sh
    - name: Instalar un servicio web
      yum:
        name: httpd
        state: present
    - name: Iniciar el servicio web
      service:
        name: httpd
        state: started
```

### Módulos
- Unidad de código reutilizable que ejecuta una tarea concreta en el nodo remoto (`apt`, `yum`, `file`, `user`...).
- Se pueden crear módulos propios, pero rara vez hace falta — la librería de módulos predeterminados cubre casi todo sin escribir scripts a medida.

### Colecciones
- Formato de empaquetado para distribuir contenido de Ansible junto (playbooks, plugins, módulos...).
- Se pueden crear colecciones propias para proyectos grandes — ej. una colección para automatizar infraestructura cloud (como la AWS Collection).

### Proyecto práctico: despliegue de una app
1. Crear una instancia EC2 (t2.micro), configurar acceso SSH y grupos de seguridad.
2. Escribir un playbook que: instale Node.js/npm, copie y descomprima el artefacto de la app, la inicie, y verifique que responde correctamente.

## Notas y gotchas

- La diferencia central frente a un script Bash no es solo sintaxis — es que Ansible es **idempotente**: ejecutar el mismo playbook dos veces no reinstala/reconfigura de más, solo aplica lo que falta. Un script Bash típico no garantiza eso por sí mismo.
- El nodo de control necesita Python; los nodos gestionados solo necesitan ser alcanzables por SSH/WinRM — no hace falta instalar Ansible ni ningún agente en ellos, es la base del modelo "agentless".
- Nunca poner contraseñas en texto plano en el inventario (`ansible_ssh_pass=...`) — está bien para una demo rápida, pero en cualquier proyecto real hay que usar Ansible Vault desde el principio, no añadirlo después.
- Tratar los playbooks como código (control de versiones, revisión de cambios) es lo que realmente distingue "usar Ansible" de "tener un montón de YAMLs sueltos" — la disciplina importa tanto como la herramienta.

## Recursos

- https://docs.ansible.com/projects/ansible/latest/getting_started/index.html
- https://docs.ansible.com/projects/ansible/latest/inventory_guide/intro_inventory.html
- https://docs.ansible.com/projects/ansible/latest/playbook_guide/playbooks_intro.html
- https://docs.ansible.com/projects/ansible/latest/vault_guide/index.html
