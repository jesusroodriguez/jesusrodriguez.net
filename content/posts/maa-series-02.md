---
title: MAA Series 02 - Instalación ASM+GRID 19c
date: 2026-04-26
draft: false
tags:
  - ASM
  - GRID
  - MAA
---

---
En este post voy a detallar los pasos que he dado para realizar la instalación de asm y grid en la máquina Oracle Linux 8.10 que tengo para el laboratorio de MAA.

La decisión de instalar GRID 19c es por que considero que a día de hoy es una de las versiones más usadas, que me va a dar juego para hacer pruebas de multitenant, migraciones, etc. 

El GRID hay que instalarlo para tener Oracle Restart en la base de datos que montaré más adelante. Oracle Restart es una de las best practices comentadas en este post https://jesusrodriguez.net/posts/fundamentos-bronze-maa/ .

## ASM
---
Para instalar ASM existen varias maneras, yo he optado por utilizar **asmlib**, ya que es la forma actual y recomendada para instalar ASM en el sistema. **ASMFD** es la forma antigua, deprecada desde Oracle Database 19c.

>⚠️ Este post se ha desarrollado con las versiones asmlib v3.1, oracle linux 8.10, kernel 5.15.0-318.199.3.2.1.el8uek.x86_64.

Al no tener suscripción ULN hay que bajar los paquetes a mano desde la siguiente web:
>https://www.oracle.com/linux/downloads/linux-asmlib-v8-downloads.html#

El paquete de **oracleasm-support** se puede descargar desde la web: 
>https://yum.oracle.com/repo/OracleLinux/OL8/addons/x86_64/index.html

## Instalación ASM
Una vez que están los paquetes en la máquina, se instalan con los siguientes comandos:
```sh
rpm -ivh oracleasm-support-3.1.1-5.el8.x86_64.rpm
rpm -ivh oracleasmlib-3.1.1-1.el8.x86_64.rpm
```

Una vez instalados los paquetes anteriores hay que configurar el **ASM**. Lo primero que hago es ver los discos que tengo asignados en la VM.
```sh
[root@localhost tmp]# lsblk
NAME        MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda           8:0    0   50G  0 disk 
|-sda1        8:1    0  600M  0 part /boot/efi
|-sda2        8:2    0    1G  0 part /boot
`-sda3        8:3    0 48.4G  0 part 
  |-ol-root 252:0    0 44.5G  0 lvm  /
  `-ol-swap 252:1    0  3.9G  0 lvm  [SWAP]
sdb           8:16   0   20G  0 disk <- Este es uno
sdc           8:32   0   20G  0 disk <- Este es uno
sdd           8:48   0   20G  0 disk <- Este es uno
sr0          11:0    1 1024M  0 rom 
```

Se configura con los siguientes comandos: 
```sh
oracleasm configure -i
oracleasm configure
```
>⚠️ Introducir grid como owner y asmadmin como group.

Habilitar el servicio de ASM y escanear los discos:
```sh
systemctl enable --now oracleasm
oracleasm scandisks
```

Antes de crear los discos para ASM es necesario crear las particiones. *Opciones n,p,1,enter,enter,w*
```sh
fdisk /dev/sdb
fdisk /dev/sdc
fdisk /dev/sdd
```

Para crear los tres discos destinados a ASM ejecuto los siguientes comandos:
```sh
oracleasm createdisk asmdisk1 /dev/sdb1
oracleasm createdisk asmdisk2 /dev/sdc1
oracleasm createdisk asmdisk3 /dev/sdd1
```

Una vez que se han creado los discos solo queda comprobar listando desde ASM los discos que ve:
```sh
[root@localhost tmp]# oracleasm listdisks
ASMDISK1
ASMDISK2
ASMDISK3
```

## GRID
---
La instalación de GRID es necesaria para poder tener Oracle Restart en la instalación de Oracle Database Single Instance.

Lo primero es descargar el software de GRID desde el siguiente enlace:
>https://www.oracle.com/database/technologies/oracle19c-linux-downloads.html

Los paquetes no  he mencionado antes, pero se pueden descargar desde la máquina destino o bajarlos en local y subirlos mediante `scp`.

## Instalación GRID

Antes de comenzar con la instalación es necesario preparar el entorno.

