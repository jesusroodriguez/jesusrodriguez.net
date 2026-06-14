---
title: Oracle a fondo, qué plataforma de infraestructura elegir según tu caso de uso
date: 2026-06-14
draft: false
tags:
  - linux
  - virtualización
  - servers
---

---
## Oracle a fondo: qué plataforma de infraestructura elegir según tu caso de uso

![comparativa](/images/comparativa-plataformas-oracle.png)

El ecosistema de Oracle es más amplio de lo que parece, y hay que saber que infraestructura necesitamos para cada caso de uso. 

¿Me hace falta virtualbox? O mejor, ¿Debería tener un exadata?

En este post voy a dar mi punto de vista respecto a las diferentes plataformas, teniendo en cuenta sus ventajas, desventajas y casos de uso reales. 

Esto puede ayudarnos a elegir mejor lo que necesitamos realmente.

---

### 1. Oracle VirtualBox

#### ¿Qué es?

VirtualBox es un hipervisor de tipo 2 (hosted) de código abierto y uso gratuito, que se ejecuta sobre un sistema operativo anfitrión (Windows, Linux, macOS). Permite crear y gestionar máquinas virtuales en un entorno de escritorio o portátil.

#### Ventajas

- Gratuito y open source.
- Multiplataforma: Windows, Linux, macOS.
- Ideal para aprendizaje, desarrollo y pruebas locales.
- Fácil de instalar y usar, con interfaz gráfica intuitiva.
- Soporte de snapshots y clonado de VMs.

#### Desventajas

- No recomendable para producción ni cargas críticas.
- Rendimiento limitado por ser hipervisor tipo 2.
- Escalabilidad prácticamente nula.

#### Casos de uso

- Entornos de desarrollo y pruebas en el portátil.
- Formación y certificaciones ppor ejemplo las de Oracle (OCA, OCP...).
- Laboratorios de aprendizaje de bases de datos Oracle, Linux, etc.
- Pruebas de compatibilidad de software antes de despliegues.
- Probar sistemas operativos sin necesidad de baremetal nuevo.

---

### 2. Oracle VM Server (OVM)

#### ¿Qué es?

Oracle VM Server es un hipervisor de tipo 1 (bare-metal) basado en Xen, diseñado para entornos empresariales on-premise. Incluye Oracle VM Manager para gestión centralizada de pools de servidores y almacenamiento compartido. 

#### Ventajas

- Hipervisor bare-metal: mejor rendimiento que VirtualBox.
- Soporte certificado para cargas Oracle (RDBMS, RAC, WebLogic...)
- Integración con Oracle Linux y Oracle Engineered Systems.
- Oracle VM Templates: despliegue rápido de pilas Oracle.

#### Desventajas

- Tecnología sin soporte desde Junio de 2024: Oracle ha apostado por KVM (OLVM).
- Curva de aprendizaje en gestión del clúster Xen.
- Menor ecosistema de herramientas de terceros que VMware vSphere.

#### Casos de uso

- Consolidación de servidores Oracle on-premise en entornos ya establecidos con OVM.
- Virtualización de bases de datos Oracle y middleware en centros de datos propios.
- Organizaciones que aprovechan el licenciamiento "hard partitioning" de Oracle.
- Para mi esto sería un primer paso para empezar a montar un homelab.

---

### 3. Oracle Linux Virtualization Manager (OLVM)

#### ¿Qué es?

OLVM es la evolución de OVM, basada en KVM (Kernel-based Virtual Machine) y oVirt. Es la apuesta actual de Oracle para la virtualización empresarial on-premise, ofreciendo una plataforma moderna y con soporte completo de Oracle.

Con esta tecnología monté yo mi laboratorio para hacer pruebas con las VMs de Oracle.

#### Ventajas

- Hipervisor KVM: estándar de la industria, ampliamente soportado.
- Sustituto natural de OVM con mayor comunidad y ecosistema.
- Integrado con Oracle Linux y su UEK (Unbreakable Enterprise Kernel).
- Sin coste de licencia adicional si ya tienes Oracle Linux Premier Support.
- Compatible con hard partitioning para licenciamiento Oracle DB.

#### Desventajas

- Menos maduro que VMware vSphere.
- La migración desde OVM o VMware requiere planificación.
- Documentación y casos de referencia en producción aún limitados respecto a competidores.

#### Casos de uso

- Modernización de entornos OVM existentes hacia KVM.
- Nuevos proyectos de virtualización on-premise en entornos Oracle-céntricos.
- Organizaciones que quieren evitar el coste de licencias VMware/vSphere.
- Plataforma de virtualización para cargas Oracle con necesidad de hard partitioning.

---

### 4. Oracle Database Appliance (ODA)

#### ¿Qué es?

ODA es un appliance convergente (hardware + software + licencias preintegradas) diseñado específicamente para ejecutar Oracle Database. Incluye Oracle Linux, OLVM o bare-metal, almacenamiento NVMe y red de alta velocidad, todo gestionado con un único CLI/GUI simplificado.

#### Ventajas

