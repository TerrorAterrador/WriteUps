# [Vacaciones](https://dockerlabs.es/)

## Despliegue

Primero desplegamos la máquina con `bash auto_deploy.sh vacaciones.tar` (si no sabes en la página de DockerLabs ahí un pdf que lo explica).

## Reconocimiento

Una vez desplegada comprobamos que tenemos conectividad con `ping -c 1 172.17.0.2` 
![ping](https://github.com/TerrorAterrador/WriteUps/assets/128630899/c84ab9ce-1758-4a9c-8679-a7ee2a43c3be)
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
Podemos ver los reultados en el archivos grepeable haciendo `cat allPorts`, observamos que están abiertos los puertos **22** y **80**
![nmap](https://github.com/TerrorAterrador/WriteUps/assets/128630899/ef28633b-ba49-4df1-8d2d-4efc516c8471)

<br>
<br>

## Página Web (Puerto 80)

Al ver que está abierto el puerto 80 nos dirigimos al Navegador Web e introducimos la dirección IP como URL. Podemos ver que la página está completamente en blanco. <br>
![image](https://github.com/TerrorAterrador/WriteUps/assets/128630899/9590b22b-e92e-4576-8c81-8e00036f3abb)
<br>
<br>
Pasemos a ver el código fuente `click derecho View Page Source`, vemos que hay un comentario y encontramos dos nombres **Juan** y **Camilo** <br>
![image](https://github.com/TerrorAterrador/WriteUps/assets/128630899/1560326c-a5b2-402d-b350-300e194a108c)
<br>
## Medusa / Hydra
Ahora conocemos dos usuarios con posibilidades de ser candidatos a pertenecer a la máquina Vacaciones. Gracias a medusa haremos un ataque de fuerza fruta al puerto 22, el cual aloja el servicio SSH. `medusa -h 172.17.0.2 -U users -P /usr/share/wordlists/rockyou.txt -M ssh` <br>
`-h` ⮞ dirección IP de la máquina victima <br>
`-U` ⮞ el archivo con los posibles usuarios es decir (camilo,juan) <br> 
`-P` ⮞ ruta al diccionario [rockyou]([rockyou](https://github.com/brannondorsey/naive-hashcat/releases/download/data/rockyou.txt) <br> 
`-M` ⮞ modulo que sería el protocolo ssh <br>
![image](https://github.com/TerrorAterrador/WriteUps/assets/128630899/b61cd171-07c5-44f4-9510-1adf069b785f)
<br>
<br>
## SSH (Puerto 22)
Una vez conocemos el usuario y su contraseña probramos a entrar a la máquina Vacaciones con `ssh camilo@172.17.0.2`, y a continuación nos pedirá la contraseña. *Si te aparece un error como este [aquí](https://desarrolloweb.com/faq/solucionar-remote-host-identification-has-changed-al-hacer-ssh) puedes encontrar la solción.* ![image](https://github.com/TerrorAterrador/WriteUps/assets/128630899/a9aa6ab0-8f37-4306-baa4-928cfac2bc3f) 


## Escala de Privilegios
Comprobamos que hemos podido ingresar a la Máquina Víctima como **camilo**, podemos escribir `bash` para poder estar más cómodos operando en la máquina. <br>
Si ejecutamos `sudo -l` podemos ver que no podemos correr nada como sudo.<br>
![image](https://github.com/TerrorAterrador/WriteUps/assets/128630899/32048e1e-4f33-4de4-ad64-8ebb9cc44714)
 <br>
 <br>
`-l` ⮞ listar comandos que podemos ejecutar como sudo <br>
<br>
<br>
Al igual que si ejecutamos `find / -perm -4000 2>/dev/null` en búsqueda de permisos SUID no encontramos nada potencial para escalar privilegios. <br>
![image](https://github.com/TerrorAterrador/WriteUps/assets/128630899/5c7f9741-947c-4ef3-ae77-1e4bd20360c9)
<br>
`/` ⮞ buscamos desde la raíz
`-perm -4000` ⮞ mostrar los permisos SUID <br>
`2>/dev/null` ⮞ para que no nos muestre los errores <br>
<br>
<br>
Por lo que ahora deberíamos buscar el correo que había enviado Juan para Camilo si hacemos `find / -name "correo" 2>/dev/null`, vemos que no hay nada. <br>
![image](https://github.com/TerrorAterrador/WriteUps/assets/128630899/4ab0c8ef-4680-4362-8eeb-69b827c94198)
 <br>
 <br>
`-name` ⮞ nombre que queremos encontrar <br>
En cambio si probamos a filtrar por **mail** `find / -name "mail" 2>/dev/null` nos encontramos con lo siguiente: <br>
![image](https://github.com/TerrorAterrador/WriteUps/assets/128630899/a124a789-1550-47a9-86d2-00c346672ffb)
<br>
<br>
Por lo que nos dirigimos a esa ruta **/var/mail** vemos que dentro de la carpeta camilo hay un archivo llamado **correo.txt** y si hacemos `cat correo.txt` vemos lo siguiente: <br>
![image](https://github.com/TerrorAterrador/WriteUps/assets/128630899/fa0424d5-95a0-49bc-b68a-36359364d7b9)
<br>
<br>
Por lo que vamos a a probar camibar al usuario **juan** que era el que había enviado el correo, si hacemos `su juan` para cambiar de usuario y ponemos la contraseña, vemos que hemos cambiado al usuario juan.<br>
![image](https://github.com/TerrorAterrador/WriteUps/assets/128630899/3152c784-7761-4885-ab5d-947cf823b1aa)
<br>
<br>
Ahora que estamos como juan vamos a listar los permisos que podemos ejecutar como **sudo** con `sudo -l`, y vemos que podemos ejecutar **/usr/bin/ruby**. <br>
![image](https://github.com/TerrorAterrador/WriteUps/assets/128630899/6ffb810b-3612-4bbb-bd5c-eca73f2643db)
<br>
<br>
Por lo que deberíamos hacer ahora es dirigirnos a la página [GTFOBins](https://gtfobins.github.io/) (está página nos indica como elevar privilegios dependiendo del binario que podamos ejecutar), después nos vamos a la parte de sudo en ruby, y nos encontramos con el siguiente comando `sudo ruby -e 'exec "/bin/bash"'`, probamos a ejecutarlo en la máquina y listo hemos elevado a root. <br>
![image](https://github.com/TerrorAterrador/WriteUps/assets/128630899/375ab6da-cf9a-4cec-af6a-47237ba931e8)






