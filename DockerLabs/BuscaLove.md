# [BuscaLove](https://dockerlabs.es/)

## Despliegue

Primero desplegamos la máquina con ```bash auto_deploy.sh buscalove.tar``` (si no sabes en la página de DockerLabs ahí un pdf que lo explica).


## Reconocimiento

Una vez desplegada comprobamos que tenemos conectividad con ```ping -c 1 172.18.0.2``` 
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/efc1166f-2d5f-4e1b-a8cb-f76ae0103390)
<br>

`-c 1` ⮞ solo lo repite una vez

<br>

Ahora vamos con el reconocimiento de nmap ```nmap -p- --open --min-rate 5000 -sS -vvv -n -Pn 172.18.0.2 -oG allPorts``` <br>
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

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/f54c631e-1952-4498-a5ed-b2f9adf3e3e0)

<br>
<br>

## Página Web (Puerto 80)

Al ver que aloja un servidor web, nos dirigimos a él poniendo en el buscador la ip que en este caso sería `172.18.0.2`. Vemos que se trata de la página por defecto de Apache2:
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/ae35a7d5-c88a-4f21-9965-231b41008c8c)

<br>
<br>

### Gobuster

Tras inspeccionar el código fuente y no encontrar nada, pasaremos a hacer fuzzing a la página web para encontrar posibles directorios `http://172.18.0.2/`, lo haremos de la siguiente manera `gobuster dir -w /home/kali/WordLists/directory-medium -u http://172.18.0.2 -x txt,sql,py,js,php,html`, y nos encuentra lo siguiente:
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/0221e5cd-23ad-40a2-860f-cb1acba67363)

<br>

Por lo que entendemos que la página aloja un wordpress. Ahora pasaremos a hacer fuzzing al wordpress de la siguiente forma
`gobuster dir -w /home/kali/WordLists/directory-medium -u http://172.18.0.2/wordpress -x txt,sql,py,js,php,html`, y nos encuentra el siguiente archivo `.php`:
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/49ed03b1-1d00-4abc-8129-27459221fe7b)

<br>

Una vez finalizado la enumeración de posibles archivos y/o directorios dentro de la dirección del wordpress pasamos a ver la página web y tendría esta apariencia:
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/267d040a-cff4-476f-9c00-dfc80b608133)

<br>

Ahora pasamos a inspeccionar el código fuente con `CTRL + U`, y vemos la siguiente pista:
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/3dc40b55-3bc3-4307-b6ef-0011b8193dab)

<br>

Volvemos a la página web, pero esta vez accederemos al archivo que con extensión `.php` que nos detectó gobuster:
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/559d2507-77ec-4b01-8cd8-e1622a5153db)

<br>

Vemos que no ha cambiado la apariencia, por lo que en este punto intentaremos probar si existe un LFI (Local File Inclusion) y lo haremos con `wfuzz` de la siguiente manera: <br>
`wfuzz -w /usr/share/wordlists/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -u http://172.18.0.2/wordpress/index.php?FUZZ=../../../../../etc/passwd --hc 404 --hl 40`
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/3b7616c7-2e09-4b94-a80e-22478331b80c)

<br>

Vemos que dicha página es vulnerable a un LFI pasándole la palabra `love`como parámetro al `index.php` (El comentario que encontramos antes nos daba una pista), por lo que nos dirigimos a la web e introducimos en la URL lo siguiente: <br> `http://172.18.0.2/wordpress/index.php?love=../../../../../etc/passwd`
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/3bc715a0-d561-4548-913d-0d76488ce800)

<br>

Podemos observar que existen varios usuarios como **pedro** y **rosa**.

<br>
<br>

Una vez conocemos los posibles usuarios haremos un ataque de fuerza bruta al servicio ssh usando hydra en primer lugar, usaremos el usuario bob `hydra -l pedro -P /usr/share/wordlists/rockyou.txt ssh://172.17.0.2`. <br> 
`-h` ⮞ dirección IP de la máquina victima <br>
`-u` ⮞ nombre del posible usuario <br> 
`-P` ⮞ ruta del rockyou. Para descargar el diccionario [rockyou](https://github.com/brannondorsey/naive-hashcat/releases/download/data/rockyou.txt).

<br>

No encontraríamos nada, por lo que haremos fuerza bruta al otro usuro (Carlota) de la siguiente forma `hydra -l rosa -P /usr/share/wordlists/rockyou.txt ssh://172.18.0.2`, en esta caso si que nos encontraría la contraseña:
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/e4787deb-88e5-4601-ac7c-052017db974c)

