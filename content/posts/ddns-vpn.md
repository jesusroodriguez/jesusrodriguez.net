---
title: Perdí la conexión a la VPN y así lo he resuelto
date: 2026-05-31
draft: false
tags:
  - redes
---

---
## He perdido la conexión a mi VPN
Para introducir este post, voy a contar la situación.

Hace unos meses monté una VPN con wireguard en mi servidor de proxmox, utilizando una nueva VM para dicha función.

La VM la monté con IP estática, etc. Esto está montado con un docker (por motivos de seguridad) no lo hice en la VM en la que tengo todos los dockers.

Bien, pues conseguí conectar mis portátiles, iphone... y yo tan contento.

Pues últimamente, hará un par de semanas, intentaba conectarme desde fuera de casa a la VPN y conectaba sin problemas, pero no era capaz de navegar ni nada en mi red local.

Me parecía extraño pero no le di importancia. Tras varias veces probando y llegando al mismo punto decidí investigar.

Tengo un router de la operadora DIGI, para los que no lo conozcáis DIGI utiliza CGNAT, es decir compartes tu IP pública con más usuarios. Pero existe una "Conexión plus" pagando 1€ más al mes y te sacan del CGNAT, esto es cojonudo para poder montar VPN, abrir puertos...

Si, yo pago religiosamente el € de más.

Mi sospresa es que cuando se reinicia el router, se va la luz, etc, el router coge otra ip pública.

Y yo tenía configurado mi wireguard con mi ip pública. 

MAL. FATAL.

Por eso no era capaz de navegar en mi red local.
## ¿Cómo lo he solucionado?
Lo primero que estuve probando fue cambiar la ip pública manualmente en el docker del wireguard. Después volví a configurar los clientes uno a uno para poder volver a navegar.

Hasta aquí solucionado.

Pero me parecía una mierda estar cada vez que el router se reinicie o lo que sea estar haciendo esto. Y más, si me pilla fuera de casa que pierdo la conexión a mi red local.

Estuve investigando y encontré DDNS. 

¿Qué consigo con DDNS? Consigo poner un dominio en vez de una IP y poder apuntar siempre al mismo dominio, pero la IP del dominio es la que varía.

Al tener dominio con cloudflare para publicar este blog, estuve investigando de montar el DDNS con ellos.

Lo que tuve que hacer es modificar el docker en vez de apuntar a la IP pública, que apuntara al dominio que iba a crear luego en cloudflare.

> ejemplo.jesusrodriguez.net

Desde cloudflare, utilizando mi dominio he configurado un nuevo DNS de tipo A, apuntando a la IP pública actual y al dominio ejemplo.jesusrodriguez.net.

Una vez que tenía esto preparado, modifiqué los clientes uno a uno para que apuntaran al dominio que he creado de tipo A.

Con esto ya era capaz de navegar desde fuera de mi casa en mi red local utilizando DDNS.

## La mejora real
Bien llegados a este punto, tenía lo mismo pero con más cosas añadidas para que funcionara es decir, me conectaba desde fuera a mi VPN y podía navegar.

Pero yo quería que esto fuera trasparente para mi, no depender de cambiar la IP de cloudflare etc.

Por lo que en la VM donde tengo el docker de wireguard, hice un script que conecta con la API de cloudflare.

Este script lo que hace es cada x tiempo comprobar la IP pública asignada en el DNS de tipo A que he creado en cloudflare y obtener la IP pública real. Si la IP es la misma no hace nada, si es diferente, pushea la nueva IP pública a cloudflare a través de la API.

Con esto he conseguido no tener que volver a preocuparme de perder la conexión fuera de casa a mi red local, salvo si se va la luz. Ya que el router y el minipc quedarían apagados.

Muy contento con el progreso de esta semana.

Os dejo por aquí el script para conectar con la API de cloudflare por si a alguien le sirve de ayuda:
```bash
#!/bin/bash

DNS_ZONE='<tu dns zone>'
DNS_RECORD='<tu dns record>'
AUTH_KEY='<tu api key>'
EMAIL_ADDRESS='<tu email>'
DNS_RECORD_NAME='<el dominio que has creado>'

CURRENT_IP_ADDRESS=$(curl -s ip.me)
CURRENT_DNS_VALUE=$(curl -sX GET "https://api.cloudflare.com/client/v4/zones/${DNS_ZONE}/dns_records/${DNS_RECORD}" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer ${AUTH_KEY}" | jq -r '.result.content') 

if [ "${CURRENT_DNS_VALUE}" != "${CURRENT_IP_ADDRESS}" ]; then
  echo "IP cambiada: ${CURRENT_DNS_VALUE} -> ${CURRENT_IP_ADDRESS}. Actualizando..."
  curl -sX PUT "https://api.cloudflare.com/client/v4/zones/${DNS_ZONE}/dns_records/${DNS_RECORD}" \
    -H "Content-Type: application/json" \
    -H "Authorization: Bearer ${AUTH_KEY}" \
    --data "{\"type\":\"A\",\"name\":\"${DNS_RECORD_NAME}\",\"content\":\"${CURRENT_IP_ADDRESS}\"}"
  echo "DNS actualizado correctamente a " ${CURRENT_IP_ADDRESS} >> ip-change.log
else
  echo "IP sin cambios: ${CURRENT_IP_ADDRESS}"
fi
```


---

*Esto es un blog personal con carácter informativo. Puede contener errores.*

