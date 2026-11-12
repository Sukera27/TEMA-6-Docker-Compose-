# Tema 6 · Docker en Azure — Container Registry, máquina virtual y docker-compose (front + API + MySQL)

Apuntes prácticos de la asignatura **Despliegue de Aplicaciones Web (DAW)**. Este documento recoge, en orden cronológico, todos los pasos y comandos de la sesión: se empaquetan en **imágenes de Docker** una API Spring Boot (**pokeAPI**) y su front en Astro, se suben a un **Azure Container Registry**, y en una **máquina virtual de Azure** se descargan y se arrancan juntas con **docker-compose**, conectadas a una base de datos **MySQL de Azure**.

Los apuntes en formato web (con la teoría, el esquema y las tablas) están en [`index.html`](./index.html).

## Contenido de esta carpeta

| Archivo / carpeta | Descripción |
|---|---|
| `index.html` | Apuntes del Tema 6 (arquitectura, recursos de Azure, registro de imágenes, máquina virtual, compose) |
| `README_6.md` | Este archivo: pasos y comandos en orden cronológico |
| `ficheros-azure/` | Ficheros de ejemplo: `back/Dockerfile`, `front/Dockerfile`, `docker-compose.yml` y `.env.example` |
| `img/` | Favicon usado por `index.html` |

## Requisitos previos

- Haber hecho el **Tema 5** (Dockerfile y docker-compose en local).
- **Docker Desktop** en tu ordenador, abierto y en marcha.
- Una **cuenta de Azure** con suscripción activa.
- Para compilar: **Java 17** (se usa `./mvnw`) y **Node.js 20 o superior** (para el front).
- Los proyectos **`pokeAPI`** (Spring Boot 4, puerto 8085) y **`pokeAPI-front`** (Astro con Node).

## La arquitectura

```
Navegador ──(puerto 80)──► Máquina virtual de Azure
                           ┌──────────────────────────────────────┐
                           │  docker-compose                      │
                           │  ┌───────────┐       ┌────────────┐  │
                           │  │ frontend  │──────►│  backend   │──┼──► Azure Database for MySQL
                           │  │ (Astro)   │ :8085 │ (Spring)   │  │      (en la nube)
                           │  └───────────┘       └────────────┘  │
                           └──────────────────────────────────────┘
                                      ▲
                                      │ docker pull
                           Azure Container Registry  ◄── docker push ── tu ordenador
```

- El **front** es el único que se publica al exterior (puerto 80). El **back** solo es visible para el front, por la red interna de Docker.
- La **base de datos no va en un contenedor**: es un servicio de Azure (*Azure Database for MySQL*), así que el compose solo tiene dos servicios.
- Las imágenes **no se construyen en la máquina virtual**: se construyen en tu ordenador, se suben al registro y la máquina solo las descarga.

> **Por qué el front llama al back con `http://backend:8085`:** el front usa renderizado en servidor (SSR). Quien llama al back es el servidor de Node que corre **dentro** del contenedor del front, no el navegador. Dentro de la red de Docker, cada servicio se encuentra por el nombre que tiene en el `docker-compose.yml`, y `backend` es el nombre del servicio. No hace falta publicar el puerto 8085 ni usar la IP pública.

---

# Práctica 1 · Preparar y construir las imágenes (en tu ordenador)

## 1. Carpeta de despliegue

Se crea una carpeta con una subcarpeta para cada imagen, cada una con su `Dockerfile`:

```
pokeapi-deploy/
├── back/
│   ├── Dockerfile
│   └── pokeAPI-0.0.1-SNAPSHOT.jar        ← el .jar compilado
└── front/
    ├── Dockerfile
    ├── package.json
    ├── package-lock.json
    └── dist/                              ← la build de Astro
```

Los `Dockerfile` están en `ficheros-azure/`.

## 2. La API: configuración por variables de entorno

Los datos de la base de datos **no se escriben dentro del `.jar`** (acabarían en el registro, a la vista de quien descargue la imagen). El `application.yaml` los lee de variables de entorno:

```yaml
spring:
  datasource:
    url: jdbc:mysql://TU_SERVIDOR.mysql.database.azure.com/pokemondb?useSSL=true&createDatabaseIfNotExist=true
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
```

Esas variables las pondrá el `docker-compose.yml` al arrancar el contenedor.

## 3. Compilar el .jar y copiarlo

```bash
cd pokeAPI
./mvnw clean package -DskipTests
cp target/pokeAPI-0.0.1-SNAPSHOT.jar ../pokeapi-deploy/back/
```

