[← Volver](../README.md)

# Entrega X: Raspberry Pi como router con Bluetooth Tethering

![Raspberry Pi como router](media/router.jpg)

## Integrantes del Grupo 5

- Lorenzo Palacios
- Benjamin Fasioli
- Octavio Paez
- Ramiro Cardo
- Sebastian Weiss

## Objetivo

Configurar una Raspberry Pi 3 B+ para que funcione como router, obteniendo su conexión a Internet desde un celular mediante Bluetooth tethering (PAN) y compartiéndola con los dispositivos de la red local a través de sus interfaces WiFi (como punto de acceso) y Ethernet (conectada a un switch).

La Raspberry Pi ya contaba con un servidor DHCP configurado, por lo que el trabajo se centró en la conexión de salida (uplink), el reenvío de paquetes, la traducción de direcciones (NAT) y el punto de acceso inalámbrico.

## Entorno de trabajo

| Elemento | Información |
|---|---|
| Modelo de Raspberry Pi | **Raspberry Pi 3 B+** |
| Sistema operativo de la Raspberry Pi | **Raspberry Pi OS Lite** |
| Conexión a Internet (uplink) | **Bluetooth tethering desde un celular** (interfaz `bnep0`) |
| Interfaz Ethernet (hacia el switch) | `eth0` — `192.168.5.1/24` |
| Interfaz WiFi (punto de acceso) | `wlan0` — `192.168.6.1/24` |
| Nombre del usuario de la Raspberry Pi | **grupo-05** |
| Servidor DHCP | **isc-dhcp-server** (ya configurado) |
| Punto de acceso WiFi | **hostapd** |
| Red WiFi (SSID) | `MiRouterPi` |

> **Nota:** las direcciones IP, el nombre de la red WiFi y la dirección MAC del celular pueden adaptarse a la configuración real de cada grupo.

## Materiales y recursos utilizados

- Raspberry Pi 3 B+ con tarjeta microSD y Raspberry Pi OS instalado.
- Fuente de alimentación para la Raspberry Pi.
- Celular con plan de datos y función de compartir conexión por Bluetooth.
- Switch y cable Ethernet.
- Computadora o dispositivo cliente conectado por Ethernet al switch.
- Dispositivo cliente con WiFi (celular o notebook).
- Monitor, teclado y cable HDMI (o acceso por SSH) para la configuración.
- Paquetes `bluez`, `bluez-tools`, `hostapd`, `isc-dhcp-server` e `iptables-persistent`.

## Definiciones

**Router:** es un dispositivo que conecta redes distintas y reenvía paquetes entre ellas. En esta actividad, la Raspberry Pi conecta la red local (WiFi y Ethernet) con Internet.

**Bluetooth tethering:** es la función de un celular que permite compartir su conexión de datos móviles con otros dispositivos mediante Bluetooth.

**PAN (Personal Area Network):** es el perfil de Bluetooth que permite crear una red de área personal entre dispositivos. El celular actúa como **NAP** (*Network Access Point*), es decir, el que ofrece el acceso a la red, y la Raspberry Pi como **PANU** (*PAN User*), es decir, el que la utiliza.

**`bnep0`:** es la interfaz de red virtual que aparece en Linux cuando se establece una conexión PAN por Bluetooth. Funciona como el "cable" por el que llega Internet a la Raspberry Pi.

**Uplink:** es la conexión que une a un router con la red externa (Internet). En este caso, `bnep0`.

**Reenvío de paquetes (IP forwarding):** es la capacidad del kernel de Linux de pasar paquetes de una interfaz de red a otra. Sin esta opción, el equipo descarta los paquetes que no están dirigidos a él y no puede actuar como router.

**NAT (Network Address Translation) / MASQUERADE:** es una técnica que reemplaza la dirección IP privada de origen de los paquetes por la dirección IP de la interfaz de salida. Permite que varios dispositivos con IP privadas compartan una única conexión a Internet.

**iptables:** es la herramienta de Linux que permite definir reglas de filtrado y traducción de paquetes (firewall y NAT).

**hostapd:** es un software que permite que una placa WiFi funcione como punto de acceso, es decir, que otros dispositivos se conecten a ella de forma inalámbrica.

