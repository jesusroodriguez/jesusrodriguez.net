---
title: Homelab para aprender kubernetes - fase 1
date: 2026-05-24
draft: false
tags:
  - kubernetes
  - proxmox
---

---
## Snapshots, backups y hardening

### Snapshots
En proxmox se pueden hacer snapshots, que suelen tardar segundos en hacerse.

El poder hacer snapshots me permite tener puntos restauración temporales para volver después de una prueba. Puedo hacer una snapshot antes de instalar o modificar nada y curarme en salud. Para probar Disastre recovery.

Las snapshots solo guardan los cambios realizados, por lo que en cuanto a la utilización espacio son eficientes.

> Nota: Las snapshots no sustituyen a los backups.

Para hacer las snapshots he utilizado los siguientes comandos:
```bash
qm snapshot 220 before-hardening --description "20260507 - Clean VM before hardening" 
qm snapshot 221 before-hardening --description "20260507 - Clean VM before hardening" 
```

Como buena práctica recomendaría poner la fecha en la que se hacen las snapshots.

He creado el siguiente script para poder hacer los snapshots al automáticamente.
```bash
#!/bin/bash  
# snapshot-k8s-vms.sh  
# Snapshots de todas las VMs del cluster de kubernetes

set -e

#Chequeo la entrada de los argumentos
if [ "$#" -ne 2 ];  then 
	echo "Uso: $0 <argumento1> <argumento2>" 
	exit 1 
fi

#Capturo los argumentos y ejecuto las snapshots
NOMBRE_SNAPSHOT="${1}"  
DESCRIPCION="${2}"  
for i in 0 1; do  
echo "Creando snapshot '$SNAPSHOT_NAME' para la VM 22$i..."  
qm snapshot 22$i "$SNAPSHOT_NAME" --description "$DESCRIPTION"  
done  
echo "Todas las snapshots creadas correctamente"
```

### Backups
Lo bueno de los backups en proxmox es que se pueden automatizar con el scheduler, tienen compresión y se pueden hacer en diferentes almacenamientos. 

Para hacer el primer backup de cada VM utilizo el siguiente comando:
```bash
# Backup all K8s VMs  
vzdump 220 221 --compress zstd --mode snapshot --storage local
```

Los backups en modo snapshots es una funcionalidad que se puede hacer teniendo instalados los agentes qemu en las VMs y hace un backup consistente aún estando la VM corriendo.

Para automatizar los backups con el scheduler lo hago desde la ventana gráfica ya que no he encontrado como hacerlo desde el terminal. Añado captura de imagen:
![minipc](/images/scheduler-backups-proxmox.png)

Yo he creado una tarea para que se ejecute todos los domingos a las 4 de la mañana y tenga una retención de los cuatro últimos backups.

Como best practice para los backups, recomiendo la estrategia 3-2-1. 
- 3 copias de los datos.
- 2 copias en sitios diferentes.
- 1 copia offsite.

Me falta por solventar la copia offsite, que voy a enchufar un ssd externo al minipc de vez en cuando para hacer copia de los backups ahí.

### Hardening
Antes de ponerme en materia a instalar y hacer pruebas y aprender voy a empezar a securizar todo desde la base.

Lo primero es mantener actualizado el hipervisor proxmox:
```bash
apt update  
apt dist-upgrade -y
pve-efiboot-tool refresh
```

Lo siguiente que hago es denegar el acceso ssh mediante contraseña a root. Antes de nada recordad pasaros la key desde vuestro workstation, en mi caso desde el mac.

```bash
ssh-copy-id root@192.168.1.210
```

```bash
vi /etc/ssh/sshd_config
```

```txt
PermitRootLogin prohibit-password  
PasswordAuthentication no  
PubkeyAuthentication yes
```

Una vez hecho los cambios reiniciar el ssh:
```bash
systemctl restart sshd
```

Permitir solo el acceso a la interfaz de proxmox desde mi red local:
```bash
vi /etc/default/pveproxy
```

```txt
ALLOW_FROM="192.168.1.0/24"
```

Reiniciar el proxy.
```bash
systemctl restart pveproxy
```

Activar el firewall de proxmox a nivel datacenter. Con los siguientes comandos lo que hago es permitir el tráfico hacia los puertos 22 y 8006, para ssh y la interfaz web respectivamente.
```bash
pvesh set /cluster/firewall/options --enable 1  
pvesh create /cluster/firewall/rules --type in --action ACCEPT --proto tcp --dport 22 --comment "Permitir SSH"   
pvesh create /cluster/firewall/rules --type in --action ACCEPT --proto tcp --dport 8006 --comment "Permitir Proxmox GUI"
```

---

*Las opiniones y contenidos de este blog son míos y no representan a Oracle.*


