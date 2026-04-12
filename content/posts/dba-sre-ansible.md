---
title: Cuando un DBA se cree un SRE. Aprendiendo Ansible
date: 2026-04-12
draft: false
tags:
  - DBRE
  - SRE
  - DBA
  - Ansible
---

Como eterno aprendiz siempre estoy intentando mejorar mis scripts, procesos o flujos de trabajo.  Llevo tiempo leyendo sobre SRE y me llama la atención los pensamientos que se tienen en este rol sobre automatización, pruebas, metodologías, etc. He decidido empezar aprendiendo Ansible y aplicarlo en mi día a día como DBA.

No se me ha ocurrido mejor manera para practicar que implementarlo en tareas repetitivas. Es por eso que en este post voy a pasar los prerrequisitos al SO utilizando ansible antes de instalar GRID, ASM y el RDBMS 19c.

---

## Ansible
Lo primero que he querido aprender es como funciona ansible. Ansible se instala y configura en **máquina controladora** , en mi caso en mi portátil macOS desde donde gestiono todo mi laboratorio. Desde esta máquina controladora se conecta a las máquinas donde se quieren realizar las configuraciones, que son los **nodos gestionados**.

Para instalar ansible en macOS he utilizado `brew`.
```bash
brew install ansible
ansible --version
```

### Requisitos
Para conseguir que todo funcione correctamente he necesitado python y ssh en ambas máquinas.

Lo primero ha sido crear las claves ssh para poder acceder sin utilizar password en las conexiones. 
```bash
ssh-keygen -t rsa -b 4096 -C "ansible-controller"
ssh-copy-id root@<IP-MAQ-DESTINO>
ssh root@<IP-MAQ-DESTINO> "hostname"
```

En la máquina destino he tenido que instalar python 3.9, ya que la versión que venía con el sistema Oracle Linux 8.10 me estaba dando problemas en las pruebas realizadas.
```bash
dnf install python39 -y
```

## Preparación de la magia
En la maquina controladora he creado los siguientes directorios para ir generando los ficheros **.yml**.
```bash
mkdir -p ~/Documents/oracle-iac/ansible
cd ~/Documents/oracle-iac/ansible
mkdir -p inventories/dev/group_vars
mkdir -p roles/oracle_prereqs/{tasks,defaults,templates}
mkdir -p playbooks
```

Para verlo más gráficamente añado el arbol de directorios resultante:
```
~/Documents/oracle-iac/ansible
├── inventories
│   └── dev
│       └── group_vars
├── playbooks
└── roles
    └── oracle_prereqs
        └── tasks
        └── defaults
        └── templates
```

Para que la magia suceda tengo que indicarle a ansible algunas cosas en ficheros de configuración. 
###  Fichero de configuración ansible.cfg
Con este fichero estoy indicando a ansible donde está el inventario de las máquinas a las que se tiene que conectar, que usuario utilizar y como escalar privilegios.

```bash
cat > ~/Documents/oracle-iac/ansible/ansible.cfg << 'EOF'
[defaults]
inventory       = inventories/dev/hosts.yml
remote_user     = oracle
host_key_checking = False          
roles_path      = roles
result_format = yaml

[privilege_escalation]
become          = True
become_method   = sudo
become_user     = root
EOF
```

### Fichero de configuración hosts.yml
En este fichero le enseño a ansible cual es mi infraestructura, en este caso le indico la máquina destino donde quiero hacer las configuraciones. Se pueden tener servidores en diferentes **entornos** como dev, pre y pro por ejemplo. Al estar aprendiendo, de momento yo utilizo entorno dev.

```bash
cat > ~/Documents/oracle-iac/ansible/inventories/dev/hosts.yml << 'EOF'
---
all:
  children:
    oracle_servers:
      hosts:
        db-server-01:
          ansible_host: <IP-MAQ-DESTINO>
          ansible_user: root
          ansible_ssh_private_key_file: ~/.ssh/id_rsa
EOF
```

Antes de continuar explicando a ansible lo que quiero que haga en esa máquina voy a comprobar que es capaz de conectarse a ella.
```bash
ansible oracle_servers -m ping
```

