# [Bozaruwarahctf](https://dockerlabs.es/)

## Despliegue

Primero desplegamos la máquina con ```bash auto_deploy.sh bozaruwarahctf.tar``` (si no sabes en la página de DockerLabs ahí un pdf que lo explica).


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

![image](https://github.com/TerrorAterrador/WriteUps/assets/128630899/da1b6454-c069-4b1c-93bc-266a6a2de61e)

<br>
<br>

## Página Web (Puerto 80)

Al ver que aloja un servidor web, nos dirigimos a él poniendo en el buscador la ip que en este caso sería `172.17.0.2`. Observamos que en dicho servidor web, hay una imagen de un kinder sorpresa, probamos a descargar la imagen con `Click Derecho -> Guardar imagen como`.
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/128630899/288084ce-9d72-4365-bc73-f3dedf16fe81)

<br>
<br>

## Estenografía

Una vez descargada la imagen, intentamos ver si hay algún archivo oculto dentro de esta imagen usando el comando `steghide extract -sf imagen.jpeg`, nos pedirá una passphrase como no tenemos le damos a enter y vemos que realmente si que había un archivo oculto llamado `secreto.txt`.
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/128630899/6f62ac4d-e0bb-4767-9d77-633747ddbab7)

<br>

Para ver el contenido del archivo usaremos el comando `cat secreto.txt` y observamos lo siguiente:
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/128630899/fe7640b1-039b-49b3-bfe9-118f1e44828e)

<br>

Lo cual nos da una pista. Atendiendo a intentamos buscar más información dentro de esta imagen como son los metadatos de está usando la herramienta `exiftool`, por lo cual usando el comando `exiftool imagen.jpeg` nos reporta lo siguiente:
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/128630899/0eea7991-2c3b-4973-be3c-71e09f30da6f)
<br>

Observamos el nombre de un usuario llamado `borazuwarah`.

<br>
<br>

## Hydra

Una vez conocemos el posible usuario aplicamos un ataque de fuerza fruta al puerto 22 (ssh) con hydra con el siguiente comando: `hydra -l borazuwarah -P /usr/share/wordlists/rockyou.txt ssh://172.17.0.2`. <br> 
`-h` ⮞ dirección IP de la máquina victima <br>
`-u` ⮞ nombre del posible usuario (borazuwarah) <br> 
`-P` ⮞ ruta del rockyou. Para descargar el diccionario [rockyou](https://github.com/brannondorsey/naive-hashcat/releases/download/data/rockyou.txt) <br> 
<br>
Nos reporta lo siguiente: 
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/128630899/f26de55b-9bdb-4b66-8729-1d0e2363867b)

<br>
<br>

## SSH (Puerto 22)

Por lo tanto ya conocemos un usuario y una contraseña por lo cual probamos a autenticarnos al servicio ssh con el siguiente comando `ssh borazuwarah@172.17.0.2`, y listo somos el usuario `borazuwarah`:
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/128630899/edd66ba8-39c2-44ea-8e58-cdd391bb00de)

<br>
<br>

## Escalada de Privilegios

Ejecutamos el comando `sudo -l`. <br>
`-l` ⮞ listar comandos que podemos ejecutar como sudo <br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/128630899/7812848c-90a9-4f25-91d1-644738d66a9b)

<br>

Observamos que podemos correr `/bin/bash`, por que hacemos `sudo -u root /bin/bash` que nos permitirá ejecutar la bash como root sin proporcionar contraseña, y listo ya somos root!
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/128630899/9fdeb6b7-caea-4865-9bf2-d11b985f6c86)