Conectado con el usuario GRID, creo un profile con las siguientes variables:
```sh
export ORACLE_SID=+ASM
export ORACLE_BASE=/u01/app/grid
export ORACLE_HOME=/u01/app/19.3.0/grid
export PATH=$ORACLE_HOME/bin:$PATH
export LD_LIBRARY_PATH=$ORACLE_HOME/lib:/lib:/usr/lib
export CLASSPATH=$ORACLE_HOME/JRE:$ORACLE_HOME/jlib:$ORACLE_HOME/rdbms/jlib
```

Cargo el profile y descomprimo el software en el ORACLE_HOME:
```sh
. .profile
cd software
unzip LINUX.X64_193000_grid_home.zip -d /u01/app/19.3.0/grid/
```

Oracle Linux 8.10 es lo que tenemos y según la documentación, Oracle Linux 8.8 va con GRID 19.23c. Lo ideal sería hacer la instalación aplicando la versión correspondiente, pero eso lo dejaré para otro post. Para poder prevenir el fallo de versión antes de la instalación, se puede ejecutar lo siguiente:

>💡 export CV_ASSUME_DISTID=OEL8.8

Revisar y/o modificar los permisos correspondientes en el directorio de inventario que pasaremos más adelante como parámetro:
```sh
chown grid:oinstall /u01/app/oraInventory/
```

Ahora viene la parte interesante, donde comenzamos realmente con la instalación. Tener en cuenta el parámetro **oracle.install.option=HA_CONFIG** ya que este es el que indica que sea Oracle Restart.

Yo tengo la manía o mala costumbre de hacer todo por consola, soy de la vieja escuela. Es por eso que siempre que pueda os enseñaré a instalar todo sin las *X Windows*. Para indicar al gridSetup que vamos mediante consola se utiliza el parámetro **-silent**:
```sh
./gridSetup.sh -silent \
INVENTORY_LOCATION=/u01/app/oraInventory \
SELECTED_LANGUAGES=en \
ORACLE_BASE=$ORACLE_BASE \
ORACLE_HOME=$ORACLE_HOME \
oracle.install.option=HA_CONFIG \
oracle.install.asm.OSDBA=asmdba \
oracle.install.asm.OSOPER=asmoper \
oracle.install.asm.OSASM=asmadmin \
oracle.install.asm.diskGroup.name=DATA \
oracle.install.asm.diskGroup.redundancy=EXTERNAL \
oracle.install.asm.diskGroup.diskDiscoveryString=/dev/sd* \
oracle.install.asm.diskGroup.disks=ORCL:ASMDISK1,ORCL:ASMDISK2 \
oracle.install.asm.SYSASMPassword=oracle \
oracle.install.asm.monitorPassword=oracle
```

Una vez que finaliza la instalación ejecutar los siguientes comandos como **root** :
```sh
/u01/app/oraInventory/orainstRoot.sh
/u01/app/19.0.0/grid/root.sh
```

Antes de finalizar la configuración hay que ejecutar un comando con el usuario grid: 

>Nota: Antes de lanzarlo completar las password dentro del fichero rsp que aparece.

```sh
/u01/app/19.3.0/grid/gridSetup.sh -silent -executeConfigTools -responseFile /u01/app/19.3.0/grid/install/response/grid_2026-04-21_08-34-35AM.rsp
```

---
Ya tendríamos el GRID con ASM instalado. Ahora llega la parte más divertida (comprobar que todo ha quedado correctamente).

Con el usuario grid se puede hacer una prueba de acceso a la base de datos de ASM:
```sql
[grid@localhost diag]$ sqlplus / as sysasm

SQL*Plus: Release 19.0.0.0.0 - Production on Tue Apr 21 08:52:20 2026
Version 19.3.0.0.0

Copyright (c) 1982, 2019, Oracle.  All rights reserved.


Connected to:
Oracle Database 19c Enterprise Edition Release 19.0.0.0.0 - Production
Version 19.3.0.0.0

SQL> 
```

Creo el diskgroup FRA ya que este no se ha creado durante la instalación utilizando el siguiente comando:
```sql
CREATE DISKGROUP FRA EXTERNAL REDUNDANCY DISK 'ORCL:ASMDISK3' ATTRIBUTE 'compatible.asm' = '19.0';
```

Este es uno de los primeros post que hago a modo **runbook**. Si encontráis algún error podéis escribirme por linkedin.

Os espero en el siguiente post.

---

*Las opiniones y contenidos de este blog son míos y no representan a Oracle.*