Si todo va bien, contestará con un ping-pong.
```bash
db-server-01 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3.9"
    },
    "changed": false,
    "ping": "pong"
}
```

### Fichero de variables 
Fichero donde indico los directorios y usuarios.
```bash
cat > ~/Documents/oracle-iac/ansible/inventories/dev/group_vars/oracle_servers.yml << 'EOF'
---
# ============================================================
# USUARIOS Y GRUPOS ORACLE
# ============================================================
oracle_user: oracle
grid_user: grid
oracle_group: oinstall

# ============================================================
# DIRECTORIOS BASE
# ============================================================
oracle_base: /u01/app/oracle
grid_base: /u01/app/grid
oracle_inventory: /u01/app/oraInventory
grid_home: /u01/app/19.3.0/grid
oracle_home: /u01/app/oracle/product/19.3.0/dbhome_1
EOF
```

En este fichero especifico los id para los diferentes usuarios y grupos que voy a generar a continuación.
```bash
cat > ~/Documents/oracle-iac/ansible/roles/oracle_prereqs/defaults/main.yml << 'EOF'
---
swap_size_mb: 8192
oracle_user_uid: 54321
grid_user_uid: 54331
oinstall_gid: 54321
dba_gid: 54322
oper_gid: 54323
asmdba_gid: 54327
asmoper_gid: 54328
asmadmin_gid: 54329
EOF
```

