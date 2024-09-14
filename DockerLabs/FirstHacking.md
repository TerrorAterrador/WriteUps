# [FirstHacking](https://dockerlabs.es/)

## Despliegue

Primero desplegamos la máquina con ```bash auto_deploy.sh firsthacking.tar``` (si no sabes en la página de DockerLabs ahí un pdf que lo explica).


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

Podemos ver los resultados en el archivo grepeable haciendo ```cat allPorts```, observamos que tan solo está abierto el puerto **21**
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/c42712ae-3adb-4232-98c4-a787f7784cd4)
<br>
<br>

Al ver que solo tenemos un puerto abierto vamos ha hacer un escaneo de nmap pero para que nos liste más información. Para llevar a cabo eso debemos hacer ```nmap -p21 -sCV 172.17.0.2 -oN targeted``` <br>
`-p21` ⮞ aplicar el escaneo solo al puerto 21 >
`-sC` ⮞ ejecuta los scripts de reconocimiento básico, los más comunes <br> 
`-sV` ⮞ para conocer la versión del servicio que corre por el puerto (se puede juntar con el anterior y quedaría así `-sCV`)<br> 
`-sS` ⮞ para descubrir puertos de manera silenciosa y rápida <br> 
`-oN` ⮞ lo exporta en formato nmap al archivo targeted 
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/efd74f95-4541-45e2-9857-c6b81c6c6d86)
<br>
<br>

## FTP (Puerto 21)

Una vez ya conozcamos la versión del ftp podemos buscar por ella en Internet en búsquedad de algún exploit de esta forma `ftp vsftpd 2.3.4 exploit github`. <br> Nos encontramos con este repositorio de [github](https://github.com/Hellsender01/vsftpd_2.3.4_Exploit). Si seguimos las instrucciones del repositorio instalando los requirimientos con `sudo python3 -m pip install pwntools`. <br> Nos descargamos el repositorio con `git clone https://github.com/Hellsender01/vsftpd_2.3.4_Exploit.git`. Una vez esté descargado probamos ha hacer lo que nos dice que sería `python3 exploit.py 172.17.0.2`. Y listo ya seríamos root.
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/345f784d-7c90-4ac1-948a-b27703598104)
