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
![paso 2](media/paso2.png)
![paso 3](media/paso3.png)
![paso 4](media/paso4.png)
![paso 4b](media/paso4b.png)
![paso 5](media/paso5.png)
![paso 5b](media/paso5b.png)
![paso 5c](media/paso5c.png)
![paso 6](media/paso6.png)
![paso 7](media/paso7.png)


## Opción `-X` y opción `-Y`

SSH ofrece dos opciones relacionadas con el reenvío X11:

| Opción | Funcionamiento | Seguridad |
|---|---|---|
| `-X` | Habilita un reenvío X11 con restricciones. | Es la opción recomendada para esta práctica. |
| `-Y` | Habilita un reenvío X11 considerado confiable. | Tiene menos restricciones, pero concede más permisos a las aplicaciones remotas. |

Utilizamos `-X` porque ofrece mayores restricciones de seguridad. La opción `-Y` debería emplearse únicamente cuando se confía en el equipo remoto y una aplicación no funciona correctamente con `-X`.

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

La prueba fue exitosa: el buscaminas se ejecutó en la Raspberry Pi y su ventana se mostró en la computadora cliente a través de la conexión SSH.

![Buscaminas ejecutado mediante X11 Forwarding](media/buscaminas.png)

**Figura 1.** Aplicación de buscaminas ejecutada en la Raspberry Pi y visualizada en la computadora cliente mediante X11 Forwarding sobre SSH.

## Posibles errores y soluciones

### La variable `DISPLAY` aparece vacía

Se debe comprobar que:

- La conexión se haya iniciado con `ssh -X`.
- La línea `X11Forwarding yes` se encuentre activa en `/etc/ssh/sshd_config`.
- El servicio SSH se haya reiniciado después de modificar el archivo.
- `xauth` esté instalado en la Raspberry Pi.

### La aplicación indica que no puede abrir la pantalla

Puede aparecer un mensaje similar a:

```text
Error: Can't open display
```

En ese caso, se debe revisar el valor de `DISPLAY`, la disponibilidad del entorno X11 o XWayland en la computadora cliente y la configuración del servidor SSH.

### La conexión SSH funciona, pero la ventana no aparece

Se puede iniciar la conexión en modo detallado para revisar el proceso:

```bash
ssh -v -X usuario@direccion_ip
```

La opción `-v` muestra información de diagnóstico y permite comprobar si SSH solicitó correctamente el reenvío X11.

## Consideraciones de seguridad

- El tráfico de X11 se transporta dentro del canal cifrado de SSH.
- No se deben escribir contraseñas dentro del informe ni mostrarlas en capturas de pantalla.
- Es preferible utilizar `ssh -X` en lugar de `ssh -Y`, salvo que exista una razón técnica justificada.
- Solo se deben ejecutar aplicaciones provenientes de equipos remotos confiables.
- No es recomendable desactivar controles de autenticación mediante comandos como `xhost +`, debido a que permitirían conexiones gráficas sin una restricción adecuada.

## Conclusión

Con esta actividad logramos configurar y comprobar el reenvío X11 mediante SSH. La aplicación gráfica se ejecutó en la Raspberry Pi, mientras que su ventana se mostró en la computadora cliente por medio de una conexión cifrada.

También comprendimos la función de la opción `-X`, de la herramienta `xauth`, de la variable `DISPLAY` y del archivo `/etc/ssh/sshd_config`. La ejecución del buscaminas confirmó que la configuración funcionaba correctamente.

Esta configuración puede facilitar la administración remota de aplicaciones gráficas instaladas en la Raspberry Pi. Sin embargo, X11 Forwarding no es un requisito para utilizar posteriormente la Raspberry Pi como router, sino una herramienta adicional para su administración remota.

## Evidencias recomendadas para la entrega

Para que el procedimiento quede completamente demostrado, se recomienda incluir capturas de:

1. La línea `X11Forwarding yes` en `/etc/ssh/sshd_config`.
2. El resultado de `sudo sshd -t` y `sudo systemctl status ssh`.
3. La comprobación de `xauth` mediante `which xauth`.
4. La conexión realizada con `ssh -X`.
5. El resultado de `echo $DISPLAY`.
6. La ventana del buscaminas visible en la computadora cliente.

Cada captura debe tener un número, un título breve y una explicación de lo que demuestra.