**Punto de acceso (AP):** es un dispositivo que crea una red WiFi y permite que los equipos inalámbricos se conecten a la red cableada o a Internet.

**DHCP (Dynamic Host Configuration Protocol):** es el protocolo que asigna automáticamente a los dispositivos una dirección IP, la puerta de enlace y los servidores DNS.

**isc-dhcp-server:** es la implementación del servidor DHCP desarrollada por el Internet Systems Consortium (ISC). Aunque se encuentra discontinuada (deprecada) a favor de Kea DHCP, sigue disponible en los repositorios y es ampliamente utilizada.

**Switch:** es un dispositivo que conecta varios equipos de una red local por cable Ethernet y reenvía las tramas entre ellos.

**DNS (Domain Name System):** es el sistema que traduce nombres de dominio (como `google.com`) a direcciones IP.

**Interfaz de red:** es el punto de conexión de un equipo a una red. En la Raspberry Pi: `eth0` (Ethernet), `wlan0` (WiFi) y `bnep0` (Bluetooth PAN).

## Procedimiento

### 1. Verificación del estado inicial de las interfaces

Antes de comenzar, comprobamos las interfaces de red de la Raspberry Pi y sus direcciones IP:

```bash
ip a
```

El comando `ip a` (abreviatura de `ip address`) muestra todas las interfaces de red y las direcciones asignadas. En este punto deben aparecer `eth0` y `wlan0`; todavía no existe `bnep0`.

![Estado inicial de las interfaces](media/paso1.png)

### 2. Instalación de los paquetes necesarios

Instalamos en un solo comando todos los paquetes que no estaban presentes en el sistema:

```bash
sudo apt install bluez-tools hostapd iptables-persistent
```

El comando utiliza `sudo` para obtener permisos de administrador, `apt install` para instalar paquetes y, a continuación, los paquetes que queremos instalar:

- `bluez-tools`: incluye el comando `bt-network`, necesario para establecer la conexión PAN con el celular. Depende de `bluez`, la pila Bluetooth oficial de Linux, que se instala automáticamente si no estaba presente.
- `hostapd`: permite que la interfaz `wlan0` funcione como punto de acceso WiFi.
- `iptables-persistent`: permite guardar las reglas de `iptables` para que se restauren automáticamente en cada reinicio.

Durante la instalación de `iptables-persistent` el sistema pregunta si se desean guardar las reglas actuales de IPv4 e IPv6. En este momento se puede responder que sí, ya que más adelante se volverán a guardar con las reglas definitivas.

El servidor DHCP (`isc-dhcp-server`) ya estaba instalado y configurado, por lo que no se incluye en este comando.

![Instalación de los paquetes](media/paso2.png)

### 3. Emparejamiento del celular con la Raspberry Pi

Antes de este paso, en el celular activamos **Compartir conexión por Bluetooth** (*Bluetooth tethering*) desde los ajustes de conexión.

Luego ingresamos a la herramienta de control de Bluetooth de la Raspberry Pi:

```bash
bluetoothctl
```

Dentro de la consola interactiva de `bluetoothctl` ejecutamos los siguientes comandos:

```text
power on
agent on
scan on
```

- `power on`: enciende el adaptador Bluetooth.
- `agent on`: activa el agente que gestiona el emparejamiento (confirmación de códigos).
- `scan on`: busca dispositivos cercanos. Allí identificamos la dirección MAC del celular.

Una vez detectado el celular, lo emparejamos y lo marcamos como confiable:

```text
pair XX:XX:XX:XX:XX:XX
trust XX:XX:XX:XX:XX:XX
exit
```

- `pair`: realiza el emparejamiento. Hay que aceptar el código de confirmación en ambos dispositivos.
- `trust`: marca el dispositivo como confiable para que se pueda reconectar sin volver a pedir autorización.
- `exit`: sale de la consola de `bluetoothctl`.

`XX:XX:XX:XX:XX:XX` debe reemplazarse por la dirección MAC real del celular.

![Emparejamiento con bluetoothctl](media/paso3.png)

### 4. Establecimiento de la conexión PAN

Con el celular emparejado, creamos la conexión de red por Bluetooth:

```bash
bt-network -c XX:XX:XX:XX:XX:XX nap
```

