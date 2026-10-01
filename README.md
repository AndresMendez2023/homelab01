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
| **Seguridad aplicada** | Firewall, SSH solo con llave (sin contraseña), sin login de root, VPN sin abrir puertos en el router |
| **Consumo estimado** | 8 a 15 W en reposo (estimación, no medido) |

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

---

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

---

## 6. Qué aprendí (conceptos)

- **Linux básico:** `sudo`, `apt`, `nano`, `systemctl`, permisos de archivos.
- **Redes:** IP, DHCP, puertos, DNS, puerta de enlace, `ping` y `nslookup`.
- **Seguridad:** firewall, autenticación por llave, mínimo privilegio, riesgo de exponer servicios y la importancia de las copias de seguridad.
- **Contenedores:** qué es Docker y cómo se despliega un servicio con Compose.
- **VPN:** acceso remoto sin abrir puertos.
- **Buenas prácticas:** probar los cambios con una vía de escape (`netplan try`, sesión SSH de respaldo) y documentar.

---

## 7. Observaciones del equipo

- **Batería:** conserva alrededor del 58 % de su capacidad original, lo que daría unas pocas horas en un apagón corto (estimación). Si se hincha, hay que retirarla.
- **Apagones:** el router también se apaga, así que sin luz no habría acceso remoto aunque el Mac siga encendido.
- **Temperatura:** entre 55 y 65 °C con la tapa cerrada. Hay que dejarlo ventilado y enchufado.
- **Consumo:** muy bajo en reposo (estimación de 8 a 15 W, equivalente a unos 7 a 11 kWh al mes).

---

## 8. Pendiente / próximos pasos

- [ ] Comprobar e instalar **fail2ban** (bloqueo automático de IPs que fallan al entrar por SSH).
- [ ] **Copias de seguridad** con `restic` a un disco externo y **probar la restauración**.
- [ ] Configurar el **apagado ordenado** con poca batería.
- [ ] Montar una **nube personal** (por ejemplo Nextcloud, con Docker), accediendo solo por Tailscale.
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
