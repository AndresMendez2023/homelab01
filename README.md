# Mi Homelab: de un MacBook viejo a un servidor casero

Documentación paso a paso de cómo convertí un **MacBook Pro de 2012** en un servidor casero con **Ubuntu Server**, accesible de forma segura desde cualquier lugar. Está escrita para que la pueda seguir alguien **sin experiencia previa**.

> **Aviso de seguridad:** en esta guía uso datos de ejemplo (`<IP-DEL-SERVIDOR>`, `usuario`, `TU_CLAVE`). Si repites el proyecto, usa los tuyos y **nunca publiques contraseñas, llaves privadas ni tokens**.

---

## 1. Resumen del proyecto

| Tema | Detalle |
|---|---|
| **Objetivo** | Aprender Linux, redes, contenedores y seguridad con un proyecto real, y tener un servidor propio 24/7 |
| **Equipo** | MacBook Pro (2012), 16 GB de RAM, SSD de ~480 GB |
| **Sistema operativo** | Ubuntu Server 26.04.1 LTS (antes tenía Fedora, que se borró) |
| **Nombre del servidor** | `homelab01` |
| **Servicios instalados** | SSH, firewall (UFW), Docker, Tailscale (VPN), Pi-hole (bloqueo de anuncios y DNS) |
| **Seguridad aplicada** | Firewall, SSH solo con llave, sin login de root, fail2ban, VPN sin abrir puertos, copias cifradas automáticas | 
| **Consumo estimado** | 8 a 15 W en reposo (estimación, no medido) |
| **Servicios instalados** | SSH, firewall (UFW), Docker, Tailscale (VPN), Pi-hole, Nextcloud (nube personal) |
| **Seguridad aplicada** | Firewall, SSH solo con llave, sin login de root, fail2ban, VPN sin abrir puertos, HTTPS dentro de la VPN, copias cifradas automáticas |

### Qué logré

- Un servidor funcionando **con la tapa cerrada**, las 24 horas.
- Acceso remoto seguro **desde casa y desde fuera** sin abrir puertos en el router.
- Un bloqueador de anuncios para toda mi red.
- Una base lista para montar mi propia nube de archivos.

---

## 2. Esquema general

```
 [ Mi PC con Windows ] ──SSH (con llave)──┐
                                          ▼
 [ iPhone ] ──Tailscale (VPN)──►  [ Servidor homelab01 (MacBook) ]
                                          │  Ubuntu Server
                                          ├─ Firewall (UFW)
                                          ├─ Docker
                                          │    └─ Pi-hole (DNS y bloqueo de anuncios)
                                          └─ Tailscale (VPN)
                                          └─ Nextcloud (nube) + PostgreSQL
                                          │
                                  [ Router ]── Internet
```

---

## 3. Glosario para principiantes

| Término | Qué significa |
|---|---|
| **Servidor** | Un ordenador que se queda encendido ofreciendo servicios a otros dispositivos |
| **Linux / Ubuntu** | Sistema operativo gratuito; Ubuntu Server es la versión sin escritorio gráfico, pensada para servidores |
| **Terminal / consola** | Pantalla donde se controla el equipo escribiendo comandos |
| **ISO** | Archivo que contiene un sistema operativo para instalarlo |
| **SSH** | Forma segura de controlar otro ordenador desde tu terminal, por red |
| **IP** | La "dirección" de un dispositivo en una red |
| **DHCP** | El router reparte direcciones IP automáticamente; una IP "dinámica" puede cambiar |
| **IP fija (estática)** | Una dirección que no cambia, necesaria para un servidor |
| **DNS** | La "agenda telefónica" de internet: traduce nombres (google.com) a direcciones IP |
| **Puerto** | Una "puerta" numerada de un equipo, usada por un servicio (SSH usa el 22, DNS el 53) |
| **Firewall** | Filtro que decide qué conexiones se permiten entrar o salir |
| **VPN** | Túnel privado y cifrado entre dispositivos |
| **Docker / contenedor** | Forma de ejecutar programas aislados y fáciles de instalar o borrar |
| **Pi-hole** | Programa que actúa como DNS y bloquea dominios de publicidad y rastreo |
| **Llave SSH** | Par de archivos (privada y pública) que sustituye a la contraseña para entrar por SSH |
| **sudo** | Ejecutar un comando con permisos de administrador |
| **Nextcloud** |	Programa que convierte un servidor en una nube personal de archivos |
| **Sincronización** |	Mantener una carpeta idéntica en dos dispositivos. Si borras en uno, se borra en el otro |
| **Copia de seguridad** |	Duplicado guardado en otro dispositivo, con versiones, para recuperar datos. No es lo mismo que sincronizar |
| **HTTPS** | certificado	Conexión cifrada y verificada por un certificado |
| **Volcado de base de datos** |	Archivo con todo el contenido de una base de datos, que se puede restaurar |
| **Modo mantenimiento** |	Estado en que Nextcloud no acepta cambios, para copiar datos coherentes |
| **Cron** |	Programador de tareas que ejecuta comandos a una hora fija |
---

## 4. Paso a paso

### Paso 1. Preparar el instalador (en Windows)

1. Descargar **Ubuntu Server 26.04.1 LTS** desde ubuntu.com/download/server (la ISO pesa ~2 GB).
2. Conectar un USB de **8 GB o más** (se borrará todo lo que tenga).
3. Grabar la ISO con **Rufus** (esquema de partición **GPT**, modo imagen ISO). También sirve balenaEtcher.

### Paso 2. Arrancar el MacBook desde el USB

1. Apagar el Mac, conectar el USB y un **cable Ethernet**.
2. Encender manteniendo pulsada la tecla **Option (⌥)**.
3. Elegir la opción **EFI Boot**.
4. En el menú de Ubuntu, elegir **Try or Install Ubuntu Server**.

> **Por qué cable:** el Wi-Fi de este modelo suele necesitar drivers extra, y con cable la instalación es más sencilla.

### Paso 3. Instalar Ubuntu Server

En el instalador (se maneja con flechas, Tab, espacio y Enter):

1. **Idioma y teclado:** elegir los propios.
2. **Red:** debe aparecer el cable con una IP asignada por DHCP.
3. **Almacenamiento:** usar **todo el disco** (esto **borra** el sistema anterior).
4. **Ajuste del tamaño:** el instalador asigna por defecto solo 100 GB a la raíz `/`, y se amplió al máximo disponible para aprovechar todo el SSD.
5. **Perfil:** nombre, nombre del servidor (`homelab01`), usuario y contraseña.
6. **Ubuntu Pro:** omitir (*Skip for now*).
7. **SSH:** marcar **Install OpenSSH server**.
8. **Snaps:** no marcar nada.
9. Esperar, elegir **Reboot Now** y **retirar el USB** cuando lo pida.

### Paso 4. Primer inicio y actualización

Iniciar sesión con el usuario creado y actualizar el sistema:

```bash
sudo apt update && sudo apt upgrade -y
sudo reboot
```

> **Concepto:** `apt` es el instalador de programas de Ubuntu. Actualizar corrige fallos y vulnerabilidades.

### Paso 5. Conectarme por SSH desde Windows

Desde PowerShell (Windows), no desde el propio servidor:

```powershell
ssh usuario@<IP-DEL-SERVIDOR>
```

La primera vez pregunta si confías en el equipo (**yes**). A partir de aquí ya no hace falta pantalla ni teclado en el Mac.

### Paso 6. Que siga encendido con la tapa cerrada

Por defecto Ubuntu suspende el equipo al cerrar la tapa. Para evitarlo:

```bash
sudo mkdir -p /etc/systemd/logind.conf.d
sudo tee /etc/systemd/logind.conf.d/lid.conf > /dev/null << 'EOF'
[Login]
HandleLidSwitch=ignore
HandleLidSwitchExternalPower=ignore
HandleLidSwitchDocked=ignore
EOF
sudo systemctl restart systemd-logind
```

**Prueba:** cerrar la tapa y hacer `ping <IP-DEL-SERVIDOR>` desde otro equipo. Si responde, funciona.

### Paso 7. Activar el firewall (UFW)

```bash
sudo ufw allow OpenSSH
sudo ufw enable
sudo ufw status
```

> **Importante:** permitir SSH **antes** de activar el firewall. Si no, el firewall cortaría tu propia conexión.

### Paso 8. Ajustar la zona horaria

```bash
sudo timedatectl set-timezone TU_ZONA_HORARIA   # por ejemplo: Europe/Madrid
timedatectl
```

### Paso 9. Instalar Docker

```bash
sudo apt install -y docker.io docker-compose-v2
sudo usermod -aG docker $USER
exit    # cerrar sesión y volver a entrar para aplicar el grupo
docker run hello-world
```

Si aparece **"Hello from Docker!"**, quedó instalado y funcionando.

> **Ojo de seguridad:** pertenecer al grupo `docker` equivale en la práctica a tener permisos de administrador. Además, **Docker publica puertos saltándose las reglas de UFW**, así que nunca hay que exponer esos puertos a internet desde el router.

### Paso 10. Instalar Tailscale (VPN)

Tailscale permite llegar al servidor desde cualquier sitio **sin abrir puertos en el router**.

1. Crear una cuenta gratuita en tailscale.com.
2. En el servidor:
   ```bash
   curl -fsSL https://tailscale.com/install.sh | sh
   sudo tailscale up
   ```
3. Abrir el enlace que muestra, en el navegador, e iniciar sesión para autorizar el servidor.
4. Ver su IP de VPN con `tailscale ip -4` (empieza por `100.`).
5. Instalar Tailscale también en el PC y el móvil, con la **misma cuenta**.
6. En el panel de administración (login.tailscale.com/admin/machines), en `homelab01` elegir **Disable key expiry** para que el servidor no se desconecte con el tiempo.