El comando utiliza `bt-network` para gestionar redes Bluetooth, la opción `-c` para conectarse al dispositivo indicado por su MAC y `nap` para indicar que el celular actúa como punto de acceso a la red.

Este comando queda ejecutándose mientras la conexión permanezca activa, por lo que los pasos siguientes se realizan desde otra terminal (o a través de una segunda sesión SSH). Al establecerse la conexión aparece la interfaz `bnep0`.

![Conexión con bt-network](media/paso4.png)

### 5. Obtención de una dirección IP en bnep0

La interfaz `bnep0` no tiene IP hasta que el celular se la asigne. Para solicitarla ejecutamos:

```bash
sudo dhclient bnep0
```

`dhclient` es un cliente DHCP: le pide una dirección IP, una puerta de enlace y un servidor DNS al DHCP del celular. De esta forma la Raspberry Pi obtiene datos de conexión para salir a Internet.

![Solicitud de IP con dhclient](media/paso5.png)

### 6. Verificación de la dirección IP de bnep0

Comprobamos que `bnep0` haya recibido una dirección:

```bash
ip a show bnep0
```

El comando muestra únicamente la información de la interfaz `bnep0`. Debe aparecer una línea `inet` con la IP asignada por el celular (generalmente en el rango `192.168.44.x` en Android).

![Verificación de bnep0](media/paso6.png)

### 7. Prueba de conectividad a Internet por Bluetooth

Verificamos que la Raspberry Pi tenga salida a Internet a través de `bnep0`:

```bash
ping -I bnep0 8.8.8.8
```

El comando `ping` envía paquetes de prueba, la opción `-I bnep0` fuerza a que salgan por esa interfaz y `8.8.8.8` es el servidor DNS público de Google, utilizado como destino de prueba. Si se reciben respuestas, la Raspberry Pi tiene Internet por Bluetooth.

![Ping por bnep0](media/paso7.png)

### 8. Habilitación del reenvío de paquetes (parte 1: archivo de configuración)

Para que la Raspberry Pi pueda pasar tráfico entre sus interfaces, creamos un archivo de configuración con la opción de reenvío activada:

```bash
echo "net.ipv4.ip_forward=1" | sudo tee /etc/sysctl.d/99-forward.conf
```

El comando `echo` genera el texto `net.ipv4.ip_forward=1`, la barra `|` lo envía a `tee`, que lo escribe con permisos de administrador en `/etc/sysctl.d/99-forward.conf`. Al estar en este archivo, el cambio se mantiene después de reiniciar.

![Archivo de reenvío de paquetes](media/paso8.png)

### 9. Habilitación del reenvío de paquetes (parte 2: aplicación)

Aplicamos la configuración sin necesidad de reiniciar:

```bash
sudo sysctl --system
```

El comando `sysctl --system` vuelve a cargar todos los archivos de configuración del kernel, incluido el que acabamos de crear. Para comprobarlo, se puede ejecutar `cat /proc/sys/net/ipv4/ip_forward`, que debe devolver `1`.

![Aplicación de sysctl](media/paso9.png)

### 10. Regla de NAT (MASQUERADE)

Configuramos el NAT para que los dispositivos de la red local salgan a Internet con la IP de `bnep0`:

```bash
sudo iptables -t nat -A POSTROUTING -o bnep0 -j MASQUERADE
```

- `-t nat`: trabaja sobre la tabla de traducción de direcciones.
- `-A POSTROUTING`: agrega la regla en la cadena que se aplica justo antes de que el paquete salga.
- `-o bnep0`: indica que la regla se aplica a los paquetes que salen por `bnep0`.
- `-j MASQUERADE`: reemplaza la IP de origen por la de la interfaz de salida.

Sin esta regla, el celular recibiría paquetes con IP privadas de nuestra red local que no sabe cómo responder.

![Regla de MASQUERADE](media/paso10.png)

### 11. Regla de reenvío para las respuestas de Internet

Permitimos que las respuestas que llegan desde Internet vuelvan hacia la red local:

```bash
sudo iptables -A FORWARD -i bnep0 -m conntrack --ctstate RELATED,ESTABLISHED -j ACCEPT
```

