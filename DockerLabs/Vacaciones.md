# [Vacaciones](https://dockerlabs.es/)

### Despliegue

Primero desplegamos la máquina (si no sabes en la página de DockerLabs ahí un pdf que lo explica).

### Reconocimiento

Una vez desplegada comprobamos que tenemos conectividad con `ping -c 1 172.17.0.2` <br>
![ping](ping.jpg) <br>
`-c 1` ⮞ solo lo repite una vez<br>

Ahora vamos con el reconocimiento de nmap `nmap -p- --open --min-rate 5000 -sS -vvv -n -Pn 172.17.0.2 -oG allPorts`
![nmap](nmap.jpg) <br>
`-p-` ⮞ aplicar reconocimiento a todos los puertos
`--open` ⮞ solo a los que esten abiertos
`--min-rate 5000` ⮞ para enviar paquetes más rápido
`-sS` ⮞ para descubrir puertos de manera silenciosa y rápida
`-vvv` ⮞ conforme descubre un puerto nos lo muestra por pantalla
`-n` ⮞ no aplica la resolución DNS (tarda mucho en el caso de que no pongamos dicho parámetro)
`-Pn` ⮞ ignora si esta activa o no la IP
`-oG` ⮞ exportamos el resultado en formato grepeable (para extraer mejor los datos con herramientas como grep, awk)





