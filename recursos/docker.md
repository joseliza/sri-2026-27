# 🐳 Guía básica de Docker para SRI

Docker nos permite levantar un servicio (DNS, web, FTP, correo…) en segundos, sin instalar una máquina virtual completa y sin ensuciar el sistema. Durante el curso lo usaremos siempre que sea posible para practicar.

> **Idea clave:** una máquina virtual arranca un sistema operativo entero; un contenedor solo ejecuta **un proceso** aislado, compartiendo el kernel del anfitrión. Por eso es ligero y rápido.

## 1. Conceptos

| Concepto | Qué es | Analogía |
|----------|--------|----------|
| **Imagen** | Plantilla de solo lectura con el programa y sus dependencias (p. ej. `nginx`, `debian`) | El ISO / la plantilla de una VM |
| **Contenedor** | Instancia en ejecución de una imagen | La VM encendida |
| **Registro** | Repositorio de imágenes (por defecto [Docker Hub](https://hub.docker.com)) | Un almacén de ISOs |
| **Volumen** | Almacenamiento persistente fuera del contenedor | Un disco duro extra |
| **Red** | Red virtual que conecta contenedores entre sí y con el exterior | Un switch virtual |
| **Dockerfile** | Receta para construir una imagen propia | Un script de instalación |
| **Compose** | Fichero YAML que describe varios contenedores a la vez | Un "escenario" completo |

Los contenedores son **efímeros**: al borrarlos se pierde todo lo que no esté en un volumen o en una carpeta montada.

## 2. Instalación

### Debian

```bash
sudo apt update
sudo apt install docker.io docker-compose
sudo usermod -aG docker $USER     # para no usar sudo en cada comando
newgrp docker                     # o cierra sesión y vuelve a entrar
docker run hello-world            # comprobación
```

Alternativa con el repositorio oficial de Docker (versión más reciente, incluye `docker compose`): <https://docs.docker.com/engine/install/debian/>

### Windows 10/11 (equipo personal)

Se usa **Docker Desktop**, que ejecuta los contenedores Linux sobre WSL 2.

1. **Requisitos:** Windows 10 (64 bits, versión 22H2) o Windows 11, virtualización activada en la BIOS/UEFI (comprueba en el Administrador de tareas → Rendimiento → CPU → «Virtualización: habilitada»).
2. **Instalar WSL 2** (PowerShell como administrador) y reiniciar:
   ```powershell
   wsl --install
   ```
   Más información: <https://learn.microsoft.com/es-es/windows/wsl/install>
3. **Descargar e instalar Docker Desktop:** <https://docs.docker.com/desktop/setup/install/windows-install/>
   Durante la instalación deja marcada la opción **«Use WSL 2 instead of Hyper-V»**.
4. **Abrir Docker Desktop**, aceptar las condiciones y esperar a que el icono de la ballena indique que está en marcha.
5. **Comprobar** desde PowerShell:
   ```powershell
   docker version
   docker run hello-world
   ```

Notas para Windows:

- Docker Desktop tiene licencia gratuita para uso personal y educativo; para uso profesional en empresas grandes es de pago.
- Ejecuta los comandos desde PowerShell o desde una terminal WSL (Debian/Ubuntu). En rutas de volúmenes, `$(pwd)` es de bash; en PowerShell usa `${PWD}`.
- Las redes `macvlan` y `host` **no funcionan igual** en Docker Desktop (los contenedores corren dentro de una VM), así que las prácticas de DHCP no se pueden hacer en Windows: hazlas en Debian.
- Alternativa sin Docker Desktop: instalar Docker Engine dentro de una distribución WSL (Debian) siguiendo la guía de Debian anterior.

> Con la instalación oficial de Docker, Compose se usa como `docker compose` (sin guion). Aquí usaremos esa forma.

## 3. Comandos esenciales

### Imágenes

```bash
docker pull debian:12          # descargar una imagen (nombre:etiqueta)
docker images                  # listar imágenes locales
docker rmi debian:12           # borrar una imagen
```

### Contenedores

```bash
docker run -it debian:12 bash              # arrancar y entrar de forma interactiva
docker run -d --name web -p 8080:80 nginx  # en segundo plano, con nombre y puerto publicado
docker ps                                  # contenedores en ejecución
docker ps -a                               # todos, incluso los parados
docker stop web                            # parar
docker start web                           # volver a arrancar
docker rm web                              # borrar (parado)
docker rm -f web                           # borrar aunque esté en ejecución
```

### Inspeccionar y depurar

```bash
docker logs web                # salida del servicio
docker logs -f web             # seguirla en directo
docker exec -it web bash       # abrir una shell dentro de un contenedor ya en marcha
docker inspect web             # todos los detalles (IP, montajes, variables…)
docker stats                   # consumo de CPU y memoria
```

### Limpieza

```bash
docker container prune         # borra los contenedores parados
docker system prune            # borra también redes e imágenes sin uso
```

## 4. Opciones de `docker run` que más usaremos

| Opción | Para qué sirve | Ejemplo |
|--------|----------------|---------|
| `-d` | Segundo plano | `docker run -d nginx` |
| `-it` | Terminal interactiva | `docker run -it debian bash` |
| `--name` | Ponerle nombre | `--name dns1` |
| `-p host:contenedor` | Publicar un puerto | `-p 8080:80` |
| `-v origen:destino` | Montar carpeta o volumen | `-v ./web:/usr/share/nginx/html` |
| `-e VAR=valor` | Variable de entorno | `-e MYSQL_ROOT_PASSWORD=1234` |
| `--network` | Conectar a una red | `--network mired` |
| `--rm` | Borrar al terminar | `docker run --rm -it debian bash` |
| `--restart` | Política de reinicio | `--restart unless-stopped` |

## 5. Volúmenes y carpetas montadas

Para que la configuración de un servicio sobreviva al contenedor (y poder editarla desde tu equipo):

```bash
# Montar una carpeta local (bind mount): editas fuera, se ve dentro
docker run -d --name web -p 8080:80 \
  -v $(pwd)/html:/usr/share/nginx/html:ro nginx

# Volumen gestionado por Docker
docker volume create datos
docker run -d -v datos:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=1234 mariadb
```

Prueba: crea `html/index.html`, abre `http://localhost:8080` y modifica el fichero.

## 6. Redes

Cada contenedor recibe una IP en una red virtual. Con una **red definida por el usuario**, los contenedores se resuelven por nombre (DNS interno de Docker):

```bash
docker network create mired
docker run -d --name servidor --network mired nginx
docker run --rm -it --network mired debian bash
# dentro:  apt update && apt install -y iputils-ping && ping servidor
```

Tipos de red que nos interesan:

| Driver | Comportamiento | Uso en SRI |
|--------|----------------|------------|
| `bridge` (defecto) | Red privada NAT; hay que publicar puertos con `-p` | Web, FTP, correo, DNS |
| `host` | El contenedor usa directamente la red del anfitrión | Servicios que necesitan puertos "reales" |
| `macvlan` / `ipvlan` | El contenedor aparece con IP propia en la LAN | Prácticas de DHCP (ver aviso) |
| `none` | Sin red | Aislamiento total |

Comandos útiles: `docker network ls`, `docker network inspect mired`, `docker network rm mired`.

## 7. Dockerfile: crear tu propia imagen

```dockerfile
FROM debian:12
RUN apt-get update && apt-get install -y --no-install-recommends bind9 \
    && rm -rf /var/lib/apt/lists/*
COPY named.conf.local /etc/bind/named.conf.local
EXPOSE 53/udp 53/tcp
CMD ["named", "-g", "-u", "bind"]
```

```bash
docker build -t mi-dns .
docker run -d --name dns1 -p 5353:53/udp -p 5353:53/tcp mi-dns
```

- `FROM`: imagen base · `RUN`: ejecuta al construir · `COPY`: copia ficheros · `EXPOSE`: documenta puertos · `CMD`: proceso principal del contenedor.
- El contenedor vive mientras viva su proceso principal, y este debe ejecutarse **en primer plano** (`-g`, `daemon off;`…).

## 8. Docker Compose: escenarios completos

Un `compose.yaml` describe varios servicios y sus redes. Ejemplo: web + base de datos.

```yaml
services:
  web:
    image: nginx
    ports:
      - "8080:80"
    volumes:
      - ./html:/usr/share/nginx/html:ro
    depends_on:
      - db
  db:
    image: mariadb
    environment:
      MYSQL_ROOT_PASSWORD: "1234"
    volumes:
      - datos:/var/lib/mysql

volumes:
  datos:
```

```bash
docker compose up -d       # crear y arrancar todo
docker compose ps          # estado
docker compose logs -f     # registros
docker compose down        # parar y borrar contenedores y redes
docker compose down -v     # ... y también los volúmenes
```

Con Compose, cada servicio es accesible por su nombre (`web`, `db`) desde los demás. Un fichero, un comando: el escenario completo es reproducible y se puede entregar como práctica.

## 9. Docker en el módulo SRI

| UT | Servicio | Imagen habitual | Observaciones |
|----|----------|-----------------|---------------|
| 1 | DHCP | `networkboot/dhcpd`, `debian` + `isc-dhcp-server` | Necesita red `macvlan` o `host`; en `bridge` no recibe los broadcast de la LAN. Ojo con no interferir con el DHCP del aula |
| 2 | DNS | `ubuntu/bind9`, `debian` + `bind9` | Publicar 53/udp **y** 53/tcp; usar un puerto alto si el 53 está ocupado (`systemd-resolved`) |
| 3 | SSH | `debian` + `openssh-server` | Publicar el 22 en otro puerto (p. ej. `-p 2222:22`) |
| 4 | FTP | `delfer/alpine-ftp-server`, `stilliard/pure-ftpd` | El modo pasivo exige publicar un rango de puertos |
| 5 | Web | `nginx`, `httpd`, `php:apache` | Montar el directorio del sitio como volumen |
| 6 | Correo | `docker-mailserver`, `mailhog` | Práctica avanzada; MailHog sirve para pruebas |
| 8 | Mensajería | `prosody`, `ejabberd` | Servidor XMPP |
| 9 | Voz IP | `asterisk`, `freepbx` | Necesita `host` por el rango RTP |

> Es una referencia orientativa: las imágenes concretas se indicarán en cada unidad.

## 10. Buenas prácticas y errores frecuentes

- **Persistencia:** si no usas volumen, al borrar el contenedor pierdes los datos.
- **Un servicio por contenedor.** No instales "de todo" en uno solo; usa varios y conéctalos por red.
- **Etiquetas fijas** (`nginx:1.27`) en lugar de `latest` para que la práctica sea reproducible.
- **Puerto ocupado** (`address already in use`): cambia el puerto de la izquierda de `-p` o para el servicio que lo usa.
- **El contenedor se para al instante:** revisa `docker logs <nombre>`; casi siempre el proceso principal ha fallado o se ha ido a segundo plano.
- **Sin `--privileged`** salvo que sea imprescindible: rompe el aislamiento.
- **Contraseñas en Compose/Dockerfile** solo en entornos de práctica; nunca las subas a un repositorio real.
- **Limpia** de vez en cuando (`docker system prune`) para no llenar el disco.

## 11. Chuleta rápida

```text
docker run -d --name X -p H:C imagen     crear y arrancar
docker ps [-a]                           listar
docker logs -f X                         ver registros
docker exec -it X bash                   entrar
docker stop X && docker rm X             parar y borrar
docker compose up -d / down              escenarios completos
docker system prune                      limpiar
```

## 12. Ejercicios de calentamiento

1. Arranca un `nginx` en el puerto 8080 y cambia su página de inicio con un volumen.
2. Lanza dos contenedores `debian` en una red propia y haz `ping` entre ellos por nombre.
3. Crea un `compose.yaml` con `nginx` + `mariadb` y comprueba con `docker compose ps` que ambos están activos.
4. Escribe un `Dockerfile` que instale `bind9` y arranca un DNS en el puerto 5353; consúltalo con `dig @127.0.0.1 -p 5353`.

## 📚 Para profundizar

- Documentación oficial: <https://docs.docker.com/get-started/>
- Referencia de Dockerfile: <https://docs.docker.com/reference/dockerfile/>
- Referencia de Compose: <https://docs.docker.com/reference/compose-file/>
- Imágenes: <https://hub.docker.com>