- `-A FORWARD`: agrega la regla en la cadena de paquetes que atraviesan el equipo.
- `-i bnep0`: aplica a los paquetes que ingresan por `bnep0`.
- `-m conntrack --ctstate RELATED,ESTABLISHED`: solo acepta paquetes que pertenecen a conexiones ya iniciadas desde la red local.
- `-j ACCEPT`: acepta los paquetes.

De esta manera, Internet no puede iniciar conexiones hacia la red local, pero sí responder a las que se originan dentro de ella.

![Regla de FORWARD para bnep0](media/paso11.png)

### 12. Regla de reenvío desde la red WiFi hacia Internet

Permitimos que el tráfico de los dispositivos conectados por WiFi salga hacia Internet:

```bash
sudo iptables -A FORWARD -i wlan0 -o bnep0 -j ACCEPT
```

La regla acepta los paquetes que entran por `wlan0` (clientes WiFi) y salen por `bnep0` (Bluetooth).

![Regla de FORWARD para wlan0](media/paso12.png)

### 13. Regla de reenvío desde la red Ethernet hacia Internet

Permitimos que el tráfico de los dispositivos conectados al switch salga hacia Internet:

```bash
sudo iptables -A FORWARD -i eth0 -o bnep0 -j ACCEPT
```

La regla acepta los paquetes que entran por `eth0` (red cableada) y salen por `bnep0`.

![Regla de FORWARD para eth0](media/paso13.png)

### 14. Guardado de las reglas de iptables

Guardamos las reglas para que no se pierdan al reiniciar la Raspberry Pi:

```bash
sudo netfilter-persistent save
```

El comando `netfilter-persistent save` almacena las reglas actuales en `/etc/iptables/rules.v4`, desde donde se cargan automáticamente al iniciar el sistema.

![Guardado de reglas](media/paso14.png)

### 15. Configuración del punto de acceso WiFi (hostapd.conf)

Creamos el archivo de configuración del punto de acceso:

```bash
sudo nano /etc/hostapd/hostapd.conf
```

Dentro del archivo agregamos:

```conf
interface=wlan0
driver=nl80211
ssid=MiRouterPi
country_code=AR
hw_mode=a
channel=36
ieee80211n=1
wmm_enabled=1
wpa=2
wpa_passphrase=unaClaveSegura
wpa_key_mgmt=WPA-PSK
rsn_pairwise=CCMP
```

Estas opciones definen el comportamiento del punto de acceso:

- `interface=wlan0`: interfaz que funcionará como punto de acceso.
- `driver=nl80211`: controlador estándar de Linux para placas WiFi.
- `ssid=MiRouterPi`: nombre de la red WiFi que verán los clientes.
- `country_code=AR`: país para aplicar las regulaciones de canales y potencia (Argentina).
- `hw_mode=a` y `channel=36`: usa la banda de 5 GHz, canal 36.
- `ieee80211n=1` y `wmm_enabled=1`: habilitan el estándar 802.11n y la calidad de servicio WMM.
- `wpa=2`: usa seguridad WPA2.
- `wpa_passphrase=unaClaveSegura`: contraseña de la red WiFi (reemplazar por una propia, de 8 caracteres o más).
- `wpa_key_mgmt=WPA-PSK`: autenticación mediante clave precompartida.
- `rsn_pairwise=CCMP`: cifrado AES/CCMP.

Se eligió la banda de **5 GHz** a propósito: en la Raspberry Pi 3 B+ el WiFi y el Bluetooth comparten chip y la banda de 2,4 GHz, por lo que usarlos a la vez en esa banda genera interferencias y reduce el rendimiento. Si algún dispositivo cliente no soporta 5 GHz, se puede usar `hw_mode=g` con `channel=1` o `channel=6`.

![Configuración de hostapd](media/paso15.png)

### 16. Indicar a hostapd cuál es su archivo de configuración

Editamos el archivo que utiliza el servicio para saber qué configuración cargar:

```bash
sudo nano /etc/default/hostapd
```

Buscamos la línea `DAEMON_CONF`, la descomentamos (quitamos el `#`) y le indicamos la ruta del archivo creado en el paso anterior:

```conf
DAEMON_CONF="/etc/hostapd/hostapd.conf"
```

