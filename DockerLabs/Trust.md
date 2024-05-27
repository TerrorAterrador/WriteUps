# [Trust](https://dockerlabs.es/)

## Despliegue

Primero desplegamos la máquina con `bash auto_deploy.sh trust.tar` (si no sabes en la página de DockerLabs ahí un pdf que lo explica).

## Reconocimiento

Una vez desplegada comprobamos que tenemos conectividad con `ping -c 1 172.17.0.2` 
![ping](https://github.com/TerrorAterrador/WriteUps/assets/128630899/c84ab9ce-1758-4a9c-8679-a7ee2a43c3be)
<br>
`-c 1` ⮞ solo lo repite una vez<br>

Ahora vamos con el reconocimiento de nmap `nmap -p- --open --min-rate 5000 -sS -vvv -n -Pn 172.17.0.2 -oG allPorts` <br>
`-p-` ⮞ aplicar reconocimiento a todos los puertos <br>
`--open` ⮞ solo a los que estén abiertos <br>
`--min-rate 5000` ⮞ para enviar paquetes más rápido <br> 
`-sS` ⮞ para descubrir puertos de manera silenciosa y rápida <br> 
`-vvv` ⮞ conforme descubre un puerto nos lo muestra por pantalla <br> 
`-n` ⮞ no aplica la resolución DNS (tarda mucho en el caso de que no pongamos dicho parámetro)<br> 
`-Pn` ⮞ ignora si esta activa o no la IP<br> 
`-oG` ⮞ exportamos el resultado en formato grepeable (para extraer mejor los datos con herramientas como grep, awk) <br>
<br>
Podemos ver los resultados en el archivo grepeable haciendo `cat allPorts`, observamos que están abiertos los puertos **22** y **80**

![nmap](https://github.com/TerrorAterrador/WriteUps/assets/128630899/ef28633b-ba49-4df1-8d2d-4efc516c8471)

<br>
<br>

## Página Web (Puerto 80)

Al ver que está abierto el puerto 80 nos dirigimos al Navegador Web e introducimos la dirección IP como URL. Podemos ver la página por defecto de Apache2. <br>
![alt text](image.png)
<br>
<br>

## Gobuster

Vamos a listar posibles directorios y/o archivos que estén en este servidor http podemos ver que nos reporta lo siguiente: 

![alt text](image-1.png)

<br>
<br>
Por lo que vamos a la siguiente dirección `http://172.17.0.2/secret.php`, y nos aparece lo siguiente: 

![alt text](image-2.png)

<br>
<br>


## Medusa / Hydra
Ahora conocemos un posible usuario de nombre `mario` con posibilidades de ser candidato a pertenecer a la máquina Trust. Gracias a hydra haremos un ataque de fuerza fruta al puerto 22, el cual aloja el servicio SSH. `hydra -l mario -P /usr/share/wordlists/rockyou.txt ssh://172.17.0.2 ` <br>
`-l` ⮞ nombre del posible usuario <br> 
`-P` ⮞ ruta para descargar el diccionario [rockyou](https://github.com/brannondorsey/naive-hashcat/releases/download/data/rockyou.txt) <br> 
`ssh://172.17.0.2` ⮞ especificamos el servicio (ssh) y la ruta de la máquina <br>

![alt text](image-3.png)

<br>
<br>

## SSH (Puerto 22)
Una vez conocemos el usuario y su contraseña probamos a entrar a la máquina Trust con `ssh mario@172.17.0.2`, y a continuación nos pedirá la contraseña. *Si te aparece un error como este [aquí](https://desarrolloweb.com/faq/solucionar-remote-host-identification-has-changed-al-hacer-ssh) puedes encontrar la solución.* <br>![image](https://github.com/TerrorAterrador/WriteUps/assets/128630899/2128bd5f-33a2-4bb0-ac54-6555c7aa5817)

<br>
<br>

## Escala de Privilegios
Comprobamos que hemos podido ingresar a la Máquina Víctima como **mario** si hacemos `whoami`.
![alt text](image-4.png)
<br>
<br>

Si ejecutamos `sudo -l` podemos ver que no podemos correr `/usr/bin/vim` como sudo.<br>
`-l` ⮞ listar comandos que podemos ejecutar como sudo

![alt text](image-5.png) 

<br>
<br>

Por lo que deberíamos hacer ahora es dirigirnos a la página [GTFOBins](https://gtfobins.github.io/) (está página nos indica como elevar privilegios dependiendo del binario que podamos ejecutar), después nos vamos a la parte de sudo en vim, y nos encontramos con lo siguiente `sudo vim -c ':!/bin/sh'`, que lo que haría sería ejecutar vim y después escaparte de este editor con la `!` y escribiendo ``/bin/sh`.

A continuación probaremos a ejecutar dicho comando en primer lugar ponemos -> `sudo vim`, una vez dentro del editor de vim escribimos `:!/bin/bash`
![alt text](image-6.png)
<br>
<br>

Y listo ya seríamos root!!