**Uso fuera de casa:** `ssh usuario@100.x.x.x` (la IP de Tailscale), con Tailscale activo en el dispositivo.

### Paso 11. IP fija en el servidor

El router asigna IPs por DHCP y pueden cambiar. Un servidor necesita una dirección estable. No pude acceder al panel del router, así que la fijé **en el propio servidor** con *netplan*.

1. Copia de seguridad de la configuración:
   ```bash
   sudo cp /etc/netplan/*.yaml ~/netplan-backup.yaml
   ```
2. Archivo con la IP fija (cambia los valores por los de tu red):
   ```yaml
   network:
     ethernets:
       enp1s0f0:            # nombre de tu interfaz de red
         dhcp4: false
         addresses:
           - 192.168.X.X/24     # la IP que ya tenía el servidor
         routes:
           - to: default
             via: 192.168.X.1   # la puerta de enlace (router)
         nameservers:
           addresses: [192.168.X.1, 1.1.1.1]
     version: 2
   ```
3. Probar con seguridad:
   ```bash
   sudo netplan try
   ```
   Hay que pulsar **Enter** antes de que termine la cuenta atrás para confirmar. Si no, el sistema **revierte solo** los cambios (así no te quedas sin conexión).
4. Verificar: `ip a` y `ping -c 3 google.com`.

### Paso 12. Instalar Pi-hole (con Docker)

**Qué hace:** actúa como servidor DNS de la red. Si un dominio está en su lista de publicidad o rastreo, no responde y el anuncio no carga.

1. **Liberar el puerto 53**, que Ubuntu usa por defecto y chocaría con Pi-hole:
   ```bash
   sudo mkdir -p /etc/systemd/resolved.conf.d
   sudo tee /etc/systemd/resolved.conf.d/pihole.conf > /dev/null << 'EOF'
   [Resolve]
   DNS=1.1.1.1
   DNSStubListener=no
   EOF
   sudo ln -sf /run/systemd/resolve/resolv.conf /etc/resolv.conf
   sudo systemctl restart systemd-resolved
   ```
2. **Crear el archivo de Docker Compose** (`~/pihole/docker-compose.yml`):
   ```yaml
   services:
     pihole:
       image: pihole/pihole:latest
       container_name: pihole
       ports:
         - "53:53/tcp"
         - "53:53/udp"
         - "8080:80/tcp"
       environment:
         TZ: TU_ZONA_HORARIA
         FTLCONF_webserver_api_password: "TU_CLAVE"
         FTLCONF_dns_listeningMode: all
       volumes:
         - ./etc-pihole:/etc/pihole
       restart: unless-stopped
   ```
3. **Arrancarlo:**
   ```bash
   cd ~/pihole && docker compose up -d
   docker ps
   ```
4. **Probar el DNS:** `nslookup google.com <IP-DEL-SERVIDOR>`. Si responde, funciona.
5. **Panel de administración:** `http://<IP-DEL-SERVIDOR>:8080/admin`.
6. **Usarlo desde mis dispositivos:** en el panel de Tailscale (sección *DNS*), añadir como nameserver personalizado la IP de Tailscale del servidor y activar *Override DNS servers*. Así el PC y el móvil usan Pi-hole dentro y fuera de casa.

> **Riesgo a tener en cuenta:** si el servidor se apaga, los dispositivos con ese DNS parecen sin internet. Se soluciona desactivando *Override DNS servers* en Tailscale.

### Paso 13. Entrar solo con llave SSH (sin contraseña)

**Cómo funciona:** hay dos archivos. La **llave privada** se queda en mi PC y es mi identidad. La **llave pública** se copia al servidor y solo sirve para reconocerme. Al conectarme, el servidor lanza un reto que solo se resuelve con la privada, así que **nunca viaja por la red**. Además, protegí la llave con una **frase de contraseña**.

1. **Crear la llave (en Windows):**
   ```powershell
   ssh-keygen -t ed25519
   ```
2. **Copiar la pública al servidor:**
   ```powershell
   type $env:USERPROFILE\.ssh\id_ed25519.pub | ssh usuario@<IP-DEL-SERVIDOR> "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
   ```
3. **Probar que entra con la llave** antes de seguir.
4. **Desactivar la contraseña** (manteniendo otra sesión abierta como salvavidas):
   ```bash
   sudo tee /etc/ssh/sshd_config.d/10-hardening.conf > /dev/null << 'EOF'
   PasswordAuthentication no
   KbdInteractiveAuthentication no
   PermitRootLogin no
   EOF
   sudo sshd -t                 # comprueba que no hay errores
   sudo systemctl reload ssh
   sudo sshd -T | grep -E "passwordauthentication|permitrootlogin"
   ```
   El archivo empieza por `10-` para que tenga prioridad sobre otros que pudieran reactivar la contraseña.