![Archivo /etc/default/hostapd](media/paso16.png)

### 17. Desbloqueo de la radio WiFi

Por defecto, Raspberry Pi OS puede dejar la radio WiFi bloqueada hasta que se configura el país. Para desbloquearla ejecutamos:

```bash
sudo rfkill unblock wlan
```

`rfkill` administra el bloqueo de dispositivos inalámbricos y `unblock wlan` libera la placa WiFi para que `hostapd` pueda utilizarla.

![Desbloqueo de la radio WiFi](media/paso17.png)

### 18. Asignación de una IP estática a wlan0

Un punto de acceso necesita una dirección IP fija en su interfaz, ya que esa dirección será la puerta de enlace de los clientes WiFi. Editamos el archivo de configuración de red:

```bash
sudo nano /etc/dhcpcd.conf
```

Al final del archivo agregamos:

```conf
interface wlan0
    static ip_address=192.168.6.1/24
    nohook wpa_supplicant
```

- `interface wlan0`: indica la interfaz que se configura.
- `static ip_address=192.168.6.1/24`: asigna la IP fija `192.168.6.1` con máscara `255.255.255.0`.
- `nohook wpa_supplicant`: evita que `wpa_supplicant` tome control de la interfaz, ya que ahora funciona como punto de acceso y no como cliente.

> **Nota:** `eth0` ya tiene su IP fija (`192.168.5.1/24`) configurada en el práctico anterior. Si el sistema utiliza NetworkManager en lugar de `dhcpcd`, la IP de `wlan0` se configura con `nmcli` en su lugar.

![Configuración de dhcpcd](media/paso18.png)

### 19. Verificación de las subredes del servidor DHCP (dhcpd.conf)

Como el servidor DHCP ya estaba configurado, solo verificamos que contemple las dos redes (Ethernet y WiFi) y que entregue un servidor DNS válido. Para ello abrimos su archivo principal de configuración:

```bash
sudo nano /etc/dhcp/dhcpd.conf
```

El archivo debe contener una sección `subnet` por cada red:

```conf
default-lease-time 600;
max-lease-time 7200;
authoritative;

# Red Ethernet (switch)
subnet 192.168.5.0 netmask 255.255.255.0 {
    range 192.168.5.10 192.168.5.100;
    option routers 192.168.5.1;
    option domain-name-servers 8.8.8.8, 1.1.1.1;
}

# Red WiFi (punto de acceso)
subnet 192.168.6.0 netmask 255.255.255.0 {
    range 192.168.6.10 192.168.6.100;
    option routers 192.168.6.1;
    option domain-name-servers 8.8.8.8, 1.1.1.1;
}
```

- `default-lease-time` y `max-lease-time`: tiempo (en segundos) que un cliente conserva su IP antes de renovarla y tiempo máximo de la concesión.
- `authoritative`: indica que este es el servidor DHCP oficial de la red.
- `subnet ... netmask ...`: define cada red a la que el servidor entrega direcciones.
- `range`: rango de direcciones IP que se pueden asignar a los clientes.
- `option routers`: puerta de enlace que se entrega a los clientes, es decir, la IP de la Raspberry Pi en cada red.
- `option domain-name-servers`: servidores DNS que se entregan a los clientes (Google y Cloudflare). Si los clientes obtienen IP pero no pueden navegar por nombre, casi siempre es un problema en esta opción.

> **Nota:** `isc-dhcp-server` se encuentra discontinuado (deprecado) a favor de Kea DHCP, pero sigue estando disponible en los repositorios y funciona correctamente para esta actividad.

![Configuración de dhcpd.conf](media/paso19.png)

### 20. Verificación de las interfaces del servidor DHCP (isc-dhcp-server)

Además de definir las subredes, hay que indicarle al servicio en qué interfaces debe escuchar las solicitudes de los clientes:

```bash
sudo nano /etc/default/isc-dhcp-server
```

Buscamos la línea `INTERFACESv4` y le indicamos las dos interfaces:

```conf
INTERFACESv4="eth0 wlan0"
```

`INTERFACESv4` define las interfaces en las que el servidor atiende solicitudes DHCP para IPv4. Cada interfaz debe tener una dirección IP dentro de la subred que le corresponde en `dhcpd.conf`; de lo contrario el servicio no inicia. Por eso fue necesario asignar antes una IP estática a `wlan0`.