## 4. Compilar el front y copiar la build

```bash
cd pokeAPI-front
npm install
npm run build
cp -R dist ../pokeapi-deploy/front/
cp package.json package-lock.json ../pokeapi-deploy/front/
```

`npm run build` genera la carpeta `dist/` (con `client/` y `server/`). **No se copia `node_modules`**: se instala dentro de la imagen (`npm ci` en el `Dockerfile`), ya preparado para Linux.

## 5. Los dos Dockerfile

`back/Dockerfile`:

```dockerfile
FROM eclipse-temurin:17-jre

WORKDIR /app
COPY pokeAPI-0.0.1-SNAPSHOT.jar app.jar

EXPOSE 8085

ENTRYPOINT ["java", "-jar", "app.jar"]
```

`front/Dockerfile`:

```dockerfile
FROM node:22-alpine

WORKDIR /app
ENV NODE_ENV=production \
    HOST=0.0.0.0 \
    PORT=4321

COPY package.json package-lock.json ./
RUN npm ci --omit=dev

COPY dist ./dist

EXPOSE 4321
CMD ["node", "./dist/server/entry.mjs"]
```

| Instrucción nueva | Qué hace |
|---|---|
| `ENV` | Define variables de entorno que la imagen lleva siempre. `HOST=0.0.0.0` es obligatorio en un contenedor: sin él, el servidor solo escucha dentro de sí mismo y no se puede acceder desde fuera |
| `RUN` | Ejecuta un comando **al construir** la imagen (aquí, instalar las dependencias) |
| `CMD` | Comando que se ejecuta al arrancar el contenedor (parecido a `ENTRYPOINT`) |

> **Por qué se copia `package.json` antes que `dist`:** Docker guarda en caché cada instrucción. Si `package.json` no cambia, `npm ci` (lo más lento) no se repite; al cambiar solo el código, solo se rehace la copia de `dist`.

---

# Práctica 2 · Azure Container Registry

Un **registro** es un almacén de imágenes. Docker Hub es el público; **Azure Container Registry (ACR)** es el privado de Azure, donde solo entra quien tenga las credenciales.

## 6. Crear el registro

En el portal de Azure: **Registros de contenedor → Crear**.

- **Grupo de recursos**: el mismo donde estarán la máquina virtual y la base de datos.
- **Nombre del registro**: único en todo Azure, en minúsculas y sin guiones (por ejemplo `miregistro`). Su dirección será `miregistro.azurecr.io`.
- **SKU**: Basic.

## 7. Activar el usuario administrador

En el registro: **Configuración → Claves de acceso → Usuario administrador (activar)**. Aparecen el **nombre de usuario** y dos **contraseñas**; se usarán para iniciar sesión desde Docker.

## 8. Iniciar sesión desde Docker

```bash
docker login miregistro.azurecr.io
```

Pide el usuario y la contraseña de las claves de acceso. Debe responder `Login Succeeded`.

## 9. Construir las imágenes con el nombre del registro

El nombre de una imagen lleva **delante la dirección del registro**; así Docker sabe a dónde subirla:

```bash
cd pokeapi-deploy
docker build -t miregistro.azurecr.io/pokeapi-back:1.0  ./back
docker build -t miregistro.azurecr.io/pokeapi-front:1.0 ./front
docker images
```

> **Arquitectura (solo si tu ordenador es Apple Silicon o ARM):** las máquinas virtuales de Azure usan procesadores `amd64`. Si construyes en un Mac con chip M1/M2/M3, añade `--platform linux/amd64` al `docker build`; si no, la imagen se descargará pero no arrancará (`exec format error`). Se comprueba con:
> ```bash
> docker image inspect miregistro.azurecr.io/pokeapi-back:1.0 --format '{{.Architecture}}'
> ```
> Debe decir `amd64`.

## 10. Subir las imágenes

```bash
docker push miregistro.azurecr.io/pokeapi-back:1.0
docker push miregistro.azurecr.io/pokeapi-front:1.0
```

Se comprueba en el portal, en **Servicios → Repositorios** del registro: deben aparecer `pokeapi-back` y `pokeapi-front` con la etiqueta `1.0`.

---

# Práctica 3 · La máquina virtual

## 11. Crear la máquina

En el portal: **Máquinas virtuales → Crear**.