5. **Probar desde una ventana nueva:**
   - `ssh usuario@<IP-DEL-SERVIDOR>` debe entrar con la frase de la llave.
   - `ssh -o PubkeyAuthentication=no usuario@<IP-DEL-SERVIDOR>` debe responder `Permission denied (publickey)`, lo que confirma que la contraseña ya no sirve.
6. **Guardar una copia de la llave privada** en un USB guardado en un lugar seguro. Si se pierde el PC sin copia, se pierde el acceso por SSH.

**Qué pasa si cambio de ordenador:** creo una llave nueva en el equipo nuevo, añado su parte pública a `~/.ssh/authorized_keys` del servidor (desde el equipo viejo, mientras aún puedo entrar) y, si dejo de usar el viejo, borro su línea de ese archivo.

Paso 14. Copias de seguridad automáticas (restic)

Por qué: un servidor puede fallar (disco dañado, error mío, apagón brusco). Una copia de seguridad es un duplicado de lo importante guardado en otro dispositivo, para poder recuperarlo. Si la copia está en el mismo disco que los originales, no protege de nada.

Regla 3-2-1: 3 copias de los datos, en 2 tipos de medios distintos, con 1 copia fuera de casa. Esta guía cubre la copia local; la copia fuera de casa queda pendiente (ver Limitaciones).

Herramienta: restic. Hace copias cifradas, guarda versiones con fecha y solo copia lo que cambió, así que ocupan muy poco espacio.

Qué copio:

Ruta	Qué contiene
~/pihole	El docker-compose.yml y la configuración de Pi-hole
/etc	Configuración del sistema: red, firewall, SSH, fail2ban, hora
~/.ssh	Las llaves autorizadas para entrar al servidor

Qué no copio: el sistema operativo y las imágenes de Docker, porque se reinstalan o se vuelven a descargar solos. Lo valioso es la configuración.

Destino: una memoria USB de 7 GB formateada en NTFS, conectada al servidor.

14.1 Identificar la USB (sin equivocarme de disco)

Antes de montar nada, hay que estar seguro de cuál es cada dispositivo. Estos comandos solo leen, no cambian nada:

bash
lsblk -f
lsblk -o NAME,SIZE,TYPE,TRAN,MODEL

La USB aparece con TRAN = usb y su tamaño (en mi caso ~7,5 GB). El disco interno del servidor aparece como sata y no se toca.

14.2 Instalar y montar
bash
sudo apt install -y ntfs-3g restic
sudo mkdir -p /mnt/backup
sudo mount -o uid=$(id -u),gid=$(id -g) /dev/sdb2 /mnt/backup   # sdb2 = mi USB; el tuyo puede ser otro
mountpoint /mnt/backup

Importante: mountpoint debe responder is a mountpoint antes de crear el repositorio. Si no, la copia se guardaría en el disco interno en lugar de la USB.

14.3 Crear el repositorio de copias
bash
restic init --repo /mnt/backup/restic

Pide una contraseña de cifrado. Se guarda en un gestor de contraseñas antes de escribirla: restic no puede recuperarla ni restablecerla, y sin ella las copias no se pueden abrir.

14.4 Primera copia
bash
sudo restic -r /mnt/backup/restic backup ~/pihole ~/.ssh /etc

La primera vez la hice sin sudo y salió el aviso at least one source file could not be read: algunos archivos de /etc solo los puede leer el administrador. Con sudo la copia queda completa.

14.5 Probar la restauración

Una copia que nunca se ha probado restaurar no es de fiar:

bash
sudo restic -r /mnt/backup/restic restore latest --target /tmp/prueba-restauracion --include /etc/hostname
sudo cat /tmp/prueba-restauracion/etc/hostname     # debe mostrar el nombre del servidor
sudo rm -r /tmp/prueba-restauracion

Se lee con sudo cat porque, al restaurar con sudo, los archivos quedan a nombre de root.

14.6 Montaje automático de la USB

Sin esto, la USB deja de estar montada cada vez que el servidor se reinicia y las copias fallarían.

bash
sudo cp /etc/fstab ~/fstab.bak                        # copia de seguridad del archivo
sudo -v                                               # pide la contraseña una sola vez
UUID_USB=$(sudo blkid -s UUID -o value /dev/sdb2)     # identificador único de la USB
echo "UUID=$UUID_USB /mnt/backup ntfs-3g uid=$(id -u),gid=$(id -g),nofail,x-systemd.device-timeout=10 0 0" | sudo tee -a /etc/fstab
sudo umount /mnt/backup && sudo mount -a && mountpoint /mnt/backup
sudo systemctl daemon-reload
Se usa el UUID y no sdb2, porque el nombre del dispositivo puede cambiar entre reinicios y el UUID no.
nofail evita que el servidor se quede bloqueado al arrancar si la USB no está conectada.
Si algo sale mal: sudo cp ~/fstab.bak /etc/fstab.
14.7 Automatizar la copia cada noche