- Despliegue muy rápido: hardware y software validado y preconfigurado.
- Gestión simplificada con `odacli`: parcheo, backups, provisionado en un solo comando.
- Coste predecible: modelo de licenciamiento incluido (capacity-on-demand).
- Alta disponibilidad integrada: clúster de 2 nodos con Oracle RAC o Data Guard.
- Ideal para entidades sin grandes equipos de infraestructura.

#### Desventajas

- Limitado a cargas de trabajo Oracle Database (no es un virtualizador de propósito general).
- Hardware propietario: menor flexibilidad en elección de componentes.
- Coste inicial elevado frente a infraestructura genérica.
- Escalabilidad vertical limitada al modelo adquirido.

#### Casos de uso

- Pequeñas y medianas empresas que necesitan Oracle DB con alta disponibilidad.
- Proyectos que requieren Oracle RAC sin la complejidad de montarlo en infraestructura propia.
- Renovaciones de hardware donde se quiere consolidar DB + almacenamiento + HA en un único sistema.

---

### 5. Oracle Exadata

#### ¿Qué es?

Exadata es la plataforma de ingeniería más avanzada de Oracle para bases de datos. Es el buque insignia. Combina servidores de cómputo, celdas de almacenamiento inteligente (Smart Scan, Storage Indexes, HCC), red InfiniBand/RDMA y software optimizado de Oracle para lograr el máximo rendimiento en OLTP, Data Warehouse y cargas mixtas. Disponible on-premise (Exadata Database Machine) y como servicio gestionado en OCI (Exadata Cloud Service) o en el cliente (Exadata Cloud@Customer).

#### Ventajas

- Rendimiento extremo: Smart Scan descarga procesamiento a las celdas de almacenamiento.
- Compresión Hybrid Columnar (HCC): reducciones de almacenamiento de 10x-50x.
- IOPS y throughput muy superiores a infraestructura convencional.
- Consolidación masiva: múltiples bases de datos en un único sistema con aislamiento.
- Parcheo y mantenimiento con zero downtime (rolling patches).
- Soporte Oracle Premier de máximo nivel.

#### Desventajas

- Coste elevado (inversión inicial + licencias + soporte).
- Exclusivo para Oracle Database: no virtualiza otras cargas.
- Requiere expertise especializado para su gestión óptima.

#### Casos de uso

- Grandes corporaciones con bases de datos Oracle misión crítica.
- Data Warehouses y entornos analíticos con volúmenes de terabytes/petabytes.
- Consolidación de decenas o cientos de bases de datos Oracle en una única plataforma.
- Entornos de banca, telco o con requisitos extremos de rendimiento y disponibilidad.
- Migración a la nube progresiva vía Exadata Cloud@Customer (cloud en el datacenter propio). Esto permite tener los firewall del cliente y todo "bajo su paraguas".

---

### 6. Oracle Cloud Infrastructure (OCI)

#### ¿Qué es?

OCI es la nube pública de Oracle, con una arquitectura de red de alta velocidad (25/100 Gbps), almacenamiento NVMe local y bare metal, y una propuesta de valor centrada en cargas empresariales críticas, especialmente Oracle Database. Ofrece IaaS, PaaS y SaaS, incluyendo Autonomous Database, Kubernetes (OKE), Object Storage, AI/ML services y mucho más.

#### Ventajas

- Precios competitivos, especialmente en cómputo y transferencia de datos.
- Autonomous Database: base de datos autogestionada con auto-tuning, auto-patching y auto-scaling.
- Rendimiento de red superior: arquitectura off-box que no consume CPU del tenant.
- Integración nativa con todo el stack Oracle (E-Business Suite, JD Edwards, PeopleSoft, Fusion).
- Free Tier generoso para pruebas y desarrollo.
- Multicloud: Interconexión certificada con Azure, Google Cloud y Amazon Web Services.

#### Desventajas

- La propuesta de valor es más fuerte para cargas Oracle, para workloads heterogéneos puede ser menos competitivo.

#### Casos de uso

- Migración de bases de datos Oracle a la nube sin cambiar licencias (BYOL).
- Autonomous Database para nuevas aplicaciones que necesitan base de datos autogestionada.
- Entornos de desarrollo, CI/CD y contenedores con Kubernetes (OKE).
- Estrategias multicloud. Utilizar bases de datos Oracle con el potencial de infraestructura Oracle en otras nubes.

---


### Conclusión

No existe una única respuesta correcta: la elección depende del tamaño de tu organización, el nivel de criticidad de tus cargas, tu estrategia cloud y, sobre todo, tu presupuesto.

La buena noticia es que Oracle ofrece un camino coherente: puedes empezar aprendiendo con VirtualBox, consolidar en producción con OLVM u ODA, escalar con Exadata y evolucionar hacia OCI, manteniendo las mismas herramientas, licencias y conocimiento en cada etapa del camino.

Espero que esta comparación sea de utilidad. 

Cualquier duda podéis contactarme a través de linkedin.

---

*_Esto es un blog personal con carácter informativo. Puede contener errores.*


