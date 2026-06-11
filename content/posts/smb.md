---
title: Solución timemachine cuando no puedes conectar usb externos
date: 2026-06-07
draft: false
tags:
  - redes
---

---
Tengo la manía de hacer backup de mis ficheros, sobre todo aquello que me ha costado ir creando con el paso del tiempo.

En macOS utilizaba timemachine, y digo utilizaba, por que hace unos meses por políticas de seguridad dejé de tener permitido conectar usb externos al ordenador.

No he dejado de hacer backups, pero no de la manera en la que los hacía. 

He pasado por varias etapas y os voy a contar como he llegado a la definitiva.

### La primera etapa

Lo que se me ocurrió en este momento fue hacer un rsync desde mis mac a un directorio de mi otro portátil. Esto parecía guay.

Me sacó del apuro, tenía mis datos en dos dispositivos y desde mi portátil utilizaba timemachine.

Lo que no me convencía de este método es que tenía la necesidad de tener ambos ordenadores encendidos para poder hacer la copia. Necesitaba una logística que me quitaba tiempo para hacer un simple backup.

### La segunda etapa
Esta etapa llega mientras sigo aprendiendo con cursos, formaciones de Linux, protocolos, etc.

Ya conocía el protocolo NFS, pero no lo había utilizado.

Esta vez dije, tengo un servidor encendido 24/7, quizá sea buen sitio para desplegar un disco con NFS y no tener la necesidad de encender los dos ordenadores a la vez para poder hacer un rsync.

Ahora lo hago al NFS.

Lo bueno de esta etapa es que la sigo utilizando, pero no como backup, si no como repositorio compartido en mi red.

Para la toma de notas utilizo obsidian. Ahora gracias a este directorio de  NFS soy capaz de tener mi repositorio de obsidian sincronizado entre mis dispositivos.

Me he creado unos alias en ambos ordenadores que hacen un "push" y un "pull" del directorio de obsidian. Consiste en un rsync que sube lo del ordenador al directorio y otro que lo descarga.  De esta manera, cuando acabo de trabajar en un portátil, hago `pushobs`.

Cuando enciendo el otro portátil, hago `pullobs`.

### La tercera etapa
La última que he añadido a mi estrategia de backups. 

Hablo de la utilización del protocolo SMB para compartir almacenamiento en red. Una vez más, la curiosidad y el querer seguir aprendiendo me ha permitido llegar a esta solución.

Timemachine soporta la utilización de directorios SMB para hacer sus copias de seguridad.

No será la mejor, ya que tengo dos discos SSDs conectados al servidor donde utilizo proxmox.

He generado una VM limpia con Ubuntu solo para compartir almacenamiento.

Lo ideal sería un NAS, pero de momento la solución para mi estrategia de backups está cubierta.

Es por esto que estoy creando este post, no para enseñar nada, pero si para transmitir que gracias al seguir aprendiendo cada día he podido diseñar una solución que se adapta a mis necesidades, con cosas que tenía por casa.

Voy a dejar los pasos que he dado para generar el repositorio SMB y poder hacer copias con timemachine por si a alguien más le puede servir.

Parto de que se tiene una máquina Ubuntu. 

**Instalación de paquetes necesarios**
```bash
apt install samba cifs-utils
```

**Crear grupo y usuario especial para utilizar el timemachine**
```bash
sudo groupadd timemachine
sudo useradd -m -s /bin/bash -g timemachine timemachine
sudo passwd timemachine
smbpasswd -a timemachine
```

**Crear directorio**
```bash
# Carpeta Compartida
sudo mkdir -p /timemachine
sudo chmod 2770 /timemachine
sudo chown root:timemachine /timemachine
```

**Generar el directorio utilizando el disco SSD**
```bash
lsblk
mkfs.ext4 /dev/sdb1
mount /dev/sdb1 /timemachine/

#Modificar el fstab para hacer permanente el mount
```

**Modificar fichero de configuración de Samba con los siguientes parámetros**
```bash
sudo vi /etc/samba/smb.conf

[timemachine]
   path = /timemachine
   browseable = yes
   writable = yes
   guest ok = no
   create mask = 0664
   directory mask = 2770
   valid users = @timemachine
   force group = timemachine
   force create mode = 0664
   force directory mode = 2770
   # Opciones específicas para Time Machine
   fruit:time machine = yes
   fruit:time machine max size = 150G
   vfs objects = catia fruit streams_xattr
```

**Reiniciar samba**
```bash
systemctl restart smbd
```

El siguiente paso es conectar el macOS al servidor de timemachine que acabamos de crear. Para ello se pueden seguir los siguientes pasos:

Desde el mac>finder>go>connect to server>introducir datos (IP y usuario del ubuntu). 
Después desde la configuración time machine dejará añadir este destino para hacer los backups.

---

*Esto es un blog personal con carácter informativo. Puede contener errores.*