a) Guardar la contraseña de restic en un archivo que solo lee el administrador:

bash
sudo nano /root/.restic-password      # escribir solo la contraseña, guardar y salir
sudo chmod 600 /root/.restic-password
sudo ls -l /root/.restic-password     # debe empezar por -rw-------

b) Crear el script /usr/local/bin/backup-homelab.sh:

bash
#!/bin/bash
set -euo pipefail
export RESTIC_REPOSITORY=/mnt/backup/restic
export RESTIC_PASSWORD_FILE=/root/.restic-password
if ! mountpoint -q /mnt/backup; then
  echo "$(date): USB no montada, copia cancelada"
  exit 1
fi
restic backup /home/usuario/pihole /home/usuario/.ssh /etc
restic forget --keep-daily 7 --keep-weekly 4 --keep-monthly 6 --prune
La comprobación mountpoint impide que la copia se guarde en el disco interno si la USB no está.
restic forget --prune aplica la política de retención: conserva 7 copias diarias, 4 semanales y 6 mensuales, y borra el resto para no llenar la USB.

Se hace ejecutable y se prueba a mano:

bash
sudo chmod +x /usr/local/bin/backup-homelab.sh
sudo /usr/local/bin/backup-homelab.sh

c) Programarlo con cron (todos los días a las 3:30):

bash
echo "30 3 * * * root /usr/local/bin/backup-homelab.sh >> /var/log/backup-homelab.log 2>&1" | sudo tee /etc/cron.d/backup-homelab
sudo chmod 644 /etc/cron.d/backup-homelab
systemctl is-active cron              # debe responder: active

d) Verificar al día siguiente:

bash
sudo tail -20 /var/log/backup-homelab.log
sudo restic -r /mnt/backup/restic --password-file /root/.restic-pas---

Limitaciones (lo que esta copia NO resuelve)
No es una copia fuera de casa. La USB está conectada al mismo equipo: un robo, un incendio o una sobretensión se llevarían ambos. Para cumplir la regla 3-2-1, falta una copia en otro lugar (por ejemplo, almacenamiento en la nube cifrado).
Las memorias USB se desgastan antes que los discos si escriben cada noche. Hay que comprobar de vez en cuando que la restauración sigue funcionando.
La contraseña de restic está guardada en el servidor para poder automatizar. Quien controle el equipo tendría acceso a ella, por eso se guarda además en un gestor de contraseñas.
El servidor debe estar encendido a las 3:30 y con la USB conectada. Si no, el script se cancela y lo anota en el registro.

Paso 15. Nube personal con Nextcloud (Docker)

Qué es: Nextcloud es una alternativa propia a Google Drive o Dropbox. Los archivos viven en mi servidor, y los puedo sincronizar con el PC y el móvil, compartir y recuperar desde la papelera.

Cómo se instala: con Docker Compose, igual que Pi-hole. Usa dos contenedores: la aplicación y una base de datos PostgreSQL (donde se guardan usuarios, carpetas y metadatos).

Crear la carpeta y el archivo de configuración:
bash
   mkdir -p ~/nextcloud && cd ~/nextcloud
   nano docker-compose.yml
Contenido (cambiar las dos contraseñas por otras largas, solo con letras y números, y guardarlas en un gestor de contraseñas; la de la base de datos debe ser idéntica en las dos líneas donde aparece):
yaml
   services:
     db:
       image: postgres:16-alpine
       container_name: nextcloud-db
       restart: unless-stopped
       environment:
         POSTGRES_DB: nextcloud
         POSTGRES_USER: nextcloud
         POSTGRES_PASSWORD: "CLAVE_BASE_DATOS"
       volumes:
         - ./db:/var/lib/postgresql/data

     app:
       image: nextcloud:latest
       container_name: nextcloud
       restart: unless-stopped
       depends_on:
         - db
       ports:
         - "8081:80"
       environment:
         POSTGRES_HOST: db
         POSTGRES_DB: nextcloud
         POSTGRES_USER: nextcloud
         POSTGRES_PASSWORD: "CLAVE_BASE_DATOS"
         NEXTCLOUD_ADMIN_USER: admin
         NEXTCLOUD_ADMIN_PASSWORD: "CLAVE_ADMIN"
         NEXTCLOUD_TRUSTED_DOMAINS: "IP-LOCAL-DEL-SERVIDOR IP-TAILSCALE-DEL-SERVIDOR localhost"
       volumes:
         - ./nextcloud:/var/www/html
Arrancar y comprobar:
bash
   docker compose up -d
   docker ps
   docker logs nextcloud --tail 20

La instalación inicial termina cuando el registro muestra Nextcloud was successfully installed y resuming normal operations. 4. Abrir http://<IP-DEL-SERVIDOR>:8081 e iniciar sesión con admin.

Buenas prácticas aplicadas

