---
title: Homelab para aprender kubernetes - fase 0
date: 2026-05-17
draft: false
tags:
  - kubernetes
  - proxmox
---

---
Hace tiempo estuve trasteando con k3s y probé muy poco de kubernetes, me quedé con buen sabor de boca. Pero quería más.

En ese momento algo me decía que tenía que seguir mejorando mi conocimiento sobre contenedores, linux, cgroups, etc. 

Tras estos meses en los que no he parado de aprender, ha llegado el momento de montar un cluster para aprender kubernetes y quien sabe si traerá alguna certificación.

**¿Por qué kubernetes?** 
Es el estándar actualmente para montar microservicios y cloud. Hay mucha tecnología pivotando hacia eso. Me permite conocer más sobre infraestructura y tener un conocimiento más transversal, debido al conocimiento en base de datos que tengo esto es un añadido, no un punto y aparte.

Por lo que he comentado anteriormente he decidido documentar el camino de aprendizaje de kubernetes.

Voy a utilizar uno de mis minipc para empezar a montar las VMs para desplegar kubernetes. El sistema operativo elegido ha sido Ubuntu, ya que es uno de los más elegidos para cloud native. 

>Nota: De momento voy a depslegar un control-plane y un worker. Ya tendré tiempo de ir escalando según necesidad y aprendizaje. Me gustaría llegar al punto de que el cluster autoescale y haga downsize según necesidad de computación. Pero voy a empezar desde abajo.
## Crear plantilla Ubuntu en proxmox con cloud-init

En este punto ya tengo promox funcionando en uno de mis minipc debido a que llevo tiempo con el homelab por lo que no voy a comentar nada sobre la instalación y configuración de proxmox.

Voy a utilizar cloud-init para desplegar las VMs de Ubuntu, esto me permitirá no tener que pasar por el proceso de instalación cada vez que necesite una nueva VM. 

El generar una plantilla me aporta rapidez en la creación de VMs ya que se clona en segundos.

### ¿Qué es cloud-init?
Es el estándar para el despliegue de instancias en la nube.

Esto permite configurar:
- hostname
- red
- crear usuarios y ssh keys
- aumentar el disco
- lanzar scripts

Esto me permite empezar automatizando despliegues de VMs.

### Bajar imagen ubuntu cloud

Voy a intentar hacer lo máximo posible desde el terminal, ya que para automatizaciones me vendrá mejor conocer los comandos. Hasta ahora he trabajado mucho desde el GUI de proxmox.

Accedo por ssh al servidor de proxmox:
```bash
ssh root@192.168.1.210
```

Ahora descargo la imagen de ubuntu:
```bash
cd /var/lib/vz/template/iso/  
# Descargar Ubuntu 24.04 LTS cloud image  
wget https://cloud-images.ubuntu.com/noble/current/noble-server-cloudimg-amd64.img
# Comprobar  
ls -lh noble-server-cloudimg-amd64.img
```

### Crear la VM plantilla
Ahora voy a crear una VM y convertirla en plantilla:
```bash
# Crear VM (ID 1000)  
qm create 1000 --name ubuntu-24.04-template --memory 2048 --cores 2 --net0 virtio,bridge=vmbr0  
# Importar la cloud image al disco de la VM
qm importdisk 1000 /var/lib/vz/template/iso/noble-server-cloudimg-amd64.img local-lvm  
# Añadir el disco a la VM  
qm set 1000 --scsihw virtio-scsi-pci --scsi0 local-lvm:vm-1000-disk-0  
# Añadir el drive de Cloud-Init CD-ROM  
qm set 1000 --ide2 local-lvm:cloudinit  
# Arrancar desde el disco  
qm set 1000 --boot c --bootdisk scsi0  
# Añadir la consola (nos puede ayudar en problemas)  
qm set 1000 --serial0 socket --vga serial0  
# Habilitar QEMU Guest Agent  
qm set 1000 --agent enabled=1
```

