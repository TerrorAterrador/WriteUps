# [Vacaciones](https://dockerlabs.es/)

## Despliegue

Primero desplegamos la máquina con `bash auto_deploy.sh vacaciones.tar` (si no sabes en la página de DockerLabs ahí un pdf que lo explica).

## Reconocimiento

Una vez desplegada comprobamos que tenemos conectividad con `ping -c 1 172.17.0.2` <br>
![ping](https://github.com/TerrorAterrador/WriteUps/assets/128630899/c84ab9ce-1758-4a9c-8679-a7ee2a43c3be)
<br>
<br>
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
![nmap](nmap.jpg)
<br>
<br>
## Página Web (Puerto 80)

Al ver que está abierto el puerto 80 nos dirigimos al Navegador Web e introducimos la dirección IP como URL. Podemos ver que la página está completamente en blanco. <br>
![navegador](navegador.jpg) 
<br>
<br>
Pasemos a ver el código fuente `click derecho View Page Source`, vemos que hay un comentario y encontramos dos nombres **Juan** y **Camilo** <br>
![codigo_fuente](codigo_fuente.jpg) 
<br>
<br>
## Medusa / Hydra
Ahora conocemos dos usuarios con posibilidades de ser candidatos a pertenecer a la máquina Vacaciones. Gracias a medusa haremos un ataque de fuerza fruta al puerto 22, el cual aloja el servicio SSH. `medusa -h 172.17.0.2 -U users -P /usr/share/wordlists/rockyou.txt -M ssh` <br>
`-h` ⮞ dirección IP de la máquina victima <br>
`-U` ⮞ el archivo con los posibles usuarios es decir (camilo,juan) <br> 
`-P` ⮞ ruta al diccionario [rockyou]([rockyou](https://github.com/brannondorsey/naive-hashcat/releases/download/data/rockyou.txt) <br> 
`-M` ⮞ modulo que sería el protocolo ssh <br>
![medusa](medusa.jpg)
<br>
<br>
## SSH (Puerto 22)
Una vez conocemos el usuario y su contraseña probramos a entrar a la máquina Vacaciones con `ssh camilo@172.17.0.2`, y a continuación nos pedirá la contraseña. *Si te aparece un error como este ![errorSSH](error.jpg) aquí puedes encontrar la solción.*


## Escala de Privilegios
Comprobamos que hemos podido ingresar a la Máquina Víctima como **camilo**, podemos escribir `bash` para poder estar más cómodos operando en la máquina. <br>
Si ejecutamos `sudo -l` podemos ver que no podemos correr nada como sudo.<br>
![sudo_-l](sudo_-l)
 <br>
 <br>
`-l` ⮞ listar comandos que podemos ejecutar como sudo <br>
<br>
<br>
Al igual que si ejecutamos `find / -perm -4000 2>/dev/null` en búsqueda de permisos SUID no encontramos nada potencial para escalar privilegios. <br>
![find](find.jpg). 
<br>
`/` ⮞ buscamos desde la raíz
`-perm -4000` ⮞ mostrar los permisos SUID <br>
`2>/dev/null` ⮞ para que no nos muestre los errores <br>
<br>
<br>
Por lo que ahora deberíamos buscar el correo que había enviado Juan para Camilo si hacemos `find / -name "correo" 2>/dev/null`, vemos que no hay nada. <br>
![find_correo](find_correo.jpg)
 <br>
 <br>
`-name` ⮞ nombre que queremos encontrar <br>
En cambio si probamos a filtrar por **mail** `find / -name "mail" 2>/dev/null` nos encontramos con lo siguiente: <br>
![find_mail](find_correo.jpg) 
<br>
<br>
Por lo que nos dirigimos a esa ruta **/var/mail** vemos que dentro de la carpeta camilo hay un archivo llamado **correo.txt** y si hacemos `cat correo.txt` vemos lo siguiente: <br>
![archivo](archivo.jpg) 
<br>
<br>
Por lo que vamos a a probar camibar al usuario **juan** que era el que había enviado el correo, si hacemos `su juan` para cambiar de usuario y ponemos la contraseña, vemos que hemos cambiado al usuario juan.<br>
![juan](juan.jpg) 
<br>
<br>
Ahora que estamos como juan vamos a listar los permisos que podemos ejecutar como **sudo** con `sudo -l`, y vemos que podemos ejecutar **/usr/bin/ruby**. <br>
![ruby](ruby.jpg)
<br>
<br>
Por lo que deberíamos hacer ahora es dirigirnos a la página [GTFOBins](https://gtfobins.github.io/) (está página nos indica como elevar privilegios dependiendo del binario que podamos ejecutar), después nos vamos a la parte de sudo en ruby, y nos encontramos con el siguiente comando `sudo ruby -e 'exec "/bin/bash"'`, probamos a ejecutarlo en la máquina y listo hemos elevado a root. <br>
![root](root.jpg) 