Crear una cuenta normal para el uso diario (menú de avatar → Cuentas → Nueva cuenta) y usar admin solo para administrar. Si una cuenta de uso diario se ve comprometida, no tiene control total.
Revisar los avisos en Administración → Resumen y corregir los que importan:
bash
  # Ventana de mantenimiento: tareas pesadas de madrugada (la hora es UTC)
  docker exec -u www-data nextcloud php occ config:system:set maintenance_window_start --type=integer --value=4
  # Migraciones pendientes de tipos de archivo
  docker exec -u www-data nextcloud php occ maintenance:repair --include-expensive

Ojo de seguridad: Docker publica el puerto 8081 saltándose las reglas de UFW. Cualquier equipo de mi red local puede llegar a él, así que no se abre ese puerto en el router. El acceso desde fuera de casa es solo por Tailscale.

Paso 16. HTTPS con Tailscale (sin abrir puertos)

Por qué: Nextcloud avisa de que se accede por HTTP, y algunas funciones del navegador y las apps móviles lo exigen. Tailscale puede dar un certificado válido y publicar el servicio solo dentro de mi red privada.

En el panel de administración de Tailscale (sección DNS): comprobar que MagicDNS está activo y pulsar Enable HTTPS.
En el servidor:
bash
   sudo tailscale serve --bg 8081
   tailscale serve status

Muestra una dirección del tipo https://homelab01.tailNNNN.ts.net, marcada como tailnet only. 3. Enseñar a Nextcloud esa dirección y que está detrás de HTTPS:

bash
   docker exec -u www-data nextcloud php occ config:system:set trusted_domains 4 --value=homelab01.tailNNNN.ts.net
   docker exec -u www-data nextcloud php occ config:system:set overwriteprotocol --value=https
   docker exec -u www-data nextcloud php occ config:system:set overwrite.cli.url --value=https://homelab01.tailNNNN.ts.net
   docker exec -u www-data nextcloud php occ config:system:set trusted_proxies 0 --value=172.16.0.0/12
Desde ese momento se usa siempre la dirección HTTPS. Si algo sale mal: docker exec -u www-data nextcloud php occ config:system:delete overwriteprotocol.

Qué se usa y qué no: tailscale serve publica solo para mis dispositivos. tailscale funnel lo publicaría en internet, y no lo uso.

Aviso que se queda: Nextcloud sigue recomendando la cabecera HSTS. Se añade en el servidor web del contenedor, y al ir todo por una VPN cifrada aporta poco, así que decidí dejarlo.

Paso 17. Sincronizar archivos desde Windows
Instalar el cliente de escritorio desde la página oficial (nextcloud.com/install, sección de clientes; no el paquete del servidor).
Con Tailscale activo, iniciar sesión con la dirección HTTPS y la cuenta normal.
Elegir la carpeta de sincronización en un disco con mucho espacio libre, no en el disco del sistema, y marcar Sincronizar todo.
Probar con una carpeta pequeña antes de subir todo, y comprobar que aparece en la web.
Subir el resto copiando, no moviendo, por tandas. Hay que renombrar las carpetas de nombre larguísimo para que la ruta no supere unos 260 caracteres de Windows.
Comprobar al terminar que coinciden el número de archivos y el tamaño.

Concepto clave: sincronizar no es hacer una copia de seguridad. Si borro un archivo en el PC, se borra también en la nube. La protección real es la copia del paso siguiente.

Paso 18. Copias de seguridad de Nextcloud (disco externo)

Qué copio y cómo: los archivos de Nextcloud, el docker-compose.yml y un volcado de la base de datos. No se copian los archivos de la base de datos mientras funciona, porque la copia saldría corrupta. Mientras dura la copia, Nextcloud pasa a modo mantenimiento unos segundos.

Destino: un disco duro externo que ya tenía mis archivos. No se formatea. Las copias van en una carpeta aparte dentro de él.

Identificar el disco sin equivocarme:
bash
   lsblk -o NAME,SIZE,TYPE,FSTYPE,LABEL,TRAN,MODEL
Ver el espacio libre de sus particiones montándolas en solo lectura:
bash
   sudo mkdir -p /mnt/disco-1 /mnt/disco-2
   sudo mount -o ro /dev/sdX1 /mnt/disco-1
   df -h /mnt/disco-1

Elegí la partición con más espacio libre. sdX1 es un ejemplo, y el nombre real depende de cada equipo. 3. Montarlo con permiso de escritura y crear la carpeta de copias:

bash
   sudo umount /mnt/disco-1 /mnt/disco-2
   sudo mkdir -p /mnt/backup-externo
   sudo mount -o uid=$(id -u),gid=$(id -g) /dev/sdX1 /mnt/backup-externo
   mkdir /mnt/backup-externo/restic-nextcloud
Crear el repositorio cifrado (con una contraseña distinta a la de las otras copias, guardada antes en un gestor de contraseñas) y guardarla en un archivo que solo lee el administrador:
bash
   restic init --repo /mnt/backup-externo/restic-nextcloud
   sudo nano /root/.restic-password-nextcloud
   sudo chmod 600 /root/.restic-password-nextcloud
