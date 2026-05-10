---
title: Conectar Claude con MCP Sqlcl en RDBMS 19c
date: 2026-05-10
draft: false
tags:
  - ia
  - sqlcl
  - mcp
---

---
He querido hacer una prueba de concepto para intentar conectar un rdbms 19c con claude y poder lanzarle preguntas en lenguaje humano.

Estas pruebas han sido realizadas utilizando la versión 1.6608 de claude, oracle rdbms 19.19, openjdk 25.0.2 y sqlcl 26.1.

## Prerrequisitos

Yo he realizado las pruebas en un macOS por lo que como gestor de paquetes utilizo brew. Para instalarlo se puede hacer con el siguiente comando.

```
bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

Otro prerrequisito es tener una base de datos a la que conectar. Como no quiero mezclar esto de momento con mi homelab de oracle he decidido utilizar otra base de datos. 

Para ahorrarme tiempo creando y configurando nada, como solamente necesito la conexión voy a utilizar podman. 
La instalación de podman con brew se hace de la siguiente manera:
Install podman in our environment.

```bash
brew install podman
```

Otro de los requisitos es tener instalado openjdk para poder posteriormente instalar sqlcl
```bash
brew install openjdk
```

Una vez que se ha instalado openjdk se debe exportar el path nuevo.
```bash
echo 'export PATH="/opt/homebrew/opt/openjdk@21/bin:$PATH"' >> ~/.zshrc
source ~/.zshrc
```
## Paso 1: RDBMS 19c en podman

El primer paso es tener la base de datos lista. Al conectarse al registry oficial de oracle se necesita hacer login,
Desde la web una vez que se ha hecho login se puede conseguir el token de autenticación que se necesitará para acceder desde la consola. 

```bash
podman login container-registry.oracle.com
```

Antes de seguir con la descarga de la imagen hay que aceptar los términos y condiciones desde la web en el producto que se desea obtener.

Crear el contenedor con podman para un oracle rdbms 19c.
```bash
podman run -d --name oracle-db -p 1521:1521 -e ORACLE_PWD=oracle container-registry.oracle.com/database/enterprise_ru:19.19.0.0
```

> Nota: Ha tardado unos 26 minutos en completar el comando anterior.

## Paso 2: Instalar sqlcl

Descargar sqlcl desde la web oficial de oracle y hacer el export del path:
```bash
curl -o sqlcl-latest.zip https://download.oracle.com/otn_software/java/sqldeveloper/sqlcl-latest.zip
unzip sqlcl-latest.zip
echo 'export PATH="$HOME/sqlcl/bin:$PATH"' >> ~/.zshrc
source ~/.zshrc
```

Ahora se puede hacer una prueba de conexión contra la base de datos que se ha desplegado en el paso 1:
```bash
sql system/oracle@//localhost:1521/ORCLPDB1
```

Una vez que se ha comprobado la conexión, hay que guardarla para poder utilizarla posteriormente desde claude:
```bash
conn -save oracle19c -savepwd system/oracle@//localhost:1521/ORCLPDB1
```

## Paso 3: Instalar y configurar claude

Para instalar claude yo lo he hecho desde brew con el siguiente comando:
```bash
brew install claude
```

Lo siguiente es modificar el fichero de configuración de claude y añadir el mcp server:
```bash
vi ~/Library/Application\ Support/Claude/claude_desktop_config.json
```

Este es el texto que hay que añadir, modificar con la ruta donde se tenga el binario de sql:
```txt
{
  "mcpServers": {
    "sqlcl": {
      "command": "/Users/jesus/sqlcl/bin/sql",
      "args": ["-mcp"]
    }
  }
}
```

Reiniciar claude.

## Paso 4: Preguntas a claude
Ha llegado el momento de hacer las pruebas.

Antes de cacharrear, yo recomiendo pedir que nos liste las conexiones que tiene en sqlcl y que se conecte a la que se le indique utilizando sqlcl:run-sql. 

A continuación añado dos imágenes probando el potencial de conectar claude con un oracle rdbms.

![conexion](/images/20260510-conexion.png)

![graficos](/images/20260510-graficos.png)

## Conclusión

Con esta prueba de concepto he conseguido conectar una base de datos oracle versión 19.19 con claude para poder hacer preguntas sobre performance, seguridad y cualquier cosa que se me ocurra si necesidad de tener que estar utilizando querys con select.

Esto abre un gran abanico para poder seguir investigando por esta línea.

Como recomendación recomiendo crear un usuario para esta tarea con permisos solo de lectura o configurar en claude para que pida permiso antes de ejecutar cada query.

---

*Las opiniones y contenidos de este blog son míos y no representan a Oracle.*


