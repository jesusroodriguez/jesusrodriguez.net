---
title: MAA Series 01 - Fundamentos de arquitectura Bronce
date: 2026-04-05
draft: false
tags:
  - MAA
---
## Introducción
Es la arquitectura de entrada para la alta disponibilidad en cuanto a bases de datos Oracle. 

Tiene el servicio básico de base de datos y el menor coste posible.

Se suelte utilizar en entornos de pruebas, desarrollo y entornos no críticos.

Para mi es la base de todo lo demás. Hay que entender como funciona toda la tecnología que hay disponible en este nivel para poder utilizarla de la mejor manera.

Cuando se habla de alta disponibilidad hay dos conceptos que se deben de tener en cuenta. 

**El RTO y el RPO.**

- El **RTO** mide el tiempo máximo aceptable de inactividad para restaurar.
- El **RPO** mide la cantidad máxima de datos (en tiempo) que se puede permitir perder.

Teniendo estos conceptos claros, iré haciendo un análisis de los mismos en las distintas arquitecturas. 

Añado la tabla para la arquitectura de referencia de bronce.

| **Interrupción no planificada**              | **Objetivos de nivel de servicio de RTO/RPO**                                                              |
| -------------------------------------------- | ---------------------------------------------------------------------------------------------------------- |
| Fallo recuperable de un nodo o instancia     | RTO: Minutos a horas  <br>RPO: 0                                                                           |
| Desastre: corrupciones y fallos del sitio    | RTO: Horas a días  <br>RPO: desde la última copia de seguridad o casi cero con dispositivo de recuperación |
| **Mantenimiento planificado**                |                                                                                                            |
| Actualizaciones de software/hardware         | RTO: Minutos a horas  <br>RPO: 0                                                                           |
| Actualización importante de la base de datos | RTO: Minutos a horas  <br>RPO: 0                                                                           |

El motor para la base de datos que se utiliza es Oracle Database Enterprise Edition con configuración de instancia única.  

Estas son algunas de las características necesarias para cumplir con MAA Bronce.

- **Oracle Recovery Manager (RMAN)**: Se utiliza para realizar copias de seguridad. Se validan los datos en la copia y en la restauración. Se recomienda tener una copia en local y otra en remoto
- **Automatic Storage Managment (ASM)**: Se utiliza como gestor de volúmenes integrado con Oracle. Es un sistema de archivos inteligente que protege contra fallos de disco y algunos tipos de corrupción.
- **Oracle Flashback Technologies**: Se utiliza para hacer recuperaciones lógicas con diferente granularidad.
- **Oracle Restart**: Se utiliza para reiniciar la base de datos, el listener y otros componentes Oracle después de un fallo hardware o software.
- **Oracle Corruption Protection**: Se utiliza para comprobar si existen daños físicos o lógicos en los bloques.

Opcionalmente se pueden utilizar las siguientes características:

- **Zero Data Loss Recovery Appliance (ZDLRA)**: Se utiliza como solución para copias de seguridad de las bases de datos. Reduce la ventana de copia así como el RPO a cerca de 0.
- **Online Maintenance**: Se utiliza para hacer redefiniciones y reorganizaciones del mantenimiento de la base de datos de manera online.
- **Oracle Multitenant y Resource Manager**: son la mejor práctica para la consolidación de base de datos y virtualización desde Oracle Database 12c. Las Pluggable Databases (PDB) habilitan la alta disponibilidad en casos de realocaciones, migraciones, failover y upgrades. El Resource Manager previene a las bases de datos de un alto consumo de recursos los cuales pueden generar incidencias de disponibilidad. El Resource Manager puede controlar CPU, memoria, procesos de sistema. El I/O y la red si está en Exadata también.

![Bronze architecture MAA](/images/maa_bronze_architecture.png)

---
*PD: Me gustaría dejar claro que todo lo que publico en este blog es mi propia opinión, nada de lo que comparto sustituye la documentación oficial.*

Enlaces de interés:
https://blogs.oracle.com/maa/](https://blogs.oracle.com/maa/)