### Configurar el cloud-init
En este punto comienzo con la modificación de los parámetros del cloud-init:
```bash
# Defino el usuario  
qm set 1000 --ciuser remote  
# Pongo una pass por defecto para tenerla, ya que entraré con ssh keys.
qm set 1000 --cipassword $(openssl passwd -6 "remote")  
# Setear las ssh keys. Esto es pasando desde mi macOS al proxmox para que la setee en el cloud-init
qm set 1000 --sshkeys ~/.ssh/id_ed25519.pub  
# Usar el DHCP ya que la IP la daré al clonar  
qm set 1000 --ipconfig0 ip=dhcp
```

### Convertir a template
```bash
qm template 1000
```

### Script para crear un template
Para ir teniendo todo automatizado y poder volver a repetir los mismos pasos he creado el siguiente script. Aprovechando la creación de script y mi introducción en el mundo cloud-native he abierto una cuenta de github donde estoy subiendo mis cositas. 

https://github.com/jesusroodriguez/devops-homelab-project/tree/master/scripts

> Nota: El script de github es con 3 worker debido a que sería el HA por default que me gustaría montar en futuro, pero no quiero perderme el crecimiento desde un worker a los tres.

```bash
#!/bin/bash
# create-template.sh
# Crea la VM template de ubuntu 24.04
# Este script se lanza desde el host proxmox (192.168.1.210)

set -e

TEMPLATE_ID=1000
IMAGE_URL="https://cloud-images.ubuntu.com/noble/current/noble-server-cloudimg-amd64.img"
IMAGE_DIR="/var/lib/vz/template/iso"
IMAGE_FILE="noble-server-cloudimg-amd64.img"
STORAGE="local-lvm"

echo "====================================="
echo "Creando la VM template Ubuntu 24.04"
echo "Template ID: $TEMPLATE_ID"
echo "====================================="

# Descargar la imagen
echo ""
echo "Descargando Ubuntu 24.04 cloud image..."
cd "$IMAGE_DIR"
if [ -f "$IMAGE_FILE" ]; then
    echo "Ya existe la imagen. Se salta la descarga."
else
    wget "$IMAGE_URL"
    echo "Descarga completa: $(ls -lh $IMAGE_FILE)"
fi

# Crear VM
echo ""
echo "Creando el template..."
qm create $TEMPLATE_ID \
    --name ubuntu-24.04-template \
    --memory 2048 \
    --cores 2 \
    --net0 virtio,bridge=vmbr0

# Importo el disco de cloud init
echo ""
echo "Importando el disco de cloud init a la VM..."
qm importdisk $TEMPLATE_ID "$IMAGE_DIR/$IMAGE_FILE" $STORAGE

# Configuro la VM
echo ""
echo "Configurando VM..."
qm set $TEMPLATE_ID --scsihw virtio-scsi-pci --scsi0 $STORAGE:vm-$TEMPLATE_ID-disk-0
qm set $TEMPLATE_ID --ide2 $STORAGE:cloudinit
qm set $TEMPLATE_ID --boot c --bootdisk scsi0
qm set $TEMPLATE_ID --serial0 socket --vga serial0
qm set $TEMPLATE_ID --agent enabled=1

# Configuro los defaults de Cloud-Init
echo ""
echo "Configurando defaults de cloud init..."
qm set $TEMPLATE_ID --ciuser remote
qm set $TEMPLATE_ID --cipassword $(openssl passwd -6 "remote")

# Configuro las ssh keys
if [ -f ~/.ssh/id_ed25519.pub ]; then
    echo "Utilizo la ssh key de ~/.ssh/id_ed25519.pub"
    qm set $TEMPLATE_ID --sshkeys ~/.ssh/id_ed25519.pub
else
    echo "WARNING: No se ha encontrado la ssh key: Añadela a mano"
    echo "  qm set $TEMPLATE_ID --sshkeys ~/.ssh/ssh_key.pub"
fi

qm set $TEMPLATE_ID --ipconfig0 ip=dhcp

# Convierto a template
echo ""
echo "Convirtiendo la VM en template..."
qm template $TEMPLATE_ID

echo ""
echo "====================================="
echo "✓ Plantilla creada satisfactoriamente!"
echo "====================================="
echo "Template ID: $TEMPLATE_ID"
echo "Template Name: ubuntu-24.04-template"
echo ""
```

Para probar el funcionamiento del script he borrado el template generado y lo he creado con el script.

### Creación de las VMs
Ahora voy a crear las VMs del control-plane y el worker utilizando el template que he creado en el punto anterior.

