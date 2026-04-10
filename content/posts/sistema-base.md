---
title: "Homelab para Oracle MAA: miniPC Ryzen 7 con OLVM y KVM"
date: 2026-03-29
draft: false
tags: ["Oracle Linux", "OLVM", "KVM"]
---
## Presentación
Como centro de operaciones para montar los laboratorios de Oracle Maximum Availability Architecture (MAA) voy a utilizar un minipc con ryzen 7 5800h y 32GB de RAM.

A continuación presento con una imagen al afortunado que va a ir conmigo de la mano en este viaje.

![minipc](/images/minipc_beelink_ryzen5800h.png)

Una vez que ya tengo elegido el bare metal, necesito sistema operativo, virtualizador y algunas cosas más.

Para este viaje, he decidido montar todo lo que pueda con tecnología Oracle aprovechando que voy a ir aprendiendo por el camino.

Como sistema operativo elijo Oracle Linux 8.10 UEK7. Según la documentación oficial, se recomienda el kernel UEK. 

Enlace descargar la ISO:
https://yum.oracle.com/ISOS/OracleLinux/OL8/u10/x86_64/OracleLinux-R8-U10-x86_64-boot-uek.iso

Voy a instalar una versión con margen de crecimiento. En un futuro haré upgrade de sistema operativo para aprender cosas nuevas.

Para tener diferentes VMs y administrar todo de una manera visual he decidido montar *KVM* y *OLVM*.

Con KVM tengo el hypervisor. La posibilidad de generar máquinas virtuales en el Oracle Linux anfitrión.

Con OLVM tengo la administración en un interfaz web que me permite gestionar las distintas VMs.

La arquitectura quedaría de la siguiente manera:

```bash
Oracle Linux 8.10 (bare metal)
  └── KVM
      └── VM: olvm-manager
      └── VMs de laboratorios 
```
## Oracle Linux

Hay mucha documentación al respecto, pero a modo resumen os voy a contar lo que yo he hecho o tenido en cuenta.

Preparo el USB con la ISO usando el comando `dd`.
```bash
sudo dd if=./OracleLinux-R8-U10-x86_64-boot-uek.iso of=/dev/sda bs=4M status=progress oflag=sync
```

Se recomienda realizar la instalación con la interfaz gráfica, si se decide hacer en modo texto se utiliza anaconda.

Las particiones según la documentación deberían de ser como en la siguiente tabla:

| Partición | Tipo | Tamaño                                                                 |
| --------- | ---- | ---------------------------------------------------------------------- |
| /boot     | XFS  | 1 Gb                                                                   |
| /boot/EFI | VFAT | 600 Mb                                                                 |
| /swap     | XFS  | Entre 8 y 64 de RAM, el swap es la mitad de la memoria.                |
| /         | XFS  | 70 Gb o el 50% del disco sobrante tras configurar el /boot y el /swap. |
| /home     | XFS  | El resto de disco restante.                                            |

A tener en cuenta que  /swap, / y /home deberían ir en LVM.

Una vez instalado actualizo los paquetes con `sudo dnf update -y`.

Reviso que se ha instalado con kernel UEK
```bash
[root@localhost ~]# uname -a
Linux localhost.localdomain 5.15.0-318.199.3.2.el8uek.x86_64 #2 SMP Mon Mar 9 08:37:04 PDT 2026 x86_64 x86_64 x86_64 GNU/Linux
```

Antes de seguir, lo que hago es revisar que tengo el servicio de ssh configurado para acceder desde mi ordenador portátil y trabajar más cómodo:
```bash
[root@localhost ~]# systemctl status sshd
● sshd.service - OpenSSH server daemon
   Loaded: loaded (/usr/lib/systemd/system/sshd.service; enabled; vendor preset: enabled)
   Active: active (running) since Sun 2026-03-22 20:50:42 CET; 15min ago
     Docs: man:sshd(8)
           man:sshd_config(5)
 Main PID: 999 (sshd)
    Tasks: 1 (limit: 47920)
   Memory: 3.8M
   CGroup: /system.slice/sshd.service
           └─999 /usr/sbin/sshd -D -oCiphers=aes256-gcm@openssh.com,chacha20-poly1305@openssh.com,aes256-ctr,aes256-cbc,aes128-gcm@openssh.com,aes12>
```