Ficheros donde indico los profiles a modo plantilla para que los genere en el destino.
```bash
cat > ~/Documents/oracle-iac/ansible/roles/oracle_prereqs/templates/oracle_bash_profile.j2 << 'EOF'
# Oracle User Environment
export ORACLE_BASE={{ oracle_base }}
export ORACLE_HOME={{ oracle_home }}
export ORACLE_SID={{ db_name }}
export PATH=$ORACLE_HOME/bin:$PATH
export LD_LIBRARY_PATH=$ORACLE_HOME/lib:/lib:/usr/lib
export CLASSPATH=$ORACLE_HOME/jlib:$ORACLE_HOME/rdbms/jlib
export NLS_DATE_FORMAT="DD-MON-YYYY HH24:MI:SS"
export NLS_LANG=AMERICAN_AMERICA.AL32UTF8

# Alias útiles
alias sqlp='sqlplus / as sysdba'
alias alert='tail -200f $ORACLE_BASE/diag/rdbms/$(echo $ORACLE_SID | tr [:upper:] [:lower:])/$ORACLE_SID/trace/alert_$ORACLE_SID.log'
EOF

cat > ~/Documents/oracle-iac/ansible/roles/oracle_prereqs/templates/grid_bash_profile.j2 << 'EOF'
# Grid Infrastructure User Environment
export ORACLE_BASE={{ grid_base }}
export ORACLE_HOME={{ grid_home }}
export PATH=$ORACLE_HOME/bin:$PATH
export LD_LIBRARY_PATH=$ORACLE_HOME/lib:/lib:/usr/lib

# Alias útiles
alias crsstat='crsctl stat res -t'
alias asmcmd='asmcmd'
EOF
```
### Fichero de tareas main.yml
Este fichero es donde indico todo lo que quiero que se haga de manera automática.
```bash
cat > ~/Documents/oracle-iac/ansible/roles/oracle_prereqs/tasks/main.yml << 'EOF'
---
# ============================================================
# TAREA 1: Instalar el preinstall de Oracle Linux
# ============================================================
- name: "PREREQS | Instalar oracle-database-preinstall-19c"
  ansible.builtin.command: >
      dnf install -y
      oracle-database-preinstall-19c
  tags: prereqs

# ============================================================
# TAREA 2: Crear grupos del sistema para Oracle
# ============================================================
- name: "PREREQS | Crear grupos Oracle"
  ansible.builtin.group:
    name: "{{ item.name }}"
    gid: "{{ item.gid }}"
    state: present
  loop:
    - { name: oinstall,  gid: "{{ oinstall_gid }}" }
    - { name: dba,       gid: "{{ dba_gid }}" }
    - { name: oper,      gid: "{{ oper_gid }}" }
    - { name: asmdba,    gid: "{{ asmdba_gid }}" }
    - { name: asmoper,   gid: "{{ asmoper_gid }}" }
    - { name: asmadmin,  gid: "{{ asmadmin_gid }}" }
  tags: prereqs

# ============================================================
# TAREA 3: Crear usuario oracle
# ============================================================
- name: "PREREQS | Crear usuario oracle"
  ansible.builtin.user:
    name: oracle
    uid: "{{ oracle_user_uid }}"
    group: oinstall
    groups: "dba,oper,asmdba"
    home: /home/oracle
    shell: /bin/bash
    comment: "Oracle Software Owner"
    state: present
  tags: prereqs

# ============================================================
# TAREA 4: Crear usuario grid
# ============================================================
- name: "PREREQS | Crear usuario grid"
  ansible.builtin.user:
    name: grid
    uid: "{{ grid_user_uid }}"
    group: oinstall
    groups: "asmdba,asmoper,asmadmin"
    home: /home/grid
    shell: /bin/bash
    comment: "Oracle Grid Infrastructure Owner"
    state: present
  tags: prereqs

# ============================================================
# TAREA 5: Crear la estructura de directorios
# ============================================================
- name: "PREREQS | Crear directorios Oracle"
  ansible.builtin.file:
    path: "{{ item.path }}"
    state: directory
    owner: "{{ item.owner }}"
    group: oinstall
    mode: "0755"
  loop:
    - { path: "/u01",                          owner: root }
    - { path: "/u01/app",                      owner: oracle }
    - { path: "{{ oracle_inventory }}",        owner: oracle }
    - { path: "{{ oracle_base }}",             owner: oracle }
    - { path: "{{ oracle_home }}",             owner: oracle }
    - { path: "{{ grid_base }}",               owner: grid }
    - { path: "{{ grid_home }}",               owner: grid }
    - { path: "{{ sw_stage_dir }}",            owner: root }
  tags: prereqs

# ============================================================
# TAREA 6: .bash_profile para usuario oracle
# ============================================================
- name: "PREREQS | Configurar .bash_profile de oracle"
  ansible.builtin.template:
    src: oracle_bash_profile.j2
    dest: /home/oracle/.bash_profile
    owner: oracle
    group: oinstall
    mode: "0644"
  tags: prereqs

# ============================================================
# TAREA 7: .bash_profile para usuario grid
# ============================================================
- name: "PREREQS | Configurar .bash_profile de grid"
  ansible.builtin.template:
    src: grid_bash_profile.j2
    dest: /home/grid/.bash_profile
    owner: grid
    group: oinstall
    mode: "0644"
  tags: prereqs

# ============================================================
# TAREA 8: Deshabilitar SELinux 
# ============================================================
- name: "PREREQS | Poner SELinux en permissive (runtime)"
  ansible.builtin.command: >
      setenforce Permissive
  tags: prereqs

- name: "PREREQS | Poner SELinux en permissive (persistente)"
  ansible.builtin.replace:
    path: /etc/selinux/config
    regexp: '^SELINUX=.*'
    replace: 'SELINUX=permissive'
  tags: prereqs

# ============================================================
# TAREA 9: Deshabilitar el firewall
# ============================================================
- name: "PREREQS | Parar firewall"
  ansible.builtin.command: >
      systemctl stop firewalld
  tags: prereqs
  
- name: "PREREQS | Deshabilitar firewall"
  ansible.builtin.command: >
      systemctl disable firewalld
  tags: prereqs
EOF
```

## Donde sucede la magia, el playbook
Una vez que he preparado todo indicando variables, usuarios, etc es hora de decirle a ansible que quiero que me ejecute y donde. Para ello se necesita ejecutar un playbook.

Al ser de prerrequisitos lo he llamado 01_prereqs.yml
```bash
cat > ~/Documents/oracle-iac/ansible/playbooks/01_prereqs.yml << 'EOF'
---
- name: "PLAYBOOK 1 | Configurar prerequisitos Oracle en el SO"
  hosts: oracle_servers
  become: true

  roles:
    - role: oracle_prereqs
      tags: prereqs
EOF
```

Para ejecutar el playbook utilizo -vv para que haga el verbose y tags como buena practica para que ejecute lo que he puesto con esos tags.

```bash
ansible-playbook playbooks/01_prereqs.yml -vv --tags prereqs
```

