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

### Ejemplo real: inventario para instancias EC2
Para gestionar servidores EC2 por SSH con un par de claves (en vez de usuario/contraseña), el archivo `hosts` referencia el `.pem` de la instancia:
```ini
[aws]
<ip-publica-ec2-1>
<ip-publica-ec2-2>

[aws:vars]
ansible_ssh_private_key_file=ansible_key.pem
ansible_user=ubuntu
ansible_python_interpreter=/usr/bin/python3.12
```
- `ansible_ssh_private_key_file`: ruta al fichero `.pem` del par de claves de la instancia EC2 (el mismo que se usaría con `ssh -i`).
- `ansible_user`: usuario del sistema operativo de la AMI (`ubuntu` para AMIs Ubuntu, `ec2-user` para Amazon Linux...) — igual que al conectar por SSH manualmente.
- `ansible_python_interpreter`: ruta al Python del nodo gestionado. Ansible intenta autodetectarlo y, si no encuentra uno claro (o encuentra varios), lanza un warning; fijarlo explícitamente elimina el aviso y evita ambigüedad si la instancia tiene más de una versión de Python instalada.
- No hace falta `ansible_connection=ssh` explícito aquí porque es el valor por defecto de Ansible para hosts remotos.
- `[aws]` agrupa hosts bajo un nombre de grupo — permite dirigirse a todos ellos por ese nombre en vez de listarlos uno a uno.
- `[aws:vars]` define variables compartidas por **todos** los hosts del grupo `aws` — así `[aws]` solo necesita las IPs, sin repetir clave/usuario/intérprete en cada línea. Es la forma de evitar duplicar las mismas variables host por host cuando hay varios nodos con la misma configuración de conexión.

## Comandos clave

```bash
# Comprobar la conexión con todos los hosts del inventario (usa el módulo "ping" de Ansible, no ICMP)
ansible all -i hosts -m ping

# Comprobar la conexión solo con los hosts del grupo [aws]
ansible aws -i hosts -m ping
```
- Este `ping` es un módulo de Ansible, no el comando `ping` de red — verifica que Ansible puede autenticarse y ejecutar Python en el host, no solo que responde a ICMP. Una respuesta `pong` confirma que el inventario y las credenciales SSH están bien configurados antes de lanzar un playbook real.
- Dirigirse a un grupo (`aws`) en vez de a `all` es lo habitual en inventarios reales con varios grupos (`webservers`, `dbservers`...) — permite aplicar comandos/playbooks solo al subconjunto de hosts que corresponde.

### Archivo de configuración (`ansible.cfg`)
- Personaliza comportamiento: ruta del inventario, logging, rutas de módulos/roles/plugins, timeout, nº de forks (concurrencia).
```ini
[defaults]
inventory = /etc/ansible/hosts
log_path  = /var/log/ansible.log
timeout   = 10
forks     = 5
```

### Evitar la verificación manual de host SSH (`host_key_checking`)
Por defecto, SSH pide confirmar manualmente la huella (fingerprint) de cada host nuevo la primera vez que se conecta — algo que interrumpe la ejecución automática de Ansible contra instancias EC2 recién creadas.
```ini
[defaults]
host_key_checking = False
```
- Con esto, Ansible se conecta a hosts nuevos sin pedir esa confirmación interactiva, útil en entornos donde las instancias se crean y destruyen con frecuencia (ej. EC2 efímeras para pruebas).
- ⚠️ Desactivar la verificación de host quita una protección contra ataques de tipo man-in-the-middle — razonable en un entorno de laboratorio/CI controlado, pero es una concesión de seguridad a tener en cuenta, no algo para dejar activado sin más en producción.