- **Imagen**: Ubuntu Server 24.04 LTS.
- **Tamaño**: con **2 GB de RAM como mínimo** (por ejemplo `B1ms`). Con 1 GB, Spring Boot y Docker se quedan sin memoria.
- **Autenticación**: clave pública SSH. Al terminar, Azure descarga un fichero `.pem`: **es la llave de entrada, hay que guardarla bien**.
- **Reglas de puertos de entrada**: marcar **SSH (22)** y **HTTP (80)**.

Cuando esté creada, en **Información general** aparece su **IP pública**. Conviene fijarla: **IP pública → Configuración → Asignación: Estática**. Si es dinámica, cambia cada vez que la máquina se detiene y se vuelve a iniciar.

## 12. Entrar por SSH

```bash
chmod 400 /ruta/a/clave.pem
ssh -i /ruta/a/clave.pem azureuser@IP_DE_LA_MAQUINA
```

Si no se hace el `chmod 400`, SSH rechaza la llave con `UNPROTECTED PRIVATE KEY FILE` (el fichero no puede ser legible para otros usuarios).

## 13. Instalar Docker y Compose

```bash
sudo apt update
sudo apt install -y docker.io docker-compose-v2
sudo usermod -aG docker $USER
```

Se cierra la sesión SSH y se vuelve a entrar, para poder usar `docker` sin `sudo`. Se comprueba:

```bash
docker --version
docker compose version
docker run hello-world
```

> **`docker-compose` frente a `docker compose`:** el paquete `docker-compose` de `apt` instala la **versión 1** (comando con guion), ya obsoleta, que además falla con versiones recientes de Docker (`KeyError: 'ContainerConfig'`). El paquete `docker-compose-v2` instala la actual, que se usa **sin guion**: `docker compose`. Si ya tienes la antigua, funciona igual, pero hay que usar `sudo docker-compose ...`.

## 14. Crear un espacio de intercambio (swap)

Es muy recomendable en máquinas con poca memoria: si se agota la RAM, el sistema se bloquea y deja de responder (incluso por SSH).

```bash
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
free -h
```

`free -h` debe mostrar 2 GB en la fila `Swap`.

---

# Práctica 4 · docker-compose en la máquina

## 15. La base de datos de Azure: permitir la conexión

La base de datos (*Azure Database for MySQL*) está en la nube y tiene su propio **firewall**: por defecto no deja entrar a nadie. En el servidor MySQL: **Redes → Reglas de firewall → añadir la IP pública de la máquina virtual**.

Sin esta regla, la API no arranca y en el log aparece `Communications link failure`.

## 16. Los dos ficheros de la máquina

En la máquina solo hacen falta **dos ficheros** en una misma carpeta (las imágenes se descargan del registro):

```bash
mkdir ~/pokeapi && cd ~/pokeapi
```

`docker-compose.yml` (se sube con `scp` o se crea con `nano`):

```yaml
services:

  backend:
    image: ${REGISTRY}/pokeapi-back:1.0
    container_name: pokeapi-back
    restart: unless-stopped
    environment:
      DB_USERNAME: ${DB_USERNAME}
      DB_PASSWORD: ${DB_PASSWORD}

  frontend:
    image: ${REGISTRY}/pokeapi-front:1.0
    container_name: pokeapi-front
    restart: unless-stopped
    depends_on:
      - backend
    environment:
      API_URL: http://backend:8085/api/v1
    ports:
      - "80:4321"
```

Subirlo desde tu ordenador:

```bash
scp -i /ruta/a/clave.pem docker-compose.yml azureuser@IP_DE_LA_MAQUINA:~/pokeapi/
```

`.env`, creado **en la máquina**:

```bash
nano .env
chmod 600 .env
```

```dotenv
REGISTRY=miregistro.azurecr.io
DB_USERNAME=usuario_de_mysql
DB_PASSWORD=contraseña_de_mysql
```

El fichero debe llamarse exactamente **`.env`** (con el punto) y estar en la misma carpeta que el `docker-compose.yml`. Sin espacios alrededor del `=`.

| Clave | Qué hace |
|---|---|
| `image` | En lugar de `build`, se descarga una imagen ya hecha del registro. `${REGISTRY}` se rellena con el valor del `.env` |
| `environment` | Variables de entorno del contenedor. La API recibe aquí el usuario y la contraseña de la base de datos; el front recibe la URL del back |
| `API_URL` | `backend` es el nombre del servicio de arriba: Docker lo resuelve solo dentro de su red |
| `ports: "80:4321"` | Publica el puerto: el 80 de la máquina va al 4321 del contenedor del front. El back **no** publica ningún puerto |
| `depends_on` | El front arranca después del back |
| `restart: unless-stopped` | Si el contenedor se cae, Docker lo reinicia solo (salvo que lo pares tú) |

