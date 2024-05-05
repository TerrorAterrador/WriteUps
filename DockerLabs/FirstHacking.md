# [FirstHacking](https://dockerlabs.es/)

## Despliegue

Primero desplegamos la máquina con ```bash auto_deploy.sh firsthacking.tar``` (si no sabes en la página de DockerLabs ahí un pdf que lo explica).


## Reconocimiento

Una vez desplegada comprobamos que tenemos conectividad con ```ping -c 1 172.17.0.2``` 
![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/861bdb0c-48c1-418f-94e0-fb050e962e01)
<br>
`-c 1` ⮞ solo lo repite una vez<br>

Ahora vamos con el reconocimiento de nmap ```
nmap -p- --open --min-rate 5000 -sS -vvv -n -Pn 172.17.0.2 -oG allPorts
``` <br>
`-p-` ⮞ aplicar reconocimiento a todos los puertos <br>
`--open` ⮞ solo a los que esten abiertos <br>
`--min-rate 5000` ⮞ para enviar paquetes más rápido <br> 
`-sS` ⮞ para descubrir puertos de manera silenciosa y rápida <br> 
`-vvv` ⮞ conforme descubre un puerto nos lo muestra por pantalla <br> 
`-n` ⮞ no aplica la resolución DNS (tarda mucho en el caso de que no pongamos dicho parámetro)<br> 
`-Pn` ⮞ ignora si esta activa o no la IP<br> 
`-oG` ⮞ exportamos el resultado en formato grepeable (para extraer mejor los datos con herramientas como grep, awk) <br>
<br>

Podemos ver los reultados en el archivo grepeable haciendo ```cat allPorts```, observamos que tan solo está abierto el puerto **21**<br>
![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/afe6e330-d1a5-4d20-85c1-050ffd9c4447)
<br>

Al ver que solo tenemos un puerto abierto vamos ha hacer un escaneo de nmap pero para que nos liste más información. Para llevar a cabo eso debemos hacer ```nmap -p21 -sCV 172.17.0.2 -oN targeted``` <br>
`-p21` ⮞ aplicar el escaneo solo al puerto 21 >
`-sC` ⮞ ejecuta los scripts de reconocimiento básico, los más comunes <br> 
`-sV` ⮞ para conocer la versión del servicio que corre por el puerto (se puede juntar con el anterior y quedaría así `-sCV`<br> 
`-sS` ⮞ para descubrir puertos de manera silenciosa y rápida <br> 
`-oN` ⮞ lo exporta en formato nmap al archivo targeted<br> 
![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/06ec2233-1d98-4c27-bc73-e39b78509e98)
<br>

## FTP (Puerto 21)

Una vez ya conozcamos la versión del ftp podemos buscar por ella en Internet en búsquedad de algún exploit de esta forma ```ftp vsftpd 2.3.4 exploit github```. Nos encontramos con este repositorio de [github](https://github.com/Hellsender01/vsftpd_2.3.4_Exploit). Si seguimos las instrucciones del repositorio instalando los requirimientos con ```sudo python3 -m pip install pwntools```. Nos descargamos en repositorio con ```git clone https://github.com/Hellsender01/vsftpd_2.3.4_Exploit.git```. Una vez esté descargado probamos ha hacer lo que nos dice que sería ```python3 exploit.py 172.17.0.2```. Y listo ya seríamos root.

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/73b40cd0-6478-4f0a-9581-2dbc6320d21f)