### Playbooks
- Archivo YAML que define tareas — es el núcleo de la automatización en Ansible.
- Se tratan como **Infraestructura como Código**: se versionan en Git como cualquier otro código.
```yaml
- name: Playbook 1
  hosts: localhost
  become: yes
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
Para ejecutarlo:
```bash
ansible-playbook -i hosts my-playbook.yaml
```
- `-i hosts` apunta al mismo archivo de inventario usado con `ansible ... -m ping` — el playbook se aplica sobre los hosts/grupos que declare en su campo `hosts:` (ej. `webservers`, `aws`, `localhost`), no sobre todo el inventario indiscriminadamente.
- `become: yes` eleva privilegios (equivalente a `sudo`) para todas las tareas del playbook — necesario para instalar paquetes o gestionar servicios en la mayoría de distribuciones. `yes` y `true` son valores booleanos equivalentes en YAML, así que `become: yes` y `become: true` (visto antes en el ejemplo de Nginx) hacen lo mismo.

### Módulos
- Unidad de código reutilizable que ejecuta una tarea concreta en el nodo remoto (`apt`, `yum`, `file`, `user`...).
- Se pueden crear módulos propios, pero rara vez hace falta — la librería de módulos predeterminados cubre casi todo sin escribir scripts a medida.
- Índice completo de módulos disponibles: https://docs.ansible.com/projects/ansible/latest/collections/index_module.html

### Colecciones
- Formato de empaquetado para distribuir contenido de Ansible junto (playbooks, plugins, módulos...).
- Se pueden crear colecciones propias para proyectos grandes — ej. una colección para automatizar infraestructura cloud (como la AWS Collection).
```bash
# Listar las colecciones instaladas en el sistema
ansible-galaxy collection list
```

### Proyecto práctico: despliegue de una app Angular en EC2
El proyecto real encadena 3 playbooks + un inventario, ejecutados en orden.

**Requisito previo en Windows**: Ansible no puede correr como Control Node nativo en Windows (depende de Python y utilidades de Linux) — hace falta WSL (o una VM/servidor Linux) e instalarlo ahí (`pip3 install --user ansible pywinrm`), junto con `boto3`/`botocore` para los módulos de AWS.

**1) `launch-ec2.yml`** — crea el Security Group y la instancia EC2, usando la colección `amazon.aws` (hay que instalarla con `ansible-galaxy collection install amazon.aws`):
```yaml
- name: Lanzar una instancia EC2 en AWS
  hosts: localhost
  connection: local
  gather_facts: False
  vars:
    aws_region: "<region>"
    security_group_name: "<nombre-security-group>"
    aws_vpc_id: "<vpc-id>"
    aws_image_id: "<ami-id>"
  tasks:
    - name: Crear Grupo de Seguridad para EC2
      amazon.aws.ec2_group:
        name: "{{ security_group_name }}"
        description: "Grupo de seguridad para EC2 con acceso SSH, HTTP, HTTPS"
        region: "{{ aws_region }}"
        vpc_id: "{{ aws_vpc_id }}"
        rules:
          - proto: tcp
            ports: [22, 80, 443]
            cidr_ip: 0.0.0.0/0
      register: sg_info

    - name: Crear una nueva instancia EC2
      amazon.aws.ec2_instance:
        name: "<nombre-instancia>"
        key_name: "<key-pair-name>"
        instance_type: "t2.micro"
        security_group: "{{ security_group_name }}"
        image_id: "{{ aws_image_id }}"
        region: "{{ aws_region }}"
        state: running
      register: ec2_info

    - name: Mostrar la IP pública de la EC2
      debug:
        msg: "EC2 Public IP {{ ec2_info.instances[0].public_ip_address }}"
```
- `hosts: localhost` + `connection: local`: este playbook no se ejecuta *sobre* un servidor remoto — llama a la API de AWS desde el propio Control Node (parecido a usar Boto3 directamente).
- `register` guarda el resultado de una tarea en una variable, para usarlo después (ej. mostrar la IP pública recién asignada).

**2) `inventory.yml`** — una vez tienes la IP de la instancia, se añade a un inventario en formato YAML (alternativa al formato INI ya visto):
```yaml
all:
  hosts:
    ec2_instances:
      ansible_host: <ip-publica-ec2>
      ansible_user: ubuntu
      ansible_ssh_private_key_file: ./ansible_key.pem
      ansible_python_interpreter: /usr/bin/python3.12
```

**3) `setup-node.yml`** — prepara el servidor (Node.js, npm, unzip, nginx):
```yaml
- name: Configurar el servidor y dependencias en EC2
  hosts: ec2_instances
  become: true
  tasks:
    - name: Actualizar paquetes del sistema
      apt:
        update_cache: yes
    - name: Instalar Node.js y npm
      apt:
        name: [nodejs, npm]
        state: present
    - name: Instalar unzip en EC2
      apt:
        name: unzip
        state: present
    - name: Instalar y configurar nginx
      apt:
        name: nginx
        state: present
    - name: Reiniciar Nginx para aplicar cambios
      service:
        name: nginx
        state: restarted