## 17. Iniciar sesión en el registro, descargar y arrancar

```bash
docker login miregistro.azurecr.io
docker compose config | grep image
docker compose pull
docker compose up -d
docker compose ps
docker compose logs -f backend
```

- `docker compose config | grep image` comprueba que `${REGISTRY}` se ha rellenado (si salen imágenes que empiezan por `/pokeapi...`, falta el `.env`).
- En el log del back hay que esperar a ver `Started PokeApiApplication`. Se sale con `Ctrl+C` (los contenedores siguen funcionando).

## 18. Probar

Desde la propia máquina:

```bash
curl -I http://localhost
```

Y desde el navegador de tu ordenador: **`http://IP_DE_LA_MAQUINA`**. Deben verse los entrenadores y los Pokémon cargados de la base de datos de Azure.

Si responde en `localhost` pero no desde fuera, falta abrir el **puerto 80** en la máquina: **Redes → Reglas de puerto de entrada** → permitir TCP 80.

## 19. Actualizar la aplicación

Cuando cambia el código, no se toca la máquina: se construye y se sube una imagen nueva, y la máquina la descarga.

En tu ordenador (con una etiqueta nueva, por ejemplo `1.1`):

```bash
docker build -t miregistro.azurecr.io/pokeapi-front:1.1 ./front
docker push miregistro.azurecr.io/pokeapi-front:1.1
```

En la máquina, se cambia `1.0` por `1.1` en el `docker-compose.yml` y:

```bash
docker compose pull
docker compose up -d
```

`up -d` solo recrea los contenedores cuya imagen ha cambiado.

---

## 20. Comandos de gestión — referencia rápida

```bash
docker login miregistro.azurecr.io           # iniciar sesión en el registro
docker push miregistro.azurecr.io/img:1.0    # subir una imagen
docker compose pull                          # descargar las imágenes nuevas
docker compose up -d                         # arrancar / actualizar en segundo plano
docker compose ps                            # estado de los servicios
docker compose logs -f backend               # log de un servicio en tiempo real
docker compose restart backend               # reiniciar un servicio
docker compose down                          # parar y borrar los contenedores
```

## Solución de problemas habituales

- **`UNPROTECTED PRIVATE KEY FILE` al hacer SSH**: la llave `.pem` tiene permisos abiertos. `chmod 400 clave.pem`.
- **No se puede entrar por SSH tras apagar la máquina**: al detenerla, Azure puede cambiarle la IP pública (si era dinámica). Mirar la IP nueva en el portal, o fijarla como estática.
- **`WARNING: The REGISTRY variable is not set` / `invalid reference format`**: Compose no encuentra el `.env`. Comprobar que se llama exactamente `.env` (no `env` ni `.env.txt`), que no está vacío y que se ejecuta `docker compose` desde su misma carpeta (`ls -la`).
- **`unauthorized` / `authentication required` al hacer `pull`**: falta `docker login` en la máquina, o las credenciales no son las del registro (usuario administrador activado).
- **`exec format error` al arrancar un contenedor**: la imagen se construyó para otra arquitectura (ARM). Reconstruir con `--platform linux/amd64`.
- **La API se reinicia con `Communications link failure` o `Access denied`**: la IP de la máquina no está en el firewall del MySQL de Azure, o el usuario o la contraseña del `.env` no son correctos.
- **`Read timed out` de Compose, o la máquina deja de responder**: se ha agotado la memoria. Crear swap (paso 14) o usar una máquina con más RAM.
- **El navegador no carga la página, pero `curl -I http://localhost` sí**: el puerto 80 está cerrado en las reglas de entrada de la máquina.
- **Los formularios del front dan error 403**: si se pone un proxy inverso o HTTPS delante, hay que ajustar la comprobación de origen de Astro (`security.checkOrigin`).

---

**Idea clave:** el flujo de despliegue es siempre el mismo: *construir una imagen → subirla a un registro → descargarla en el servidor → arrancarla con compose*. El servidor no necesita el código ni compiladores: solo Docker, el `docker-compose.yml` y las variables de entorno con los secretos, que nunca van dentro de la imagen.

---

Los temas anteriores se enlazan desde el menú lateral de `index.html`.
