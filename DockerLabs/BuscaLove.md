# [BuscaLove](https://dockerlabs.es/)

## Despliegue

Primero desplegamos la máquina con ```bash auto_deploy.sh buscalove.tar``` (si no sabes en la página de DockerLabs ahí un pdf que lo explica).


## Reconocimiento

Una vez desplegada comprobamos que tenemos conectividad con ```ping -c 1 172.18.0.2``` 
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/efc1166f-2d5f-4e1b-a8cb-f76ae0103390)
<br>

`-c 1` ⮞ solo lo repite una vez<br>
<br>

Ahora vamos con el reconocimiento de nmap ```nmap -p- --open --min-rate 5000 -sS -vvv -n -Pn 172.18.0.2 -oG allPorts``` <br>
`-p-` ⮞ aplicar reconocimiento a todos los puertos <br>
`--open` ⮞ solo a los que estén abiertos <br>
`--min-rate 5000` ⮞ para enviar paquetes más rápido <br> 
`-sS` ⮞ para descubrir puertos de manera silenciosa y rápida <br> 
`-vvv` ⮞ conforme descubre un puerto nos lo muestra por pantalla <br> 
`-n` ⮞ no aplica la resolución DNS (tarda mucho en el caso de que no pongamos dicho parámetro)<br> 
`-Pn` ⮞ ignora si esta activa o no la IP<br> 
`-oG` ⮞ exportamos el resultado en formato grepeable (para extraer mejor los datos con herramientas como grep, awk)
<br>

Podemos ver los resultados en el archivo grepeable haciendo ```cat allPorts```, observamos que están abiertos los puertos **22** y **80**.
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/f54c631e-1952-4498-a5ed-b2f9adf3e3e0)

<br>
<br>

## Página Web (Puerto 80)

Al ver que aloja un servidor web, nos dirigimos a él poniendo en el buscador la ip que en este caso sería `172.18.0.2`. Vemos que se trata de la página por defecto de Apache2:
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/ce9ba3fe-f606-47cc-8e14-9323006440a0)

<br>
<br>

## Gobuster

Tras inspeccionar el código, pasaremos a hacer fuzzing a la página web para encontrar directorios `http://172.18.0.2/`, lo haremos de la siguiente manera ``, y nos encuentra lo siguiente:
