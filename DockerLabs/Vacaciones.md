# [Vacaciones](https://dockerlabs.es/)

### Despliegue

Primero desplegamos la máquina (si no sabes en la página de DockerLabs ahí un pdf que lo explica).

### Reconocimiento

Una vez desplegada comprobamos que tenemos conectividad con `ping -c 1 172.17.0.2` <br>
![ping](ping.jpg) <br>
`-c 1` ⮞ solo lo repite una vez<br>

Ahora vamos con el reconocimiento de nmap `nmap -p- --open --min-rate 5000 -sS -vvv -n -Pn 172.17.0.2 -oG allPorts` <br>
`-p-` ⮞ aplicar reconocimiento a todos los puertos <br>
`--open` ⮞ solo a los que esten abiertos <br>
`--min-rate 5000` ⮞ para enviar paquetes más rápido <br> 
`-sS` ⮞ para descubrir puertos de manera silenciosa y rápida <br> 
`-vvv` ⮞ conforme descubre un puerto nos lo muestra por pantalla <br> 
`-n` ⮞ no aplica la resolución DNS (tarda mucho en el caso de que no pongamos dicho parámetro)<br> 
`-Pn` ⮞ ignora si esta activa o no la IP<br> 
`-oG` ⮞ exportamos el resultado en formato grepeable (para extraer mejor los datos con herramientas como grep, awk) <br>
<br>
Podemos ver los reultados en el archivos grepeable haciendo `cat allPorts`, observamos que están abiertos los puertos **22** y **80**<br>
![nmap](nmap.jpg)<br>
<br>

### Página Web (Puerto 80)

Al ver que está abierto el puerto 80 nos dirigimos al Navegador Web e introducimos la dirección IP como URL. Podemos ver que la página está completamente en blanco. <br>
![navegador](navegador.jpg) <br>
<br>

Pasemos a ver el código fuente `click derecho View Page Source`, vemos que hay un comentario y encontramos dos nombres **Juan** y **Camilo** <br>
![codigo_fuente](codigo_fuente.jpg) <br>
<br>

### Medusa / Hydra