Script /usr/local/bin/backup-nextcloud.sh:
bash
   #!/bin/bash
   set -euo pipefail
   export RESTIC_REPOSITORY=/mnt/backup-externo/restic-nextcloud
   export RESTIC_PASSWORD_FILE=/root/.restic-password-nextcloud
   NC_DIR=/home/usuario/nextcloud
   DUMP_DIR=/var/backups/nextcloud-db

   if ! mountpoint -q /mnt/backup-externo; then
     echo "$(date): disco externo no montado, copia cancelada"
     exit 1
   fi

   mkdir -p "$DUMP_DIR"
   chmod 700 "$DUMP_DIR"

   docker exec -u www-data nextcloud php occ maintenance:mode --on
   trap 'docker exec -u www-data nextcloud php occ maintenance:mode --off' EXIT

   docker exec nextcloud-db pg_dump -U nextcloud nextcloud > "$DUMP_DIR/nextcloud.sql"
   restic backup "$NC_DIR/nextcloud" "$NC_DIR/docker-compose.yml" "$DUMP_DIR"

   docker exec -u www-data nextcloud php occ maintenance:mode --off
   trap - EXIT

   restic forget --keep-daily 7 --keep-weekly 4 --keep-monthly 6 --prune

El trap garantiza que Nextcloud sale del modo mantenimiento incluso si la copia falla. 6. Probar a mano y probar la restauración del volcado:

bash
   sudo chmod +x /usr/local/bin/backup-nextcloud.sh
   sudo /usr/local/bin/backup-nextcloud.sh
   sudo restic -r /mnt/backup-externo/restic-nextcloud --password-file /root/.restic-password-nextcloud restore latest --target /tmp/prueba-nc --include /var/backups/nextcloud-db
   sudo head -5 /tmp/prueba-nc/var/backups/nextcloud-db/nextcloud.sql
   sudo rm -r /tmp/prueba-nc

El archivo debe empezar con PostgreSQL database dump. Una línea \restrict ... justo después es normal en las versiones recientes de PostgreSQL. 7. Programar a las 4:00 (después de la copia de las 3:30) y dejar el montaje fijo por UUID:

bash
   echo "0 4 * * * root /usr/local/bin/backup-nextcloud.sh >> /var/log/backup-nextcloud.log 2>&1" | sudo tee /etc/cron.d/backup-nextcloud
   sudo chmod 644 /etc/cron.d/backup-nextcloud

   sudo cp /etc/fstab ~/fstab.bak2
   sudo -v
   UUID_EXT=$(sudo blkid -s UUID -o value /dev/sdX1); echo $UUID_EXT
   echo "UUID=$UUID_EXT /mnt/backup-externo ntfs-3g uid=$(id -u),gid=$(id -g),nofail,x-systemd.device-timeout=10 0 0" | sudo tee -a /etc/fstab
   sudo umount /mnt/backup-externo && sudo mount -a && mountpoint /mnt/backup-externo
   sudo systemctl daemon-reload
Prueba de reinicio: sudo reboot, y comprobar que los dos discos se montan solos, que los contenedores siguen en marcha (docker ps) y que la dirección HTTPS sigue publicada (tailscale serve status).
Verificar al día siguiente: sudo tail -20 /var/log/backup-nextcloud.log debe terminar con snapshot ... saved.

Resultado: dos copias automáticas cifradas y con versiones.

Hora	Qué copia	Destino
3:30	Pi-hole, /etc, .ssh	Memoria USB
4:00	Nextcloud: archivos y base de datos	Disco externo

## 5. Problemas que encontré y cómo los resolví

