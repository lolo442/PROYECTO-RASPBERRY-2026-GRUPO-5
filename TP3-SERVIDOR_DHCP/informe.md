[← Volver](../README.md)
# Entrega 2: Instalación y configuración de X11 mediante SSH

![Logo de X.Org](media/xorg-logo.jpg)

## Integrantes del Grupo 5

- Lorenzo Palacios
- Benjamin Fasioli
- Octavio Paez
- Ramiro Cardo
- Sebastian Weiss

## Objetivo

Configurar el reenvío de X11 mediante SSH en una Raspberry Pi para ejecutar aplicaciones gráficas de forma remota y visualizar sus ventanas en una computadora cliente a través de una conexión cifrada.

## Entorno de trabajo

Antes de entregar el informe, completar los siguientes datos con la información utilizada durante la práctica:

| Elemento | Información |
|---|---|
| Modelo de Raspberry Pi | **[Raspberry Pi b+]** |
| Sistema operativo de la Raspberry Pi | **[Raspberry Pi os Lite]** |
| Tipo de conexión de red | **[Ethernet]** |
| Nombre del usuario de la Raspberry Pi | **[grupo-05]** |
| Aplicación gráfica utilizada | **[XDEmineur]** |

Ambos equipos se encontraban conectados a la misma red local durante la actividad.

## Materiales y recursos utilizados

- Raspberry Pi.
- Tarjeta microSD con el sistema operativo instalado.
- Computadora cliente con un entorno gráfico compatible con X11 o XWayland.
- Conexión de red local mediante Ethernet o Wi-Fi.
- Fuente de alimentación para la Raspberry Pi.
- Monitor, teclado y cable HDMI para la configuración inicial de la Raspberry Pi.
- Cliente y servidor OpenSSH.
- Paquete `isc-dhcp-server`.
- Aplicación gráfica utilizada para comprobar el funcionamiento.
- Guía de referencia: [How to forward X over SSH to run graphics applications remotely](https://sobrebits.com/montar-un-servidor-casero-con-raspberry-pi-parte-3-configurar-servidor-dhcp/).

## Definiciones

## Esquema de la Red (Cisco Packet Tracer)

![cisco](media/cisco.png)

### Hay uqe poner

## Procedimiento

### 1. Instalación de Paquetes de Servidor DHCP

El primer paso es conectarnos a la Raspberry con ssh e instalar el paquete isc-dhcp-server con lo necesario para la configuración:
```
sudo apt install isc-dhcp-server
```

![paso 1](media/paso1.png)

## 2. Puerto DHCP
Configuramos /etc/default/isc-dhcp-server para especificar el puerto para la conexion con el resto de la LAN
```
sudo nano /etc/default/isc-dhcp-server
```
![paso 2](media/paso2.png)

## 3. Copia de Seguridad
Queremos configurar el servicio modificando el archivo /etc/dhcp/dhcpd.conf, asi que para poder recuperarlo en caso de errores lo copiamos con el comandp 'cp'
```
sudo cp /etc/dhcp/dhcpd.conf dhcp.conf
```
![paso 3](media/paso3.png)

## 4. Configuracion general del Servicio
Configuramos el archivo general para especificar cosas como la subred, el rango de asignacion, los servidores DNS, etc
(le configuramos un rango de .51 a .254)
![paso 4](media/paso4.png)
![paso 4b](media/paso4b.png)
(archivo despues de editado)
## 5. Usando nmtui para configurar la direccion ip del servidor, mascara de subred y gateway
(.1 es el gateway y el dhcp al mismo tiempo porque despues lo vamos a configurar como router)
Le configuramos las direcciones DNS para que use los servidores de google (8.8.8.8 y 8.8.8.4) y los de cloudflare (1.1.1.1)
![paso 5](media/paso5.png)
![paso 5b](media/paso5b.png)
![paso 5c](media/paso5c.png)
## 6. Activamos el servicio
![paso 6](media/paso6.png)
```
systemctl enable isc-dhcp-server
```

El mensaje que devuelve es porque el paquete isc-dhcp-server es antiguo y utiliza el servicio legacy sysvinit. Systemd detecta esto y automaticamente nos redirecciona el comando a sysvinit para activarlo

## 7. Chequeamos la conexion con un ping
Pingeamos la raspberry conectada con un switch a nuestra notebook
```
ping 192.168.5.1
```
![paso 7](media/paso7.png) 

## Comprobaciones realizadas

| Comprobación | Comando | Resultado esperado |
|---|---|---|
| Estado del servicio SSH | `sudo systemctl status ssh` | `active (running)` |
| Sintaxis de la configuración | `sudo sshd -t` | Sin mensajes de error |
| Reenvío X11 habilitado | `grep -i "^X11Forwarding" /etc/ssh/sshd_config` | `X11Forwarding yes` |
| Instalación de `xauth` | `which xauth` | Una ruta como `/usr/bin/xauth` |
| Conexión con reenvío gráfico | `ssh -X usuario@direccion_ip` | Conexión iniciada correctamente |
| Variable gráfica | `echo $DISPLAY` | Un valor similar a `localhost:10.0` |
| Aplicación remota | Comando del buscaminas | Ventana visible en la computadora cliente |

## Resultado

Resultado

## Posibles errores y soluciones


## Consideraciones de seguridad



## Conclusión