<br>

 > Podríamos usar un archivo el cual contenga los nombre de usuarios, pero en esta caso tardaría más en encontrarnos la contraseña siempre y cuando pongamos primero al usuario pedro.

<br>
<br>

## SSH (Puerto 22)

Una vez conocemos el usuario y su contraseña probamos a entrar a la máquina BuscaLove con `ssh rosa@172.18.0.2`, y a continuación nos pedirá la contraseña. <br>

> Si te aparece un error como este [aquí](https://desarrolloweb.com/faq/solucionar-remote-host-identification-has-changed-al-hacer-ssh) puedes encontrar la solución. <br>![image](https://github.com/TerrorAterrador/WriteUps/assets/128630899/2128bd5f-33a2-4bb0-ac54-6555c7aa5817)

<br>
<br>

## Escalada de Privilegios

Comprobamos que hemos accedido a la máquina como **rosa**:
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/c62e9b7b-ae66-43d3-81ec-f9217deac59f)
 > El (;) concatena dos comandos.
<br>

En este caso no será necesario el Tratamiento de la TTY con tan solo escribiendo bash ya tendremos una terminal cómoda. En todo caso el tratamiento sería así: <br>
1-.`script /dev/null -c bash` <br>
2-.`Pulsamos CTRL+Z` <br>
3-.`stty raw -echo; fg` <br>
4-.`reset xterm` <br>
5-.`export SHELL=bash && export TERM=xterm` <br>

<br>

Ejecutamos el comando `sudo -l`, para ver que podemos correr como sudo y encontramos lo siguiente: 
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/3ba01539-0182-4154-a2e0-6e79c27a763f)

<br>

Por lo que podemos listar directorios y ver archivos a los que tendríamos acceso, en primer lugar listamos la HOME del usuario **pedro** del siguiente forma `sudo -u root ls -la /home/pedro`, pero no encontramos nada interesante:
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/260ac429-d484-49c4-956d-e25fe58c9d5a)

<br>

Pasamos a listar la HOME del usuario **root** de esta forma `sudo -u root ls -la /root`:
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/e3a521a6-576d-496c-8c83-2059f3c6af61)

<br>

Vemos un archivo un poco extraño llamado `secret.txt`, como podemos también ver el contenido de archivos a los que no tendríamos acceso, pues vemos el contenido de `secret.txt` de esta forma `sudo -u root cat /root/secret.txt`:
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/b073f42b-6cc5-402f-aad0-5c0ecd428ae7)

<br>

Vemos lo que parece un texto codificado en hexadecimal, por lo que nos dirigimos a [CyberChef](https://gchq.github.io/CyberChef/), le demos a la barita mágica y obtendríamos lo siguiente:
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/caad82a9-64b9-4357-8501-520c8a4b68c2)

<br>

Por lo que parece que parece que tenemos una contraseña en texto plano, nos autenticaremos como **pedro** de la siguiente forma `su pedro, introducimos la contraseña descifrada y conseguimos logearnos como **pedro**:
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/5fd190b8-18ce-41d2-9fa5-63193d596f0f)

<br>

Seguimos el mismo procedimiento de antes ejecutando `sudo -l` y vemos lo siguiente:
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/87e2cad4-098d-45dd-ae69-0c007560ce62)

<br>

Por lo que deberíamos hacer ahora es dirigirnos a la página [GTFOBins](https://gtfobins.github.io/) (está página nos indica como elevar privilegios dependiendo del binario que podamos ejecutar), después nos vamos a la parte de sudo de `env`, y nos encontramos con lo siguiente:
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/fb5e13a5-311d-4ce7-84ea-0a3fe4bc436d)

<br>

Por lo que probaremos a ejecutarlo en la máquina víctima poniendo `sudo -u root /usr/bin/env /bin/bash`
<br>

Comprobamos ya somos `root` y hacemos `whoami` para comprobarlo.
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/c5130bf3-3412-40e5-9380-f9f6db06a824)