| Problema | Causa | Solución |
|---|---|---|
| Al instalar, la red mostraba `autoconfiguration failed` | No llegaba la IP del router por el cable | Revisar el cable, esperar unos segundos y reintentar el DHCP; se solucionó al reconectar |
| Solo 100 GB asignados a la raíz `/` | El instalador usa un valor por defecto con LVM | Editar el volumen lógico y ampliarlo al máximo |
| `netplan try` volvió a la configuración anterior | No pulsé Enter a tiempo | Repetirlo y confirmar con Enter antes del límite; es el comportamiento de seguridad esperado |
| No pude entrar al panel del router (`ERR_CONNECTION_REFUSED`) | El router no permitía acceso al panel | Fijar la IP en el propio servidor con netplan |
| Aviso de systemd al recargar SSH | Archivos del servicio cambiados por actualizaciones | `sudo systemctl daemon-reload` y volver a recargar; no era un error |
| Probé SSH desde dentro del propio servidor | Confusión sobre qué ventana usar | Hacer las pruebas siempre desde otro equipo (el prompt de Windows empieza por `PS C:\`) |
| Comando pegado con caracteres raros (`event not found`) | Pegado mal formado en la terminal | No hizo daño; volver a pegar el comando bien |
| `ssh-agent` en Windows: *Acceso denegado* | Faltaban permisos de administrador | Lo dejé pendiente, es solo una comodidad (recordar la frase de la llave) |
|Aviso at least one source file could not be read | La primera copia se hizo sin permisos de administrador | Repetirla con sudo |
| sudo rechazaba la contraseña varias veces | El comando usaba sudo dos veces a la vez y los dos avisos se estorbaban | Usar un solo sudo por comando y guardar el UUID en una variable antes |
| El bloque pegado con EOF se quedó esperando (>) | El EOF final tenía espacios delante y no se reconoció como cierre | Pegar el bloque con EOF en la primera columna, o usar printf |
| cat daba Permission denied tras restaurar | Con sudo, los archivos restaurados quedan a nombre de root | Leerlos con sudo cat |
| Aviso rojo "Acceso HTTPS y URLs" en el Resumen de Nextcloud | Se accedía por HTTP | Publicar con `tailscale serve` y configurar `overwriteprotocol` y los dominios de confianza |
| Error en el registro de Nextcloud (`127.0.0.1:25`) | No hay servidor de correo y falló el mensaje de bienvenida de la cuenta | Es inofensivo; solo impide restablecer contraseñas por correo, así que se guardan en un gestor de contraseñas |
| `mountpoint` daba `bad usage` | Se pasaron dos rutas, y acepta una por comando | Ejecutarlo una vez por cada ruta |
| `mount` daba `Device or resource busy` | El disco había quedado montado del intento anterior | Comprobar con `findmnt`, desmontar y volver a montar |
| `No such file or directory` al escribir una ruta | Se escribió la ruta sola, sin ningún comando delante | Anteponer el comando (`mkdir`, `cd`...) |
| El cliente de sincronización mostraba poco espacio libre | La carpeta por defecto estaba en el disco del sistema | Elegir una carpeta en otro disco con más espacio |
| Archivos "solo en línea" que no se podían copiar | Estaban solo en la nube de otro servicio, con su almacenamiento lleno | Descargarlos por tandas ("mantener siempre en este dispositivo"), copiarlos y verificar |

---

## 6. Qué aprendí (conceptos)

- **Linux básico:** `sudo`, `apt`, `nano`, `systemctl`, permisos de archivos.
- **Redes:** IP, DHCP, puertos, DNS, puerta de enlace, `ping` y `nslookup`.
- **Seguridad:** firewall, autenticación por llave, mínimo privilegio, riesgo de exponer servicios y la importancia de las copias de seguridad.
- **Contenedores:** qué es Docker y cómo se despliega un servicio con Compose.
- **VPN:** acceso remoto sin abrir puertos.
- **Buenas prácticas:** probar los cambios con una vía de escape (`netplan try`, sesión SSH de respaldo) y documentar.
- **copias cifradas con restic** política de retención, montaje automático con fstab y tareas programadas con cron.
- **NextCloud** Desplegar una aplicación con varios contenedores, HTTPS con certificados de Tailscale, diferencia entre sincronizar y respaldar, volcados de bases de datos, retención de copias y tareas programadas.
---

## 7. Observaciones del equipo

- **Batería:** conserva alrededor del 58 % de su capacidad original, lo que daría unas pocas horas en un apagón corto (estimación). Si se hincha, hay que retirarla.
- **Apagones:** el router también se apaga, así que sin luz no habría acceso remoto aunque el Mac siga encendido.
- **Temperatura:** entre 55 y 65 °C con la tapa cerrada. Hay que dejarlo ventilado y enchufado.
- **Consumo:** muy bajo en reposo (estimación de 8 a 15 W, equivalente a unos 7 a 11 kWh al mes).

---

## 8. Pendiente / próximos pasos

- [x] Comprobar e instalar **fail2ban** (bloqueo automático de IPs que fallan al entrar por SSH).
- [x] **Copias de seguridad** con `restic` a un disco externo y **probar la restauración**.
- [ ] Configurar el **apagado ordenado** con poca batería.
- [x] Montar una **nube personal** (por ejemplo Nextcloud, con Docker), accediendo solo por Tailscale.
- [ ] Opcional: `ssh-agent` en Windows para no escribir la frase en cada conexión.
- [ ] Seguir el plan de estudio: Linux, redes, Docker, Python para seguridad y fundamentos de ciberseguridad.

---

## 9. Recursos que uso para aprender

- *The Linux Command Line* (William Shotts) y OverTheWire: Bandit.
- Cisco Networking Academy (Networking Basics).
- Centro de aprendizaje de Cloudflare (DNS y VPN).
- Documentación oficial de Docker y de Tailscale.
- Professor Messer (Security+) y TryHackMe.
- INCIBE, para ciberseguridad y normas como ISO 27001.

---

*Proyecto de aprendizaje personal. Hecho con un MacBook de 2012 que merecía una segunda vida.*
