# [Fileception](https://dockerlabs.es/)

## Despliegue

Primero desplegamos la máquina con `bash auto_deploy.sh fileception.tar` (si no sabes en la página de DockerLabs ahí un pdf que lo explica).

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

Podemos ver los resultados en el archivo grepeable haciendo `cat allPorts`, observamos están abiertos los puertos **21**, **22** y el puerto **80**.
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/e19baee4-3650-49c6-af18-8971e6923999)

<br>

Al ver que está abierto el puerto FTP vamos ha hacer un escaneo de nmap pero para que nos liste más información. Para llevar a cabo eso debemos hacer ```nmap -p21 -sCV 172.17.0.2 -oN targeted``` <br>
`-p21` ⮞ aplicar el escaneo solo al puerto 21 <br>
`-sC` ⮞ ejecuta los scripts de reconocimiento básico, los más comunes <br> 
`-sV` ⮞ para conocer la versión del servicio que corre por el puerto (se puede juntar con el anterior y quedaría así `-sCV`)<br> 
`-sS` ⮞ para descubrir puertos de manera silenciosa y rápida <br> 
`-oN` ⮞ lo exporta en formato nmap al archivo targeted
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/ad242591-a85b-4933-8c45-550c60c2a7fe)

<br>
<br>

## FTP (Puerto 21)

Una vez ya conozcamos que la versión del ftp es vulnerable al login como `anonymous`, por lo que nos logeamos como este de la siguiente forma `ftp 172.17.0.2` poniendo `anonymous` como login y contraseña, y vemos que tiene un archivo llamado `hello_peter.jpg`:
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/6da74abf-4a3f-464e-bb58-9873f830e482)

<br>

Nos lo guardamos en nuestra máquina con `get hello_peter.jpg`, al tratarse de una imagen intentaremos ver los metadatos de la imagen, de la siguiente forma `steghide extract -sf hello_peter.jpg`, pero como no tenemos la passphrase no podremos ver si hay algún archivo oculta en la imagen:
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/1b087867-ae5f-4bfe-a04e-862e1558b2fd)

<br>

Si probamos a crackear la passphrase con `stegseek --crack hello_peter.jpg /usr/share/wordlists/rockyou.txt secret.txt`, tampoco tendremos éxito:
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/9fd592fa-4151-4896-9b4b-e240bd0fca5e)

<br>
<br>

## Página Web (Puerto 80)

Al ver que está abierto el puerto **80** nos dirigimos al Navegador Web e introducimos la dirección IP como URL. Nos encontramos con página por defecto de Apache2.
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/aa7e1c9e-db22-45f2-905c-57b568b4d069)

<br>

Si inspeccionamos el código fuente con `CTRL + U`, nos encontramos con el siguiente comentario que nos estaría dando unas pista:
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/2742d96a-1545-4fb0-b515-72f5b193e111)

<br>

Nos encontramos con un posible usuario **peter** y una posible contraseña **@UX=h?T9oMA7]7hA7]:YE+*g/GAhM4** encriptada en base, pero no sabemos cual, para ello enteramos a [CyberChef](https://gchq.github.io/CyberChef/)
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/0cde0221-6a6d-4358-b4c4-8c1af4d80e26)

<br>

Entonces la damos a la barita mágica, y nos detecta que es base85:
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/52d2ed0e-7742-4d49-9fff-97d46f2cf64d)

<br>

Y nos devuelve esta contraseña **base_85_decoded_password**, por lo que probaremos intentar a volver a sacar los metadatos de la imagen como intentamos anteriormente con el comando `steghide extract -sf hello_peter.jpg`, y de passphrase ponemos **base_85_decoded_password**, y nos extrae un archivo llamado `you_find_me.txt`:
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/140d0d76-c863-4ad3-a568-b514967cb754)
<br>

Dicho archivo tiene el siguiente contenido:
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/0449d054-1847-41f4-a771-d07bafb00971)
<br>

