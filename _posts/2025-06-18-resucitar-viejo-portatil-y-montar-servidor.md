---
layout: post
title: "Cómo resucitar un viejo portátil y montar un 2º servidor casero, tras quemar el 1º."
date: 2025-06-18
author: Damalga
comments: false
---
## Montar tu propia nube casera con Nextcloud y Navidrome en un viejo portátil con Debian + XFCE

### Introducción

Este documento es una guía para convertir un portátil viejo en una nube autosuficiente, equipada con:

- **Nextcloud** (tu Dropbox libre)
- **Navidrome** (tu Spotify privado)
- **Tailscale** (conexión remota sin dolores de cabeza)

### Requisitos

- Un ordenador  (en este caso, un HP Elitebook del 2015)
- Un disco duro externo (en este caso de 4TB)
- Debian 12 con entorno XFCE instalado. Seguramente el proceso sea el mismo con cualquier distribución Linux basada en Debian. En mi caso usé Debian directamente y no Ubuntu o Tails (por decir alguna), por su estabilidad y el entorno de  escritorio XFCE por lo liviano, ya que mi portatil es viejito así que cuanto menos tenga que procesar pues mejor.
- Conexión a Internet.
- Ganas de cacharrear.

---

### Paso 1: Instalar Debian 12 + XFCE

- Selecciona XFCE como entorno de escritorio (ligero y funcional).
- Particiona el disco según necesidad (modo experto si te atreves).
- Crea tu usuario.

---

### Paso 2: Conectar y montar el disco duro externo

- Conéctalo y crea un punto de montaje permanente:

```bash
sudo mkdir -p /mnt/nube
sudo blkid # Identifica UUID del disco
sudo nano /etc/fstab
# Añade la línea con el UUID, punto de montaje y tipo de sistema de archivos
```

---

### Paso 3: Instalar y configurar Nextcloud

#### Dependencias:

```bash
sudo apt update && sudo apt install apache2 mariadb-server libapache2-mod-php php php-mysql php-gd php-json php-curl php-mbstring php-xml php-zip unzip
```

#### Base de datos:

```bash
sudo mysql -u root
CREATE DATABASE nextcloud;
CREATE USER 'nextclouduser'@'localhost' IDENTIFIED BY 'tu_contra_segura';
GRANT ALL PRIVILEGES ON nextcloud.* TO 'nextclouduser'@'localhost';
FLUSH PRIVILEGES;
EXIT;
```

#### Instalar Nextcloud:

```bash
cd /tmp
wget https://download.nextcloud.com/server/releases/latest.zip
unzip latest.zip
sudo mv nextcloud /var/www/
sudo chown -R www-data:www-data /var/www/nextcloud
```

#### Configurar Apache:

```bash
sudo nano /etc/apache2/sites-available/nextcloud.conf
```

Contenido:

```
<VirtualHost *:80>
    DocumentRoot /var/www/nextcloud/
    ServerName localhost

    <Directory /var/www/nextcloud/>
        Require all granted
        AllowOverride All
        Options FollowSymLinks MultiViews
    </Directory>
</VirtualHost>
```

```bash
sudo a2ensite nextcloud.conf
sudo a2enmod rewrite headers env dir mime
sudo systemctl restart apache2
```

Ahora entra a [http://localhost](http://localhost) y completa instalación gráfica.

---

### Paso 4: Añadir tu dominio Tailscale como "trusted domain"

Edita el archivo de configuración:

```bash
sudo nano /var/www/nextcloud/config/config.php
```

Agrega tu IP Tailscale o dominio en `trusted_domains`

---

### Paso 5: Instalar Tailscale

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```

Inicia sesión con tu cuenta y apunta la IP generada.

---

### Paso 6: Instalar Navidrome

```bash
cd /opt
sudo wget https://github.com/navidrome/navidrome/releases/latest/download/navidrome_0.55.2_linux_amd64.tar.gz
sudo tar -xvzf navidrome_0.55.2_linux_amd64.tar.gz
sudo rm navidrome_0.55.2_linux_amd64.tar.gz
```

#### Crear archivo de configuración:

```bash
sudo nano /opt/navidrome/navidrome.toml
```

Contenido básico:

```
MusicFolder = "/mnt/nube/Music"
Port = "4533"
```

#### Crear servicio systemd:

```bash
sudo nano /etc/systemd/system/navidrome.service
```

Contenido:

```
[Unit]
Description=Navidrome Music Server
After=network.target

[Service]
User=kasper8
ExecStart=/opt/navidrome/navidrome
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reexec
sudo systemctl daemon-reload
sudo systemctl enable --now navidrome
```

Navidrome debería estar disponible en [http://localhost:4533](http://localhost:4533) o a través de tu IP Tailscale.

---

### Paso 7: Usar Symfonium en el móvil

- Descarga desde F-Droid o Google Play
- Conéctate a la IP Tailscale: http\://:4533
- Usuario y contraseña según hayas definido en Navidrome

---

### Fin

Ya tienes nube privada y sistema de streaming de música montado. Todo funcionando en una máquina reciclada. ¿Spotify? ¿Dropbox? No, gracias.

¡Disfruta la independencia digital!