![Configuración de isc-dhcp-server](media/paso20.png)

### 21. Inicio de los servicios

Habilitamos `hostapd` para que arranque con el sistema (el paquete lo instala "enmascarado", por lo que primero hay que desenmascararlo):

```bash
sudo systemctl unmask hostapd
```

```bash
sudo systemctl enable hostapd
```

Luego reiniciamos el punto de acceso y el servidor DHCP para aplicar la configuración. Es importante hacerlo en este orden: `hostapd` pone en funcionamiento `wlan0`, y `isc-dhcp-server` necesita que esa interfaz esté activa para poder escuchar en ella:

```bash
sudo systemctl restart hostapd
```

```bash
sudo systemctl restart isc-dhcp-server
```

- `unmask`: quita el bloqueo que impide iniciar el servicio.
- `enable`: hace que el servicio inicie automáticamente al encender la Raspberry Pi.
- `restart`: reinicia el servicio leyendo la nueva configuración.

Para verificar que ambos estén funcionando se puede usar `sudo systemctl status hostapd` y `sudo systemctl status isc-dhcp-server`; ambos deben figurar como `active (running)`.

![Inicio de los servicios](media/paso21.png)

### 22. Script de reconexión automática del Bluetooth

Si el celular se desconecta o se reinicia, la interfaz `bnep0` desaparece. Para automatizar la reconexión creamos un script:

```bash
sudo nano /usr/local/bin/bt-pan.sh
```

Con el siguiente contenido:

```bash
#!/bin/bash
MAC="XX:XX:XX:XX:XX:XX"

while true; do
    if ! ip link show bnep0 > /dev/null 2>&1; then
        bt-network -c "$MAC" nap &
        sleep 8
        dhclient bnep0
    fi
    sleep 10
done
```

El script revisa cada 10 segundos si existe la interfaz `bnep0`. Si no existe, vuelve a conectarse al celular con `bt-network` y solicita una IP con `dhclient`. Hay que reemplazar `XX:XX:XX:XX:XX:XX` por la MAC del celular.

Luego le damos permisos de ejecución:

```bash
sudo chmod +x /usr/local/bin/bt-pan.sh
```

![Script bt-pan.sh](media/paso22.png)

### 23. Servicio systemd para el script de reconexión

Para que el script se ejecute al iniciar la Raspberry Pi, creamos un servicio de systemd:

```bash
sudo nano /etc/systemd/system/bt-pan.service
```

Con el siguiente contenido:

```ini
[Unit]
Description=Conexion Bluetooth PAN automatica
After=bluetooth.target
Wants=bluetooth.target

[Service]
ExecStart=/usr/local/bin/bt-pan.sh
Restart=always

[Install]
WantedBy=multi-user.target
```

- `After=bluetooth.target`: inicia el servicio después de que el Bluetooth esté disponible.
- `ExecStart`: ruta del script que se ejecuta.
- `Restart=always`: si el script se detiene, systemd lo vuelve a iniciar.
- `WantedBy=multi-user.target`: hace que se inicie en el arranque normal del sistema.

![Servicio bt-pan.service](media/paso23.png)

### 24. Habilitación del servicio de reconexión

Recargamos systemd para que reconozca el nuevo servicio, lo habilitamos y lo iniciamos:

```bash
sudo systemctl daemon-reload
```

```bash
sudo systemctl enable --now bt-pan.service
```

- `daemon-reload`: vuelve a leer los archivos de servicio.
- `enable --now`: habilita el servicio para el arranque y lo inicia en el momento.

![Habilitación de bt-pan.service](media/paso24.png)

### 25. Prueba desde un cliente conectado por Ethernet

Conectamos una computadora al switch y comprobamos que obtuviera una IP del rango `192.168.5.x`, y que tuviera salida a Internet:

```bash
ping 8.8.8.8
```

```bash
ping google.com
```

El primer `ping` comprueba la conectividad a Internet por IP. El segundo comprueba, además, la resolución de nombres (DNS). Si el primero funciona y el segundo no, el problema está en la opción de DNS del servidor DHCP.

![Prueba desde el cliente Ethernet](media/paso25.png)

