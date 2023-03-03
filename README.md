# flexPi

Basado en [plex-rpi](https://github.com/pablokbs/plex-rpi).

La idea es tener un media server con plex, transmission, flexget y samba corriendo cada uno en un container de docker, y que lo podamos desplegar en cualquier dispositivo con docker y docker-compose. En este caso, en una raspberry pi 4b.

## Instalación
Montar raspbian lite o dietPi en la SD card. Una vez hecho, conectarse a la raspberry por ssh (o por consola si se tiene un monitor y teclado) y ejecutar los siguientes comandos:

```bash
# asegurarse de que tu usuario tenga permisos de sudo
sudo useradd miusuario -G sudo

# agregar esta linea al sudoers para correr sudo sin password
%sudo   ALL=(ALL:ALL) NOPASSWD:ALL

# agregar esta linea a sshd_config para que sólo tu usuario pueda hacer ssh
echo "AllowUsers miusuario" | sudo tee -a /etc/ssh/sshd_config

# reiniciar el servicio ssh
sudo systemctl enable ssh && sudo systemctl start ssh

# instalar paquetes básicos
sudo apt-get update && sudo apt-get install -y \
     apt-transport-https \
     ca-certificates \
     curl \
     gnupg2 \
     software-properties-common \
     vim \
     fail2ban \
     ntfs-3g \
     lsb-release

# instalar docker
curl -fsSL https://download.docker.com/linux/debian/gpg | sudo apt-key add -
sudo apt-key fingerprint 0EBFCD88
echo "deb [arch=armhf] https://download.docker.com/linux/debian \
     $(lsb_release -cs) stable" | \
    sudo tee /etc/apt/sources.list.d/docker.list
sudo apt-get update && sudo apt-get install -y --no-install-recommends docker-ce docker-compose

# agregar tu usuario al grupo docker
sudo usermod -a -G docker miusuario
# aquí podemos hacer logout y volver a loguear para que los cambios surtan efecto

# agregar esta linea al final del archivo /etc/default/docker
export DOCKER_TMPDIR="/mnt/storage/docker-tmp"

# montar el disco
sudo fdisk -l
sudo mkdir /mnt/storage
sudo ls -l /dev/disk/by-uuid/ # para obtener el UUID
echo "UUID=XXXX-XXXX /mnt/storage exfat defaults,auto 0 0" | sudo tee -a /etc/fstab
sudo mount -a

# crear directorios
mkdir /mnt/storage/media
mkdir /mnt/storage/media/pelis
mkdir /mnt/storage/media/series
mkdir /mnt/storage/torrents

# clonear este repo
git clone https://github.com/serginator/flexPi.git
cd flexPi

# modificar el archivo .env con las rutas de tus archivos y algunas contraseñas

# modificar en el archivo docker-compose.yaml el UID y GID del usuario que ejecuta los containers, en mi caso será 1000 (el usuario miusuario) y el grupo 1000 (el grupo sudo)

# lanzar los containers
docker-compose up -d
```

## Configuración

### Plex

Para configurar plex, ir a http://ip-de-la-raspberry:32400/web y seguir los pasos de configuración. Para las bibliotecas podemos añadir una de series que apunte a /mnt/storage/media/series y otra de películas que apunte a /mnt/storage/media/pelis.

### Flexget

Para configurar flexget, ir a http://ip-de-la-raspberry:5050 y seguir los pasos de configuración. La contraseña la hemos configurado en el fichero `.env` en la variable `FLEXGET_WEBUI_PASSWD`.

Si queremos que pueda acceder a nuestras listas privadas en trakt.tv y hacer marcar como vistas y demás, tendríamos que ejecutar el siguiente comando en el directorio flexPi:

```bash
docker-compose exec flexget flexget trakt auth miusuario # siendo miusuario el usuario de trakt.tv
```

### Transmission

Para configurar transmission, ir a http://ip-de-la-raspberry:9091

### Samba

Poco que decir aquí, sólo que la contraseña de root es la que hemos configurado en el fichero `.env` en la variable `SAMBA_PASSWD`.

## Otros comandos interesantes

```bash
# ver los logs de los containters
docker-compose logs -t -f

# o de un container específico
docker-compose logs -t -f flexget

# ejecutar tareas de flexget
docker-compose exec flexget flexget execute --dump --task <task_name>
```

## Lista de tareas de flexget

* `sync-series`: sincroniza las series de trakt.tv con las de flexget, con el capítulo por el que vamos
* `sync-movies`: sincroniza las películas de trakt.tv con las de flexget
* `discover-series`: se baja torrents de los capítulos de las series que ha sincronizado con trakt.tv
* `discover-movies`: se baja torrents de las películas que ha sincronizado con trakt.tv
* `sort-series`: se mueven los archivos descargados a la carpeta de series
* `sort-movies`: se mueven los archivos descargados a la carpeta de películas
* `remove-stale-torrents`: se borran los ficheros de la carpeta torrent que se han descargado y o bien se han compartido o bien llevan tiempo sin ser compartidos