```bash
# control-plane  
qm clone 1000 220 --name k8s-master --full  
qm set 220 --memory 4096 --cores 2  
qm resize 220 scsi0 +46.5G  
qm set 220 --ipconfig0 ip=192.168.1.220/24,gw=192.168.1.1  
qm set 220 --nameserver 192.168.1.1  
qm set 220 --searchdomain localdomain  
# worker-01  
qm clone 1000 221 --name k8s-worker-01 --full  
qm set 221 --memory 4096 --cores 2  
qm resize 221 scsi0 +46.5G
qm set 221 --ipconfig0 ip=192.168.1.221/24,gw=192.168.1.1  
qm set 221 --nameserver 192.168.1.1  
qm set 221 --searchdomain localdomain  
```

## Primer arranque
Ha llegado el momento de cruzar los dedos y ver que todo esto es capaz de levantar.

Para arrancar y ver el estado:
```bash
# Arrancar el cluster de kubernetes
qm start 220 # k8s-master  
qm start 221 # k8s-worker-01  
# Comprobar estado  
qm status 220  
qm status 221  
```

El estado es running. La primera vez tarda un rato haciendo el despliegue del cloud-init.

Una vez que han arrancado del todo compruebo que hago ping a las ips que he asignado en la creación.
```bash
ping -c 3 192.168.1.220  
ping -c 3 192.168.1.221  
```

Las latencias que me dan desde el macbook son las siguientes:
```bash
64 bytes from 192.168.1.220: icmp_seq=0 ttl=64 time=1.778 ms
64 bytes from 192.168.1.220: icmp_seq=1 ttl=64 time=1.435 ms
64 bytes from 192.168.1.220: icmp_seq=2 ttl=64 time=1.491 ms
```

Ahora voy a comprobar que funcioan el ssh sin contraseña gracias a las ssh keys y de paso compruebo el estado del cloud-init.
```bash
ssh remote@192.168.1.220
sudo cloud-init status
```

El status aparece en done. Hay que repetir esto en el resto de nodos.

Yo lo he probado con un bucle:
```bash
for i in 0 1  
do
ssh remote@192.168.1.22$i "hostname; sudo cloud-init status"
done
```

Ya están desplegadas las VMs para kubernetes y el template generado.

```bash
#!/bin/bash
# create-k8s-vms.sh
# Crea las VMs para el cluster de kubernetes
# Este script se lanza desde el host proxmox (192.168.1.210)

set -e

# Defino las variables comunes
TEMPLATE_ID=1000
SSH_KEY="ssh-ed25519 XXXX <-- Tu clave"
GATEWAY="192.168.1.1"
NAMESERVER="192.168.1.1"
SEARCH_DOMAIN="localdomain"

# Creo una función de como se ejecutarán los comandos
create_vm() {
    local vmid=$1 name=$2 ip=$3 memory=$4 cores=$5 disk=$6
    echo "Creando la VM $vmid: $name..."
    qm clone $TEMPLATE_ID $vmid --name $name --full
    qm set $vmid --memory $memory --cores $cores
    qm resize $vmid scsi0 "${disk}G"
    qm set $vmid --ipconfig0 "ip=${ip}/24,gw=$GATEWAY"
    qm set $vmid --nameserver $NAMESERVER 
    qm set $vmid --searchdomain $SEARCH_DOMAIN
    qm set $vmid --sshkeys <(echo "$SSH_KEY")
    echo "VM $vmid creada"
}

# Lanzo los comandos anteriores pasandole las variables para cada VM
create_vm 220 "k8s-master"     "192.168.1.220" 4096 2 50
create_vm 221 "k8s-worker-01"  "192.168.1.221" 4096 2 50

# Para arrancar las VMs
echo "Arrancar todas las VMs con el siguiente comando:"
echo "for i in 0 1 do; qm start 22\$i; done"
```

## Conclusión
En este post he conseguido montar dos VMs utilizando cloud-init. Estas dos VMs serán las bases para el clúster de kubernetes, una el control-plane y otra el worker. En el siguiente post de esta serie contaré como hacer hardening a proxmox y como he preparado las snapshots y backups de estas VMs.

---

*Las opiniones y contenidos de este blog son míos y no representan a Oracle.*


