# [Amor](https://dockerlabs.es/)

## Despliegue

Primero desplegamos la máquina con ```bash auto_deploy.sh amor.tar``` (si no sabes en la página de DockerLabs ahí un pdf que lo explica).


## Reconocimiento

Una vez desplegada comprobamos que tenemos conectividad con ```ping -c 1 172.17.0.2``` 
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/af4d0189-b640-4576-aca6-3c02c75c9434)
<br>
`-c 1` ⮞ solo lo repite una vez<br>
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

Podemos ver los resultados en el archivo grepeable haciendo ```cat allPorts```, observamos que está abierto el puerto **22** y **80**.
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/8563720d-b839-447e-988d-bb80300a4b67)

<br>
<br>

## Página Web (Puerto 80)

Al ver que aloja un servidor web, nos dirigimos a él poniendo en el buscador la ip que en este caso sería `172.17.0.2`. Vemos que nos da varias vistas:
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/ce9ba3fe-f606-47cc-8e14-9323006440a0)

Como por ejemplo, que un usuario tiene una contraseña débil y que existen dos posible usuarios `Juan` y `Carlota`.

<br>
<br>

## Hydra / Medusa

Una vez conocemos los posibles usuarios haremos un ataque de fuerza bruta al servicio ssh usando hydra en primer lugar, usaremos el usuario juan `hydra -l juan -P /usr/share/wordlists/rockyou.txt ssh://172.17.0.2`. <br> 
`-h` ⮞ dirección IP de la máquina victima <br>
`-u` ⮞ nombre del posible usuario <br> 
`-P` ⮞ ruta del rockyou. Para descargar el diccionario [rockyou](https://github.com/brannondorsey/naive-hashcat/releases/download/data/rockyou.txt).

<br>

No encontraríamos nada, por lo que haremos fuerza bruta al otro usuario (Carlota) de la siguiente forma `hydra -l carlota -P /usr/share/wordlists/rockyou.txt ssh://172.17.0.2`, en esta caso si que nos encontraría la contraseña:
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/dd0fa07f-95cd-4dfd-8661-ab1c4dcecc44)

 > Podríamos usar un archivo el cual contenga los nombre de usuarios, pero en esta caso tardaría más en encontrarnos la contraseña siempre y cuando pongamos primero al usuario juan.

<br>
<br>

## SSH (Puerto 22)

Una vez conocemos el usuario y su contraseña probamos a entrar a la máquina Amor con `ssh carlota@172.17.0.2`, y a continuación nos pedirá la contraseña. <br>

> Si te aparece un error como este [aquí](https://desarrolloweb.com/faq/solucionar-remote-host-identification-has-changed-al-hacer-ssh) puedes encontrar la solución. <br>![image](https://github.com/TerrorAterrador/WriteUps/assets/128630899/2128bd5f-33a2-4bb0-ac54-6555c7aa5817)

<br>
<br>

## Escalada de Privilegios

En este caso no será necesario el Tratamiento de la TTY con tan solo escribiendo bash ya tendremos una terminal cómoda. En todo caso el tratamiento sería así: <br>
1-.`script /dev/null -c bash` <br>
2-.`Pulsamos CTRL+Z` <br>
3-.`stty raw -echo; fg` <br>
4-.`reset xterm` <br>
5-.`export SHELL=bash && export TERM=xterm` <br>

<br>

Ejecutamos el comando `sudo -l` para ver que podemos ejecutar como sudo, pero no encontramos nada: 
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/59e0946c-7f69-430c-b586-a0ef4e31b6f0)

<br>

Al igual que si ejecutamos `find / -perm -4000 2>/dev/null` en búsqueda de permisos SUID no encontramos nada potencial para escalar privilegios. 
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/e2862f75-c832-4f97-be42-175ef81ccdf6)

`/` ⮞ buscamos desde la raíz <br>
`-perm -4000` ⮞ mostrar los permisos SUID <br>
`2>/dev/null` ⮞ para que no nos muestre los errores 

<br>

Si nos vamos a la siguiente ruta `/home/carlota/Desktop/fotos/vacaciones/imagen.jpg` dentro del directorio personal de **carlota** nos encontramos con una imagen, nos la llevamos a nuestra máquina de atacante montando un servidor http con python de la siguiente manera: `python3 -m http.server 555`.
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/c8c89ec2-e7a4-4532-a7f1-ff9757a29241)

<br>

Obtenemos dicha imagen, con el siguiente comando desde nuestra máquina de atacante `wget 172.17.0.2:555/imagen.jpg`
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/8f146058-f302-4061-9149-8d875a340341)

<br>
<br>

### Estenografía

Pasamos a analizar los metadatos de la imagen, para ver si tiene algo oculto, usaremos el siguiente comando: `steghide extract -sf imagen.jpg`, nos pedirá una passphrase pero le damos al enter, y obtendremos un archivo `secret.txt` el cual contendrá el siguiente contenido:
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/bc96bdef-8057-447b-9f64-59837d284d5b)

<br>

Parece que tiene una cadena de texto en base64 por lo que pasaremos a decodificarla con el siguiente comando `cat secret.txt`, y obtendremos lo que parece una contraseña:
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/f8ae1f7a-bed5-4212-81fa-812845870ba8)

<br>

Probamos a autenticarnos como oscar con el siguiente comando `su oscar` -> introducimos la contraseña y ya seremos oscar y escribimos `bash` para ejecutar una terminal de bash en ves de una sh:
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/56d202b6-f64c-43cf-b580-ce4585832cbc)

<br>

Seguimos el mismo procedimiento de antes, ejecutando el comando `sudo -l` y los reporta lo siguiente:
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/d70b5646-7f76-4925-9d37-fad3361ff03e)

<br>

Por lo que deberíamos hacer ahora es dirigirnos a la página [GTFOBins](https://gtfobins.github.io/) (está página nos indica como elevar privilegios dependiendo del binario que podamos ejecutar), después nos vamos a la parte de sudo en ruby, y nos encontramos con lo siguiente:
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/804e3fcd-5f15-44fa-8c3c-ed073e80517d)

<br>

Por lo cual ejecutaremos lo siguiente `sudo -u root ruby -e 'exec "/bin/bash"'`, y listo ya seremos root:
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/1716e422-11bc-4ec8-a191-49db425e1f1b)
