---
title: Como he montado este blog con HUGO, Github y Cloudflare pages
date: 2026-04-19
draft: false
tags:
  - blog
  - cloudflare
  - git
---
# Como subo este blog

Hace unos meses no sabía como tener una pagina web autohosteada. Montar este blog fue un reto que conseguí autohostear en un servidor en mi casa.

Solo necesitaba estas cosas:
- Conexión a internet.
- Una página estática.
- Un dominio.
- Un hosting.

Como en mi casa no tengo alta disponibilidad y dependo de  más cosas como actualizaciones, pruebas, etc la evolución que he hecho ahora es utilizar cloudflare pages para alojar la web.

Ahora los pasos que sigo son los siguientes:
1. En local creo el nuevo post utilizando HUGO.
2. Hago el commit a mi github. https://github.com/jesusroodriguez
3. Cloudflare pages recoge los cambios en github y los publica en mi web.

## HUGO

HUGO es un generador de código estático escrito en GO. Tiene una comunidad increíble y existen bastantes temas para personalizarlo.

### Instalación
Yo lo tengo instalado en mi macbook personal. Para instalarlo lo he hecho con brew:
```sh
brew install hugo
```

### Crear el sitio
```sh
hugo new site jesusrodriguez.net
cd jesusrodriguez.net
git init
```

### Añadir tema
Hugo necesita un tema. El que utilizo yo es PaperMod. 

Se clona como submódulo de git:
```sh
git submodule add https://github.com/adityatelange/hugo-PaperMod.git themes/PaperMod
```

Una vez puesto el tema se tiene que añadir en el fichero de configuración de HUGO `hugo.toml`:

```toml
baseURL = "https://jesusrodriguez.net"
title = "Jesús Rodríguez"
theme = "PaperMod"
```

### Crear contenido
Para crear contenido yo utilizo vim. Añado un ejemplo de lo que sería el comienzo de este post:
```md
---
title: Como subo este blog
date: 2026-04-19
draft: false
tags:
  - blog
  - cloudflare
  - git
---

# Como subo este blog
```

### Pruebas de visualización
Una vez que tengo el post hecho, antes de pushearlo a github, lo visualizo en local arrancando el servidor de hugo
```sh
hugo server -D
```

Ahora desde el navegador puedo acceder a localhost:1313 y visualizar el blog.
## Github
Github es el control de versiones. Aquí es donde voy a tener varios repositorios entre ellos el de este blog.

## Instalación
Yo lo he instalado utilizando brew
```sh
brew install gh git
```

### Configuración
Lo primero que hay que hacer es generar la configuración para acceder a github.
```sh
gh auth login
```

Una vez que he seguido los pasos anteriores, creo el repositorio para el blog.
```sh
gh repo create jesusrodriguez.net --public --source=. --remote=origin --push
```

La carpeta de public no es necesario subirla a github, ya que cloudflare la genera automáticamente.
## Cloudflare pages
Cloudflare Pages sirve contenido estático con un CDN global. 

Cloudflare ofrece el uso de cloudflare pages de manera gratuita. Además, tiene integración con github.

Lo primero que hay que hacer es conectar GitHub a Cloudflare

1. En el apartado Workers & Pages → Pages → Create a project → Connect to Git
2. Seleccionar el repositorio de GitHub (`jesusroodriguez/jesusrodriguez.net`)
3. Cloudflare pide autorizar GitHub se acepta y listo.

Configurar el build de HUGO:

```
Framework preset: Hugo
Build command: hugo
Build output directory: public
Environment variable: HUGO_VERSION = 0.160.1
```

Ahora falta conectar este proyecto con el dominio:

En Cloudflare Pages → el proyecto → Custom domains → Add domain

Al tener el dominio en Cloudflare, se configura automáticamente. 


**El flujo que sigo para crear un nuevo post**

Con todo esto configurado, lo que tengo que hacer ahora es:

```sh
hugo new content/posts/como-subo-el-blog
vim content/posts/como-subo-el-blog
git add .
git commit -m "post: nuevo post como subo el blog"
git push origin main
```

Cloudflare detecta el push, construye Hugo y lo publica en unos 30 segundos. 

Ya solo falta entrar en la web y ver el nuevo post.

---

*Las opiniones y contenidos de este blog son míos y no representan a Oracle.*