> Nota: Los tags son heredados por los roles, es decir, si dentro de un rol tengo varios tags, al ejecutar el rol ejecutará todo lo que hay por debajo.

Lo que me ha servido para aprender esto es ver los tags que tiene cada tarea `ansible-playbook playbooks/01_prereqs.yml --list-tasks`

A modo de ejemplo añado la salida "recortada" que he obtenido tras ejecutar esto.
```bash
ansible-playbook playbooks/01_prereqs.yml -vv --tags prereqs

ansible-playbook [core 2.20.4]
  config file = /Users/jesus/Documents/oracle-iac/ansible/ansible.cfg

PLAYBOOK: 01_prereqs.yml ***********************************************************************************************************************************************************
1 plays in playbooks/01_prereqs.yml

PLAY [PLAYBOOK 1 | Configurar prerequisitos Oracle en el SO] ***********************************************************************************************************************

TASK [Gathering Facts] *************************************************************************************************************************************************************
task path: /Users/jesus/Documents/oracle-iac/ansible/playbooks/01_prereqs.yml:2
ok: [db-server-01]

TASK [oracle_prereqs : PREREQS | Instalar oracle-database-preinstall-19c] **********************************************************************************************************
task path: /Users/jesus/Documents/oracle-iac/ansible/roles/oracle_prereqs/tasks/main.yml:7
changed: [db-server-01] => {"changed": true, "cmd": ["dnf", "install", "-y", "oracle-database-preinstall-19c"], "delta": "0:00:02.603811", "end": "2026-04-10 14:13:39.720259", "msg": "", "rc": 0, "start": "2026-04-10 14:13:37.116448", "stderr": "", "stderr_lines": [], "stdout": "Last metadata expiration check: 1:49:47 ago on Fri 10 Apr 2026 12:23:50 PM WEST.\nPackage oracle-database-preinstall-19c-1.0-2.el8.x86_64 is already installed.\nDependencies resolved.\nNothing to do.\nComplete!", "stdout_lines": ["Last metadata expiration check: 1:49:47 ago on Fri 10 Apr 2026 12:23:50 PM WEST.", "Package oracle-database-preinstall-19c-1.0-2.el8.x86_64 is already installed.", "Dependencies resolved.", "Nothing to do.", "Complete!"]}
.
.
.
TASK [oracle_prereqs : PREREQS | Deshabilitar firewall] ****************************************************************************************************************************
task path: /Users/jesus/Documents/oracle-iac/ansible/roles/oracle_prereqs/tasks/main.yml:135
changed: [db-server-01] => {"changed": true, "cmd": ["systemctl", "disable", "firewalld"], "delta": "0:00:00.081855", "end": "2026-04-10 14:13:48.866159", "msg": "", "rc": 0, "start": "2026-04-10 14:13:48.784304", "stderr": "", "stderr_lines": [], "stdout": "", "stdout_lines": []}

PLAY RECAP *************************************************************************************************************************************************************************
db-server-01               : ok=12   changed=4    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

En este punto pensaba que era muy bonito para ser verdad. Tenía que comprobar que efectivamente había hecho cosas. Me conecté al servidor y comprobé los usuarios y la modificación del SELinux.
```bash
[root@localhost ~]# id oracle
uid=54321(oracle) gid=54321(oinstall) groups=54321(oinstall),54322(dba),54323(oper),54327(asmdba)
[root@localhost ~]# id grid
uid=54331(grid) gid=54321(oinstall) groups=54321(oinstall),54327(asmdba),54328(asmoper),54329(asmadmin)
[root@localhost ~]# cat /etc/selinux/config | grep -i "SELINUX="
# SELINUX= can take one of these three values:
SELINUX=permissive
```

Y esto me lleva a pensar la de horas que he utilizado para preparar diferentes servidores, nodos de cluster como dba tradicional.

Como eterno aprendiz, ahora que estoy aprendiendo ansible no solo ahorraré tiempo al hacer despliegues o configuraciones en diferentes nodos.
También reduciré la posibilidad de errores en tareas repetitivas.

Cuando el DBA tradicional empieza a integrar conceptos de SRE nace el DBRE.

Ser DBA es cosa del pasado.

DBA + SRE = DBRE