### 26. Prueba desde un cliente conectado por WiFi

Desde un celular o notebook buscamos la red `MiRouterPi`, nos conectamos con la contraseña configurada y comprobamos que se obtuviera una IP del rango `192.168.6.x` y que fuera posible navegar en Internet.

![Prueba desde el cliente WiFi](media/paso26.png)

## Comprobaciones realizadas

| Comprobación | Procedimiento o comando | Resultado obtenido |
|---|---|---|
| Instalación de los paquetes | `sudo apt install bluez-tools hostapd iptables-persistent` | Se instalaron los paquetes necesarios con un solo comando. |
| Emparejamiento del celular | `bluetoothctl` → `pair` y `trust` | El celular quedó emparejado y marcado como confiable. |
| Conexión Bluetooth PAN | `bt-network -c MAC nap` | Apareció la interfaz `bnep0`. |
| Dirección IP del uplink | `sudo dhclient bnep0` y `ip a show bnep0` | `bnep0` recibió una IP del celular. |
| Internet por Bluetooth | `ping -I bnep0 8.8.8.8` | La Raspberry Pi recibió respuestas. |
| Reenvío de paquetes | `/etc/sysctl.d/99-forward.conf` y `sudo sysctl --system` | Se habilitó `net.ipv4.ip_forward=1`. |
| NAT | `iptables -t nat -A POSTROUTING -o bnep0 -j MASQUERADE` | Se configuró la traducción de direcciones. |
| Reglas de reenvío | `iptables -A FORWARD ...` (3 reglas) | Se permitió el tráfico desde `eth0` y `wlan0` hacia `bnep0`. |
| Persistencia de reglas | `sudo netfilter-persistent save` | Las reglas se guardaron para el próximo arranque. |
| Punto de acceso WiFi | `/etc/hostapd/hostapd.conf` y `systemctl restart hostapd` | Se visualizó la red `MiRouterPi`. |
| Servidor DHCP | `/etc/dhcp/dhcpd.conf`, `/etc/default/isc-dhcp-server` y `systemctl restart isc-dhcp-server` | Los clientes recibieron IP en ambas redes. |
| Reconexión automática | `bt-pan.sh` y `bt-pan.service` | La conexión Bluetooth se restablece sola. |
| Cliente Ethernet | `ping 8.8.8.8` y `ping google.com` | El cliente tuvo acceso a Internet. |
| Cliente WiFi | Conexión a `MiRouterPi` y navegación | El cliente tuvo acceso a Internet. |

## Resultado

Se logró configurar la Raspberry Pi 3 B+ como un router que obtiene Internet a través de Bluetooth tethering desde un celular y lo comparte con los dispositivos de la red local por dos vías: Ethernet, mediante un switch, y WiFi, mediante un punto de acceso creado con `hostapd`.

Los clientes de ambas redes recibieron su configuración de red desde el servidor DHCP de la Raspberry Pi y pudieron navegar por Internet gracias al reenvío de paquetes y al NAT configurado con `iptables`.

Se observó que la velocidad de la conexión es baja (del orden de 1 a 2 Mbps), debido a que el perfil PAN de Bluetooth tiene un ancho de banda muy inferior al de WiFi o Ethernet. Esto es una limitación del medio de conexión y no un error de configuración.

## Conclusión

A través de esta actividad aprendimos a transformar una Raspberry Pi en un router completo, integrando varios servicios: una conexión de entrada por Bluetooth PAN, el reenvío de paquetes del kernel, la traducción de direcciones con NAT, un punto de acceso WiFi y un servidor DHCP.

Comprendimos que cada componente cumple un rol específico y que el funcionamiento del conjunto depende de todos ellos: sin reenvío de paquetes no hay enrutamiento, sin NAT los paquetes no vuelven, sin DHCP los clientes no reciben su configuración y sin la interfaz `bnep0` no hay Internet.

También identificamos las limitaciones del Bluetooth como medio de conexión, tanto en velocidad como en estabilidad, y la importancia de automatizar la reconexión para que el servicio sea confiable.

De esta manera, logramos cumplir con el objetivo de la actividad y ampliar los servicios de la Raspberry Pi dentro de la infraestructura de red local.