```

**4) `deploy-app.yml`** — copia el artefacto comprimido, lo descomprime, construye la app y la publica en Nginx:
```yaml
- name: Desplegar app Angular en EC2
  hosts: ec2_instances
  become: true
  tasks:
    - name: Copiar la app comprimida al servidor (EC2)
      copy:
        src: files/<app>.zip
        dest: /home/ubuntu/<app>.zip
    - name: Crear la carpeta donde se descomprimirá el proyecto
     file:
      path: /home/ubuntu/<app>
      state: directory
      mode: '0755'
    - name: Descomprimir la app
      ansible.builtin.unarchive:
        src: /home/ubuntu/<app>.zip
        dest: /home/ubuntu/<app>
        remote_src: yes
    - name: Ajustar permisos de la carpeta del proyecto
      file:
        path: /home/ubuntu/<app>
        owner: ubuntu
        group: ubuntu
        mode: '0755'
        recurse: yes
    - name: Ajustar los permisos de los archivos dentro de la carpeta
      command: sudo chown -R ubuntu:ubuntu /home/ubuntu/<app>
    - name: Instalar dependencias de la app Angular
      command: npm install
      args:
        chdir: /home/ubuntu/<app>
    - name: Construir la app Angular
      shell: npm run build --prod
      args:
        chdir: /home/ubuntu/<app>
    - name: Copiar archivos de Angular al servidor Nginx
      copy:
        src: /home/ubuntu/<app>/dist/<nombre-build>/
        dest: /var/www/html/
        remote_src: yes
    - name: Reiniciar Nginx para aplicar los cambios
      service:
        name: nginx
        state: restarted
```

**Orden de ejecución completo:**
```bash
ansible-playbook launch-ec2.yml                       # crea el SG y la instancia
ansible -i inventory.yml all -m ping                  # confirma que la instancia responde
ansible-playbook -i inventory.yml setup-node.yml       # instala Node.js/npm/nginx
ansible-playbook -i inventory.yml deploy-app.yml       # despliega la app compilada
```
- `remote_src: yes` en `copy`/`unarchive` indica que el archivo origen ya está **en el nodo remoto** (no hay que subirlo desde el Control Node) — sin este flag, Ansible buscaría el origen localmente y fallaría.
- `ansible.builtin.unarchive` es el nombre completo (FQCN) del módulo `unarchive` de la colección integrada `ansible.builtin` — usar el nombre completo evita ambigüedad si hay varias colecciones con un módulo del mismo nombre corto.

## Notas y gotchas

- La diferencia central frente a un script Bash no es solo sintaxis — es que Ansible es **idempotente**: ejecutar el mismo playbook dos veces no reinstala/reconfigura de más, solo aplica lo que falta. Un script Bash típico no garantiza eso por sí mismo.
- El nodo de control necesita Python; los nodos gestionados solo necesitan ser alcanzables por SSH/WinRM — no hace falta instalar Ansible ni ningún agente en ellos, es la base del modelo "agentless".
- Windows **no puede ser Control Node** de Ansible (Ansible depende de Python y herramientas de Linux) — sí puede ser un nodo gestionado (Managed Node), pero para ejecutar Ansible desde Windows hace falta WSL, una VM Linux, o un servidor Linux remoto.
- Nunca poner contraseñas en texto plano en el inventario (`ansible_ssh_pass=...`) — está bien para una demo rápida, pero en cualquier proyecto real hay que usar Ansible Vault desde el principio, no añadirlo después.
- Tratar los playbooks como código (control de versiones, revisión de cambios) es lo que realmente distingue "usar Ansible" de "tener un montón de YAMLs sueltos" — la disciplina importa tanto como la herramienta.

## Recursos

- https://docs.ansible.com/projects/ansible/latest/getting_started/index.html
- https://docs.ansible.com/projects/ansible/latest/inventory_guide/intro_inventory.html
- https://docs.ansible.com/projects/ansible/latest/playbook_guide/playbooks_intro.html
- https://docs.ansible.com/projects/ansible/latest/vault_guide/index.html
