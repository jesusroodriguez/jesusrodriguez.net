---
title: "MAA Series - 0"
date: 2026-03-25
draft: false
tags: ["MAA"]
---

En Oracle existen unas arquitecturas de referencia que se conoce como **Maximum Availability Architecture (MAA)**. 

Con las configuraciones y recomendaciones que se proporcionan en las diferentes arquitecturas,  Oracle proporciona los niveles de **Alta Disponibilidad** necesarios para el cumplimiento del servicio según el nivel deseado.

Dentro de las arquitecturas de referencia tendremos diferentes niveles para cubrir  alta disponibilidad, protección de datos y recuperación ante desastres.  

Las arquitecturas de referencia son las siguientes: Bronze, Silver, Gold y Platinum.

### Niveles MAA
A continuación muestro una tabla con las características para cada nivel de referencia:

| Nivel       | Contexto                        | Incluye | Características                                                             |
| ----------- | ------------------------------- | ------- | --------------------------------------------------------------------------- |
| 🟠 BRONCE   | Desarrollo, pruebas, producción | —       | Instancia única con Restart, Mantenimiento online, Validar Backup/restore   |
| ⚪ PLATA     | Producción, departamental       | BRONCE+ | Base de datos HA, Clúster Activo/Activo, Continuidad de aplicación          |
| 🟡 ORO      | Negocio Crítico                 | PLATA+  | Replicación Física, Protección de datos exhaustiva                          |
| 🔵 PLATINO  | Misión Crítica                  | ORO+    | Replicación Activa/Activa Lógica, Opciones avanzadas de HA                  |
| 💎 DIAMANTE | Disponibilidad Extrema          | —       | Goldengate 26AI replicas, Base de datos 26AI, RAC Exadata, Dataguard Activo |


En la anterior tabla se puede ver el nivel Diamante, no lo he incluido en la lista anterior ya que es reciente debido a la versión 26AI, necesaria para conseguir este nivel.

### Propósito de la serie
La idea de esta serie es montar una arquitectura MAA desde cero mientras documento las pruebas, errores y aprendizajes.  Como subir de nivel

Esta serie la voy a hacer utilizando un servidor propio que daré más información en el siguiente post.

Voy a escribir las mejores prácticas para conseguir los distintos tipos de arquitectura, como ir montando todo paso a paso, explicaciones de herramientas y lo que se me vaya ocurriendo por el camino.

Algunas de las tecnologías de las que hablaré en esta serie: ASM, GRID, RAC, Dataguard, Goldengate, Oracle database, RMAN, OEM, Flashback...

---
*PD: Me gustaría dejar claro que todo lo que publico en este blog es mi propia opinión, nada de lo que comparto sustituye la documentación oficial.*

Enlaces de interés: [https://blogs.oracle.com/maa/](https://blogs.oracle.com/maa/)