Ya tengo el sistema base por fin.
## KVM
Para almacenar las VMs es necesario tener espacio de sobra en /var.

Antes de instalar, hay que tener habilitado en BIOS el Intel VT-x o el AMD-V.

Los paquetes a instalar recomendados:
```bash
sudo dnf install libvirt qemu-kvm qemu-img virt-install virt-viewer
```

Arranco el servicio:
```bash
sudo systemctl start libvirtd
sudo systemctl enable libvirtd
sudo systemctl status libvirtd
```

Con el ryzen 7 5800h (8 cores físicos) → máximo recomendado **16 vCPUs** en total entre todas las VMs.

Ahora voy a crear una VM con OEL8.10 UEK7 donde voy a configurar el OLVM.

Creo el disco para la nueva vm:
```bash
qemu-img create -f qcow2 /var/lib/libvirt/images/olvm-manager.qcow2 200G
```

Lanzo la creación de la vm:
```bash
virt-install \
  --name olvm-manager \
  --memory 8192 \
  --vcpus 2 \
  --os-variant ol8.0 \
  --disk /var/lib/libvirt/images/olvm-manager.qcow2,format=qcow2 \
  --location /var/isos/OracleLinux-R8-U10-x86_64-boot-uek.iso \
  --graphics none \
  --console pty,target_type=serial \
  --extra-args "console=ttyS0,115200n8 inst.text"
```

Al no utilizar intergaz gráfica se instala utilizando anaconda. 

Una vez acabada la instalación devuelve el prompt dentro de la vm. 

```bash
[root@oracle_server ~]# virsh list --all
setlocale: No such file or directory
 Id   Name           State
------------------------------
 3    olvm-manager   running
``` 
## OLVM
Me conecto a la VM que acabo de crear con
`virsh console olvm-manager`

Antes de comenzar la instsalación de OLVM preparo la máquina.
```bash
sudo hostnamectl set-hostname olvm-manager.homelab.local
echo "192.168.X.X olvm-manager.homelab.local olvm-manager" >> /etc/hosts
```

Instalo ovirt y algún que otro paquete que aparece en la documentación.
```bash
dnf install oracle-ovirt-release-45-el8 kernel-uek-modules-extra kernel-uek rsyslog
```

Hago una prueba con el script de prechequeo`olvm-pre-check.py`y veo todo en verde. Buena señal.

Instalo el engine y una vez que está instalado ejecuto el deploy.
```bash
sudo dnf install ovirt-engine -y
engine-setup
```

Una vez hecho el deploy consigo acceder desde el navegador del portátil al panel web.

![olvm](/images/OLVM_Manager_startScreen.png)

Y desde este panel web voy a crear y administrar el resto de VMs para montar los laboratorios de Maximum Availability Architecture y todo lo que tenga que ver con temas relacionados de Oracle.

## Problemas encontrados en este post
He tenido problemas para acceder con FQDN al panel web.  Estaba utilizando *pihole* y *nginx*. La solución ha sido utilizar solamente pihole para redirigir directamente a la máquina de OLVM.

---
*PD: Me gustaría dejar claro que todo lo que publico en este blog es mi propia opinión, nada de lo que comparto sustituye la documentación oficial.*

Enlaces de interés:
https://blogs.oracle.com/maa/](https://blogs.oracle.com/maa/)
https://docs.oracle.com/en/operating-systems/oracle-linux/8/relnotes8.10/
https://docs.oracle.com/en/virtualization/oracle-linux-virtualization-manager/