Vemos que se trata de [Ook!](https://es.wikipedia.org/wiki/Ook!)
<br>

 > En el caso de que no sepamos que se trata de Ook! siempre le podemos preguntar a [ChatGPT](https://chatgpt.com/)
<br>

Ahora debemos ver que se esconde detrás de este código de Ook!, para ellos nos iremos a está página [Cachesleuth](https://www.cachesleuth.com/bfook.html), introducimos el código el Ook! y nos devuelve la siguiente cadena:
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/0135bd9f-4a74-4524-902e-067255ef6318)

<br>

 > Hay que llevar cuidado con algunas páginas, ya que no te lo desencriptan bien.

<br>
<br>

## Escalada de Privilegios

Una vez que tenemos un posible usuario **peter** y una contraseña **9h889h23hhss2** probamos a autenticarnos por ssh ya que dicho puerto estaba abierto. Lo haremos de la siguiente forma `ssh peter@172.17.0.2` e introducimos la contraseña.

<br>

En este caso no será necesario el Tratamiento de la TTY ya que esta bien configurada esta.

<br>

Comprobamos que hemos podido ingresar a la Máquina Víctima como **peter**:
<br>

 > El (;) concatena dos comandos.

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/3108f094-8a57-45a3-88bf-7ce67160d5e1)

<br>

Si ejecutamos `sudo -l` podemos ver que no podemos correr nada<br>
`-l` ⮞ listar comandos que podemos ejecutar como sudo<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/eb708bf0-7630-4f92-a231-a1631b0a05f0)

<br>

Hacemos la siguiente comprobación para ver la posible escalada ejecutando `find / -perm -4000 2>/dev/null`, pero tampoco encontramos nada interesante.<br>
`/` ⮞ buscamos desde la raíz <br>
`-perm -4000` ⮞ mostrar los permisos SUID <br>
`2>/dev/null` ⮞ para que no nos muestre los errores
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/f63cb1d4-66f2-47a3-b81a-005268613077)

<br>

En cambio en el directorio personal de peter(`/home/peter`), nos encontramos con un `.txt` el cual nos da una pista:
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/119fd2fd-b246-468b-86cb-c2b970d65b64)

<br>

Por lo que nos dirigimos al directorio `/tmp` con `cd /tmp`, nos encontramos con los siguientes archivos:
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/46b403f9-1260-466c-b59f-dd5856e41bd9)

<br>

Si vemos el contenido del archivo `recuerdos_del_sysadmin.txt`, nos da una pista con respecto al archivo `.odt`:
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/59bbcd1c-7c97-45d8-a72d-f2af29f50371)

<br>

Por lo que nos compartimos el archivo `importante_octopus.odt` a nuestra máquina de atacante para que sea más fácil operar. Lo haremos con el siguiente comando `python3 -m http.server 90`, luego desde nuestra máquina de atacante nos lo traemos con `wget 172.17.0.2:90/importante_octopus.odt`:
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/b8cc934c-d63e-40be-9ac2-95a4cb87eae4)

<br>

Y si pensamos un poco respecto a la pista y lo que tenemos, debemos caer de si a un archivo `.odt` le cambiamos la extensión a `.zip` podemos ver el contenido de este, por lo que le cambiamos el nombre `mv importante_octopus.odt importante_octopus.zip`, lo extraemos con `unzip importante_octopus.zip` y vemos los siguiente archivos:
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/eb93b5f6-c351-4f9e-b2d7-5196abb98151)

<br>

Vemos que hay un archivo llamado `leerme.xml`, si hacemos un `cat leerme.xml` vemos que tiene el siguiente contenido:
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/aaa53acc-994b-4ad3-af28-3bc6dff173bd)

<br>

Por lo que tenemos un usuario llamado **octopus** y una contraseña encriptada **ODBoMjM4MGgzNHVvdW8zaDQ=**, observamos que la contraseña se encuentra encriptada en base64 por lo que haremos lo siguiente para desencriptarla `echo "ODBoMjM4MGgzNHVvdW8zaDQ=" | base64 -d`, y nos muestra lo siguiente:
<br>

 > Si viendo la contraseña encriptada no sabemos en que formato está siempre podemos acudir a [CyberChef](https://gchq.github.io/CyberChef/) o [ChatGPT](https://chatgpt.com/).
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/34dfd54f-f708-46b3-8b9d-80c00953a8d9)

<br>

Por lo que ya contamos con un nombre de usuario y una contraseña **octopus:80h2380h34uouo3h4**, entonces probamos a autenticarnos como este usuario de la siguiente forma `su octopus`, introducimos la contraseña y vemos que logeamos satisfactoriamente:
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/0683db57-b521-49ef-84cc-01b75c2a0c56)

<br>

Seguimos el mismo procedimiento de antes ejecutando `sudo -l` y observamos lo siguiente:
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/bbcad6b4-7656-4c25-9355-8b9bfdc57f33)

<br>

Ahora podemos ejecutar cualquier comando sin proporcionar contraseña por lo que haremos `sudo -u root /bin/bash`,
y listo ya somos **root!**
<br>

![image](https://github.com/TerrorAterrador/WriteUps/assets/146730674/3226f16d-d89e-4b89-9b7a-cae523f58d0d)
